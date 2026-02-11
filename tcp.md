---

# 1) Flow Registry v0 的目标边界

v0 只保证三件事：

1. **TCP 生命周期连续性**（SYN→EST→FIN/RST…）能物化为 flow
2. **path change = new flow** 能被识别（先用 routing/ifindex 的近似证据）
3. **process/socket change ≠ terminate**：用“continuity window + anchor evidence”跨越重绑定

v0 不做：

* UDP/ICMP 多协议统一
* 分布式 stitch
* 复杂因果推理（reason chain 先做到 L0/L1）

---

# 2) Transition Atom Schema（最小不可跳过）

你已经决定 primary anchor = TCP state transition。那 atom 的最小 schema 必须能支撑：

* 连续性（continuity）
* 分叉（diverge）
* 证据链（reason L0）
* 未来扩展 queue/routing

建议 v0 atom：

### `TcpAtom`

* `ts`：monotonic timestamp（ktime_ns）
* `cpu`：CPU id
* `type`：枚举

  * `TCP_STATE`（核心）
  * `TCP_SEG`（可选：SYN/FIN/RST/ACK…）
  * `ROUTE_DECISION`（可选：用于 path evidence）
* `conn_hint`（仅提示，不当 identity）

  * `saddr`, `daddr`, `sport`, `dport`（tuple）
  * `netns`（可选）
* `tcp_state_from`, `tcp_state_to`（当 type=TCP_STATE）
* `skb_markers`（可选）

  * `ifindex_in`, `ifindex_out`
  * `skb_hash`/`flow_hash`（提示）
* `route_hint`（当 type=ROUTE_DECISION）

  * `oif`, `dst_cache_hit`, `rt_table`
* `evidence_flags`

  * `is_retx`
  * `is_drop`
  * `is_local_origin`

注意：
`conn_hint` 在 v0 只是**寻找候选 flow 的索引**，不是 flow identity。

---

# 3) Flow Registry 的核心数据结构（timeline-native）

### 3.1 Runtime 内存模型（userspace）

你需要三个表 + 一个日志：

1. `TransitionLog`（ring / segmented log）

* 只存最近 N 秒（例如 30–120s），用于 continuity 回溯
* 每条记录就是 `TcpAtom`

2. `CandidateIndex`

* `tuple -> list(flow_candidate_id)`（短期索引）
* 目的：快速找到“可能连续”的候选实体

3. `FlowCandidate`

* `id`
* `anchor`：当前 TCP lifecycle anchor 状态（state + last_seen_ts）
* `continuity_score`：连续性评分（0..1 或整数）
* `path_fingerprint`：路径指纹（oif/ifindex/routing evidence 的聚合）
* `lineage`：父 flow（可选）
* `snapshot`：可输出的 flow snapshot（轻量）

4. `MaterializedFlow`（当 score 达阈值）

* `flow_id`（runtime 生成）
* `lifecycle`: emerge/evolve/diverge/terminate
* `evidence_ptrs`: 指向 TransitionLog 的区间引用（reason L0 基础）

这四个加起来就是你 spec 里说的 “flow registry 数据结构”。

---

# 4) Continuity Engine（v0 判定规则）

### 4.1 候选生成（candidate selection）

收到一个 `TcpAtom`：

1. 用 `conn_hint.tuple` 查 `CandidateIndex`，拿到 0..k 个候选
2. 若为空：

   * 创建新的 `FlowCandidate`（暂不 materialize）
3. 若不为空：

   * 对每个候选计算 `delta_score`，取最大者作为归属

### 4.2 连续性评分（continuity score）

v0 评分建议非常“工程化”，别哲学化：

* `time_continuity`：时间间隔越近越高（例如 exp(-Δt/τ)）
* `state_validity`：TCP 状态跳转合法性（SYN_SENT→ESTABLISHED 合法；ESTABLISHED→SYN_SENT 非法）
* `tuple_consistency`：tuple 是否一致（不一致扣分，但不直接否决——用于支持 socket 重建/重绑定的未来扩展）
* `path_consistency`：路径指纹是否稳定（变化则触发 diverge）

最后：

```
delta = w1*time + w2*state + w3*tuple + w4*path
candidate.score = clamp(candidate.score * decay + delta)
```

### 4.3 materialize 阈值

* 当 `candidate.score >= TH_EMERGE` 且出现关键 anchor 事件（例如进入 ESTABLISHED）
  → `emerge` 成 `MaterializedFlow`

这样可以避免“见到一个 SYN 就生成一堆 flow 噪音”。

---

# 5) Diverge：你 spec 的“path change = new flow”如何落地（v0）

v0 的 path evidence 不可能完美，所以要定义“可用的近似”。

### 5.1 路径指纹 `path_fingerprint`

v0 先用这几个字段组成：

* `oif / ifindex_out`（最重要）
* `route table id`（有则加）
* `dst_cache_hit`（辅助）
* （可选）`tc/qdisc` 相关 id（未来）

### 5.2 diverge 判定

当 flow 已 materialize 后：

* 若某个 `ROUTE_DECISION` 或带 `ifindex_out` 的 atom 显示：

  * `path_fingerprint` 发生变化 **且持续超过 K 个 atom / T ms**
  * 且 TCP 生命周期仍在进行（非 terminate）

则：

* 原 flow 标记 `diverge`（并 `terminate` 或进入 `forked` 状态，取决于你想不想保留 parent）
* 创建新 flow：

  * `lineage.parent = old_flow_id`
  * `emerge_reason = path_change`

这就严格符合你 spec：**path change = new flow**。

---

# 6) Terminate：TCP anchor 的终止规则（v0）

终止条件直接锚 TCP：

* 观察到 `FIN` 完成关闭状态（TIME_WAIT/ CLOSED）
* 观察到 `RST`（强终止）
* 或者 `idle_timeout`：超过一定时间无任何 atom（比如 2×RTO 或固定 60s）

注意：
**process change / socket id change 不参与 terminate**，因为 v0 根本不把 PID / socket 当 identity。

---

# 7) Reason Chain（v0 先做到 L0/L1 就够）

你 spec 里 reason chain 很关键，但 v0 不要冲到 L2/L3。

### L0 physical facts（直接来自 atom）

* state changes
* route decisions
* drops/retrans

### L1 runtime structure（来自 registry）

* flow_id
* lifecycle events（emerge/diverge/terminate）
* path fingerprint changes

输出格式上你已经规定了：

* `flow snapshots`
* `reason chain`
* `intervention logs`

那 v0 reason chain 就是：

```
reason = [
  {L0: atom_ref, fact: "tcp_state SYN_SENT→ESTABLISHED", ts: ...},
  {L0: atom_ref, fact: "route oif=eth0", ts: ...},
  {L1: flow_event, fact: "flow_emerge id=F123", ts: ...},
  ...
]
```

“必须源于物理事件”这个约束在 v0 很容易满足：
所有 reason 项都带 `atom_ref` 或 `flow_event_ref`。

---

# 8) 你可以立刻开工的最小落地清单（两天就能跑）

不需要写一堆 probe，先写最少的三个点：

1. **TCP state tracepoint/kprobe**（产出 `TCP_STATE` atom）
2. **路由决策的一个锚点**（产出 `ROUTE_DECISION` atom，哪怕粗糙）
3. userspace runtime：

   * TransitionLog
   * CandidateIndex
   * Continuity Engine（score + emerge）
   * 输出一个 `flow list` + `reason tail`

跑起来后你就能做到你最想要的：

> “排除具体软件的具体网络问题”
> 因为你终于有了**行为链**而不是日志猜谜。

---
