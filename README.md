
# 🐉 gewyvern v0.02 Design Spec

### Linux eBPF Execution Runtime (CLI-first, Flow-aware)

MIT License
Status: Draft v0.02
Delta: execution semantics / flow emergence / reason chain / protocol-stack entry

---

# 0. 不变原则（继承 v0.01）

gewyvern 仍然是：

> kernel execution runtime
> CLI-first / root-only / session-based / non-daemon

不变边界：

* 不做 orchestration
* 不跨机器
* 不解释业务语义
* 不引入 ML
* 不推理
* 不做 observability 平台

---

# 1. v0.02 新增核心定位

在 v0.01 “execution runtime” 之上增加：

> **protocol-stack behavior observation runtime**

它不观察“程序”，而观察：

> 协议栈中的行为演化。

execution plane 仍为 eBPF
但 observation plane 进入协议层语义。

---

# 2. 执行入口修正（关键变化）

v0.01 中 execution 单位是：

> eBPF program

v0.02 增加：

> execution entrypoint anchored in protocol stack

即：

runtime 默认 attach 入口优先级：

1. protocol stack state
2. queue / routing
3. socket lineage
4. syscall（最后）

不是：

> syscall first

而是：

> behavior evolution first

---

# 3. Flow Emergence Model（新增）

v0.02 引入：

> flow 是 runtime emergent entity
> 不是用户定义对象

flow 定义：

```
flow :=
    连续状态演化
    + 路径连续
    + 协议生命周期锚
```

flow 不依赖：

* PID
* socket id
* interface

flow identity 依赖：

* state evolution continuity

---

## 3.1 flow 生命周期

```
emerge
→ evolve
→ diverge (path change)
→ terminate
```

规则：

* path change = new flow
* process change ≠ flow terminate
* socket change ≠ flow terminate

---

# 4. Behavior-first observation stack

默认观测顺序：

```
protocol state
→ routing decision
→ queue behavior
→ packet execution
→ OS attribution
→ cluster context
```

不是：

```
process → socket → network
```

---

# 5. Session 与 Flow 的关系（新增）

v0.01：session = execution boundary
v0.02：session + flow 形成：

> observation sandbox

关系：

```
session
  contains:
    observation scope
    templates
    flow registry
    intervention policy
```

flow 不属于 session
flow 被 session 观察。

---

# 6. Cluster 的地位修正

cluster 在 v0.02 中：

> environment metadata layer

不是：

* flow root
* observation entry

来源：

* cgroup
* namespace
* lineage
* socket attribution

仅用于：

* reason context
* debugging background

---

# 7. Reason Chain Model（新增核心）

v0.01：reason = metadata
v0.02：reason = structured causality chain

结构：

```
reason :=
  session context
  + flow identity
  + protocol state evidence
  + path evidence
  + intervention trace
```

特点：

* 不允许用户注释生成
* 不允许 AI 推理
* 必须源于物理事件

---

## 7.1 reason 分层

### L0 — physical facts

* packet
* state change
* routing

### L1 — runtime structure

* flow
* path
* lifecycle

### L2 — template interpretations

* anomaly classification

### L3 — reason aggregation

* causality output

---

# 8. Intervention Model 修订

干预等级：

1. drop (primary)
2. replay (future)
3. redirect (later)

执行原子：

> packet

决策锚：

> flow state

执行模型：

```
flow-level decision
→ packet-level execution
```

---

## 8.1 Intervention Safety Gate

必须具备：

* session scope
* template allowance
* reason chain
* CLI override

否则拒绝执行。

---

# 9. Lazy Interfere Enforcement（新增）

原则：

> observe fully or do not observe

规则：

* session scope 内禁止 sampling
* attach scope 必须闭环
* incomplete observation 禁止 attach

---

# 10. Protocol Lifecycle Anchor（新增）

flow identity anchored in：

* TCP lifecycle
* queue lifecycle
* routing lifecycle

不绑定：

* 单协议
* 单 socket

---

# 11. Execution vs Observation 分离

execution plane：

* eBPF program lifecycle

observation plane：

* flow emergence
* behavior structure
* reason generation

两者通过：

> session runtime bridge

连接。

---

# 12. Data Output 形态修订

v0.01：protobuf pipeline
v0.02：增加结构层

输出：

```
event stream
flow snapshots
reason chain
intervention logs
```

仍然：

* protobuf native
* JSON 仅 export

---

# 13. Single-Host Doctrine（明确）

gewyvern 永远：

* 单机 runtime
* 不跨节点
* 不 stitch flows

跨节点：

> leserpent 责任

---

# 14. Not in Scope（更新）

新增：

* 不构建 distributed trace
* 不做 service graph
* 不构建 topology engine
* 不进行 anomaly inference
* 不执行 automated policy

---

# 15. v0.02 最大变化总结

| v0.01             | v0.02                        |
| ----------------- | ---------------------------- |
| execution runtime | execution + behavior runtime |
| eBPF centric      | protocol state centric       |
| program/session   | session + emergent flow      |
| metadata reason   | causality chain              |
| attach pipeline   | observation sandbox          |
| CLI runtime       | debugger-like runtime        |

---

# 16. 下一步必须落地的不是代码

而是三个“不可跳过”的定义：

### A. Flow Registry 数据结构

runtime 如何保存“状态演化实体”

### B. Protocol Anchor Interface

如何接 TCP / queue / routing 状态

### C. Reason Aggregation Engine

如何从物理证据构建 reason

没有这三块：

写 probe = 无意义。

---

