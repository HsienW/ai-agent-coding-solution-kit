# 设计模式映射：多 Agent Shared State 与 Handoff 并不是全新问题

[![English](https://img.shields.io/badge/English-555555?style=for-the-badge)](./README.md)
[![繁體中文](https://img.shields.io/badge/繁體中文-555555?style=for-the-badge)](./README-zh-TW.md)
[![简体中文](https://img.shields.io/badge/简体中文-0969da?style=for-the-badge)](./README-zh-CN.md)

## 1. 不是单一 GoF Pattern

Artifact-based Shared State + Structured Handoff 并没有一个完全一对一的 GoF 设计模式。

它更像是将多个既有架构模式组合，用来处理：

- 多个专业角色共享问题状态。
- 工作所有权转移。
- 长流程路由与失败回路。
- 状态持久化。
- 大型 Artifact 引用。
- 历史、快照与查询效率。
- 人工批准与不可逆操作。

因此，较合理的做法是理解每个 Pattern 在整体架构中的责任，不必强行宣称“它就是某个 Pattern”。

## 2. 模式对照总表

| 多 Agent 元件 | 对应模式 | 解决问题 |
|---|---|---|
| Shared State | Blackboard | 多个专家基于共同问题空间工作 |
| State Store Interface | Repository | 隔离 Domain 与 JSON／DB／Checkpointer |
| Handoff Envelope | Command Message / Envelope Wrapper | 将工作交接包成明确消息 |
| Artifact Reference | Claim Check | 传引用而非大型内容 |
| Workflow Routing | State Machine | 定义合法阶段与 Transition |
| Planner → Coder → Reviewer | Pipes and Filters | 每阶段消费 Artifact 并产生新 Artifact |
| Orchestrator | Process Manager | 根据中间结果决定下一个步骤 |
| Complete History | Event Sourcing | 以事件保存完整状态变更历史 |
| Current State | Snapshot / Materialized View | 提供高效、最新的操作视图 |
| Retry-safe Consumer | Idempotent Receiver | 避免重试重复产生副作用 |
| Human Approval | Approval Gate / Interrupt-Resume | 在高风险操作前暂停 |

## 3. Blackboard Pattern

### 核心思想

多个 Knowledge Sources 不必互相直接知道对方，而是通过共同 Blackboard 读取当前状态并写入新事实。

```text
Planner ───────┐
Implementer ───┼── Shared Blackboard
Reviewer ──────┘
```

映射到多 Agent Coding Workflow：

- Planner 写入批准 Scope 与 Acceptance Criteria。
- Implementer 写入 Implementation Result 与 Evidence。
- Reviewer 写入 Findings 与 Verdict。
- Coordinator 根据 Blackboard 做仲裁与 Readiness。

### 适用价值

- 降低 Agent 之间点对点耦合。
- 允许角色由不同模型或工具实现。
- 共同状态可以被人与程序检查。

### 风险

- 所有人都能任意写 Blackboard 时，容易污染状态。
- Blackboard 变成无结构大杂烩。
- 缺少 Control Plane 时，不知道何时轮到谁。

因此需要搭配：

- Schema。
- Ownership。
- State Machine。
- Structured Handoff。

## 4. Repository Pattern

Martin Fowler 对 Repository 的描述是：在 Domain 与数据映射层之间提供类似集合的接口，隔离底层持久化细节。

多 Agent Workflow 可以定义：

```ts
interface WorkflowStateRepository {
  getCurrentState(changeId: string): Promise<CurrentState>;
  saveArtifact(artifact: AgentArtifact): Promise<ArtifactRef>;
  updateState(expectedVersion: number, next: CurrentState): Promise<void>;
}
```

第一版底层可以是：

- 本地 JSON。

未来可以替换成：

- SQLite。
- PostgreSQL。
- Redis。
- Object Storage Metadata。
- LangGraph Checkpointer。

Agent 与 Routing Logic 不需要知道存储细节。

### 适用价值

- 支持渐进式演进。
- 易于测试。
- 避免 Orchestrator 与文件路径紧耦合。

### 风险

- 过度泛化会产生复杂抽象。
- 不同 Store 的事务与一致性能力不同，不能假装完全等价。

## 5. Command Message 与 Envelope Wrapper

Structured Handoff 接近 Command Message：

> 请指定角色，基于指定输入，执行指定阶段，产生指定结果。

Envelope 则包含：

- Routing Metadata。
- Correlation ID。
- Schema Version。
- Security Context。
- Required References。
- Expected Output。

真正的工作内容放在被引用的 Artifact 中。

### 适用价值

- Handoff 可被验证。
- 容易建立 Adapter。
- 可加入 Trace、Timeout、Retry 等 metadata。

### 风险

- 把完整 Artifact 重复塞进 Envelope，会失去分离价值。
- Envelope 字段过多时会变成另一份 Current State。

## 6. Claim Check Pattern

Claim Check Pattern 的精神是：

- 大型或敏感 Payload 放在专门 Store。
- 流程消息只携带可受控的引用。

映射：

```text
Handoff
  └─ diffRef: runtime://evidence/git-diff.patch
```

而不是：

```text
Handoff
  └─ diff: "完整十萬字 patch"
```

### 适用价值

- 降低 Token 与消息大小。
- 可独立设定 Artifact 权限与保留期限。
- 容易使用 Checksum 验证。

### 风险

- Reference 失效。
- Path Traversal。
- Artifact Store 权限错误。
- Reference 与 State 版本不一致。

## 7. State Machine

State Machine 是 Structured Handoff 的秩序来源。

```text
READY_FOR_REVIEW
    ↓ review.requested
REVIEWING
    ├─ approve → READY_FOR_READINESS_CHECK
    ├─ changes → CHANGES_REQUESTED
    └─ conflict → NEEDS_COORDINATOR_ARBITRATION
```

### 适用价值

- 非法 Transition 可被拒绝。
- 容易测试。
- 流程可视化。
- 不依赖模型自由生成下一步。

### 风险

- 状态数量无控制地膨胀。
- 将所有细节都设计成 State，造成复杂度。

建议只把影响路由、权限与恢复的状态显性化。

## 8. Pipes and Filters

规划、实现、审查都可视为 Filter：

```text
Requirement
→ Plan Filter
→ Implementation Filter
→ Review Filter
→ Readiness Filter
```

每个 Filter：

- 接受明确输入。
- 进行专业转换。
- 产生 Artifact。

### 适用价值

- 角色可替换。
- 每阶段可独立测试。
- 容易新增 Validation Filter。

### 限制

真正 Coding Workflow 通常不是纯线性：

```text
Reviewer → Implementer → Reviewer
```

因此需要搭配 State Machine 与 Process Manager。

## 9. Process Manager

Enterprise Integration Patterns 将 Process Manager 描述为：中央处理单元保存流程状态，并根据中间结果决定下一步。

这几乎就是 Node Orchestrator 的责任：

- 维持 Current State。
- 调用下一个 Agent。
- 根据 Result 路由。
- 管理 Retry 与 Human Gate。

### 适用价值

- 支持非线性流程。
- 支持条件分支与并行节点。
- 集中管理流程安全。

### 风险

- 中央瓶颈。
- 变成什么都管的 God Object。
- 所有 Domain Logic 被塞进 Orchestrator。

控制方式：

- Orchestrator 只管理流程，不承担 Agent 专业判断。
- 将 Storage、Adapter、Policy、Validation 分层。
- 保持 State Contract 稳定。

## 10. Event Sourcing

Event Sourcing 以 append-only Event 保存完整变更历史：

```text
PlanApproved
ImplementationCompleted
ReviewRequested
ChangesRequested
ReviewApproved
HumanCommitApproved
```

### 适用价值

- 完整审计。
- 可重建 State。
- 容易分析流程历史。

### 风险

- Event Versioning。
- Replay 成本。
- Ordering。
- Idempotency。
- Snapshot 与 Migration。

第一版通常不需要完整导入。可先使用：

```text
Current State + Latest Artifact + Important Approval Record
```

## 11. Snapshot / Materialized View

Current State 可以视为完整历史的操作快照：

```text
Full Events / Artifacts
          ↓ projection
Current State
```

Agent 正常执行读 Snapshot，而不是每次重播全部历史。

### 适用价值

- 降低读取与 Token 成本。
- 清楚识别当前可接受版本。
- 快速恢复。

### 风险

- Snapshot 与来源不一致。
- 更新失败造成 Stale State。

控制方式：

- State Version。
- Atomic Update。
- Checksum。
- 必要时可从事件重建。

## 12. Idempotent Receiver

Agent 或 CLI 可能因 Timeout 被重试。若同一 Result 被处理两次，不能重复：

- 修改 State。
- 建立相同 Artifact。
- 发起 Commit。
- 触发 Deploy。

可以使用：

- `runId`
- `handoffId`
- `artifactId`
- `idempotencyKey`
- Expected State Version

判断是否已处理。

## 13. 模式组合

完整架构可以表示为：

```text
                         ┌──────────────────┐
                         │ Durable Knowledge│
                         └────────┬─────────┘
                                  │
                                  ▼
┌───────────┐   Command/Handoff  ┌────────────────┐
│   Agent   │ ◄────────────────► │ Process Manager│
└─────┬─────┘                    └───────┬────────┘
      │                                  │
      │ Artifact                         │ State Transition
      ▼                                  ▼
┌────────────────────────────────────────────┐
│ Blackboard / Shared State                  │
│ Current Snapshot + Artifact References     │
└───────────────────┬────────────────────────┘
                    │ Repository
                    ▼
       ┌────────────────────────────┐
       │ Runtime / Artifact Storage │
       └────────────────────────────┘
```

模式责任：

- Blackboard：共享问题空间。
- Repository：隔离存储。
- Handoff／Command：传递工作。
- State Machine：限制流程。
- Process Manager：执行路由。
- Claim Check：引用大型 Artifact。
- Snapshot：高效读取当前状态。
- HITL Gate：保护不可逆操作。

## 14. 不要过度模式化

设计模式是帮助思考，不是要求第一版实现所有元件。

最小可用组合：

```text
Blackboard-like Current State
+ Structured Command/Handoff
+ Explicit State Machine
+ Local Repository Adapter
+ Human Approval Gate
```

只有当需求出现时，再增加：

- Process Manager 自动化。
- Event Sourcing。
- Queue。
- Distributed Lock。
- Full Trace Platform。

## 15. 参考资料

- Martin Fowler：Repository  
  <https://martinfowler.com/eaaCatalog/repository.html>
- Enterprise Integration Patterns：Process Manager  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html>
- Enterprise Integration Patterns：Message / Pipes and Filters / Claim Check  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/Message.html>  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html>  
  <https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html>
- Azure Architecture Center：Event Sourcing / Materialized View / Claim Check  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view>  
  <https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check>
