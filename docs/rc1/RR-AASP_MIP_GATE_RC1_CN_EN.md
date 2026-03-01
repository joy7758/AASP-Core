# RR-AASP RC1 Integration Addendum: MIP Capability Gate / RR-AASP RC1 集成补充：MIP 能力门禁

Updated / 更新时间：2026-03-01 23:05 UTC+08:00
Standard ID / 标准编号：`RR-AASP`
Profile / 配置：`AAR-MCP-2.0 (Concept-Open, Core-Closed)`

---

## 0) Positioning / 文档定位

**EN**  
This addendum defines an integration profile for exposing the governance interface of MIP-based capability enforcement while keeping core physical operators closed-source.  
Goal: make side-effect bypass **structurally impossible** at the architecture layer.

**中文**  
本补充定义了一个集成配置：对外公开 MIP 能力门禁的治理接口，同时保持物理核心算子闭源。  
目标是在架构层实现对越权副作用的“结构性不可行（Structurally Impossible）”。

---

## 1) Scope and Boundary / 范围与边界

### Public Surface (Open) / 对外开放层（开源）

- Broker/Gate process boundary and IPC contract
- Capability token envelope schema
- Intent traceability rules
- Fail-closed behavior and audit bundle format
- Adapter contracts for Pydantic AI and CrewAI

### Protected Core (Closed) / 受保护核心（闭源）

- Small-scale fading feature extraction operators
- PUF space alignment and spatial hash model
- AmBC nonlinear modulation internals
- Any exportable implementation details of `mip_verify_physical_fingerprint(...)`

### License Note / 许可证说明

This addendum is an architecture profile and does not override the repository root `MIT` license.  
本补充属于架构配置说明，不覆盖仓库根目录 `MIT` 许可证。

---

## 2) Threat Model / 威胁模型

### Assumed attacker capabilities / 假设攻击者能力

- Process-level code execution in tool runtime
- Journal hash replay or software-only token forgery
- Prompt-injection attempts to trigger unsafe I/O
- Dependency-level 0-day in orchestration framework

### Security objective / 安全目标

No high-risk side effect can execute unless **all three** checks pass:

1. Journal-minted context check
2. Physical fingerprint authenticity check (closed SDK)
3. Intent-to-syscall conformance check

---

## 3) RC1 Normative Requirements / RC1 规范性要求

`MUST` / `必须`:

- All critical side-effect syscalls must be mediated by Gate/Broker.
- Tool sandbox must not hold direct privileged file/socket/payment handles.
- Gate must enforce fail-closed on missing or invalid token fields.
- Physical fingerprint verification must run out-of-process via closed SDK.
- Audit bundle must include decision path and immutable correlation IDs.

`SHOULD` / `应当`:

- Use seccomp/eBPF for syscall-level interception.
- Use per-request nonce and short token TTL (<= 5s) to reduce replay window.
- Separate Journal signer key from Broker runtime key.

`MAY` / `可选`:

- Hardware-backed monotonic counters in token envelope.
- Dual-channel Broker quorum for regulated deployments.

---

## 4) Reference Flow / 参考流程

1. Tool runtime requests a side effect (`write/network/payment`).
2. Syscall interceptor marks request as critical.
3. Runtime sends IPC request to Broker with capability token envelope.
4. Broker validates:
   1) journal mint flag  
   2) MIP physical fingerprint via closed SDK  
   3) intent hash binding to concrete syscall
5. If any check fails: immediate fail-closed + forced process termination.
6. If all checks pass: Broker mints ephemeral capability handle and executes.

---

## 5) Public Token Envelope (RC1) / 对外令牌封装（RC1）

```json
{
  "token_id": "uuid-v7",
  "intent_hash": "sha256(...)",
  "syscall_class": "fs.write|net.connect|payment.transfer",
  "journal_minted": true,
  "spatial_fingerprint": "opaque-base64",
  "issued_at": "2026-03-01T23:05:00+08:00",
  "expires_at": "2026-03-01T23:05:05+08:00",
  "nonce": "96-bit-random",
  "trace_id": "rr-aasp-trace-..."
}
```

`spatial_fingerprint` is opaque by design. Only authenticity status is externally observable.  
`spatial_fingerprint` 设计为不透明字段，对外只暴露认证状态，不暴露算子细节。

---

## 6) Pydantic AI Integration Profile / Pydantic AI 集成配置

### Runtime wiring / 运行时接线

- `ToolCallPreHook`: classify side-effect risk class
- `JournalEmitter`: create signed intent record
- `BrokerAdapter`: submit capability request over IPC
- `GateDecisionHandler`: map `ALLOW | DENY | MELTDOWN` to runtime actions

### Minimal adapter contract / 最小适配契约

```python
class CapabilityBroker(Protocol):
    def request(self, envelope: dict) -> "GateDecision": ...

class GateDecision(BaseModel):
    decision: Literal["ALLOW", "DENY", "MELTDOWN"]
    reason_code: str
    trace_id: str
```

---

## 7) CrewAI Integration Profile / CrewAI 集成配置

### Runtime wiring / 运行时接线

- `TaskGuard`: enforce capability requirement before task side effects
- `AgentPolicyBridge`: bind agent intent to journal entry hash
- `BrokerAdapter`: common IPC bridge shared with Pydantic profile

### Execution policy / 执行策略

- Any task with external side effects must route through Broker.
- Agent memory updates are allowed locally, but side effects are not.
- Repeated DENY events should escalate task to supervisor or stop flow.

---

## 8) Audit Bundle Requirements / 审计包要求

Each denied or allowed critical action must emit:

- `trace_id`, `token_id`, `agent_id`, `task_id`
- decision (`ALLOW|DENY|MELTDOWN`) and reason code
- timestamp, ttl, nonce digest
- syscall class and intent hash
- immutable signature of Broker decision

Recommended storage: append-only log + periodic anchor hash export.  
建议存储：追加写日志 + 周期性锚定哈希导出。

---

## 9) Compliance Checklist (RC1 Exit) / RC1 验收清单

- [ ] No direct privileged handles in tool sandbox
- [ ] Critical syscall classes intercepted and mediated
- [ ] Physical fingerprint check invoked for every critical action
- [ ] Fail-closed and forced termination verified under bypass attempts
- [ ] Replay resistance tested with nonce + TTL expiration
- [ ] Full audit bundle generated and reproducible
- [ ] Pydantic AI and CrewAI adapters pass integration tests

---

## 10) External Communication Template / 对外沟通模板

Use this short statement for maintainers/reviewers:

> This profile separates open governance interfaces from closed physical-root operators.  
> Side effects are impossible without a journal-minted, physically authenticated, intent-bound capability token.

中文版本：

> 本配置将开放治理接口与闭源物理根算子分离。  
> 任何副作用都必须同时满足“日志铸造 + 物理认证 + 意图绑定”三重条件，否则系统硬失败。

---

## 11) Out of Scope / 非范围

- Public release of physical-layer operator math
- Export of closed SDK internals
- Claims of absolute security beyond defined threat model

---

## 12) Next Step to RC2 / 面向 RC2 的下一步

- Add formal TLA+/I/O automata model for gate liveness/safety
- Add conformance test vectors for framework adapters
- Publish reproducible red-team harness for bypass simulation

