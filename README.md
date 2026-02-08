---

# 🐉 gewyvern v0.01 Design Spec

### Linux eBPF Execution Runtime (CLI-first)

MIT License
Status: Draft v0.01

---

## 0. 定位

gewyvern 是：

> **Linux eBPF execution runtime tool**
> CLI-first、root-only、session-based、non-daemon。

它不是：

* agent
* orchestration runtime
* cluster node
* control plane

它是：

> kernel execution controller。

---

## 1. 核心原则

1. execution authority 在 runtime 本体
2. 所有任务以 session 为原子单位
3. gewyvern 生命周期结束 → 所有执行必须清理
4. 默认冷启动，不继承未知状态
5. 不扫描、不干扰其它 eBPF 工具
6. control plane 不直接操作 execution plane
7. 风险执行只能本机 CLI override

---

## 2. 权限模型

* 仅 root 支持
* 非 root 拒绝执行
* 所有 kernel 操作由本机发起

---

## 3. 运行形态

### CLI-first

```
gew session apply …
gew session detach …
gew probe list …
gew runtime info …
```

### 可选后台模式

* 仅 session 运行期存在
* 非常驻 daemon

---

## 4. 执行单位

kernel execution 最小单位：

> eBPF program

runtime 管理单位：

> session（program 组合）

---

## 5. Session 模型

session 是：

> transaction attach boundary

流程：

```
create
→ stage
→ verify
→ attach
→ commit
```

失败：

```
rollback
```

---

## 6. bpffs ownership

pin root：

```
/sys/fs/bpf/gewyvern/
```

结构：

```
instances/<instance_id>/sessions/<session_id>/
```

规则：

* 仅操作本目录
* 不扫描全局
* 不触碰其它工具对象

---

## 7. 冷启动与残留清理

启动：

```
scan instance path
→ detect orphan sessions
→ cleanup
```

cleanup：

* detach links
* remove maps
* remove programs

---

## 8. Capability negotiation

连接后必须执行：

```
GetRuntimeInfo
GetCapabilities
VerifyCompatibility
```

三态：

1. not supported
2. supported but risky
3. fully supported

---

## 9. 风险策略

默认：

* risky 禁止

允许：

```
gewy --allow-risky
```

仅本机 CLI。

control plane 不允许远程 override。

---

## 10. Pipeline spec

唯一执行契约：

> protobuf

runtime 不解析：

* CLI string
* JSON

JSON 仅用于：

* export
* replay

---

## 11. 控制接口

gRPC：

* 默认 UDS
* TCP 可选
* 必须已配对 leserpent

---

## 12. 安全模型

pairing：

* token
* public key exchange

运行期：

* 每请求签名
* 验证 leserpent 公钥

---

## 13. 不做的事情

* 不做 orchestration
* 不做用户系统
* 不做 cluster coordination
* 不做 DSL runtime

---
