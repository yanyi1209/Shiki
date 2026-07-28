# Shiki

> 面向个人开发者的本地优先 AI 编程系统：先通过普通对话理解问题，需要执行时再显式进入可审计的工作流。

Shiki 不是托管式 AI 编程 SaaS。用户自己的电脑、开发机、家庭服务器或个人 VPS 是 **Host**；运行在其上的 **Shiki Daemon** 负责对话、项目上下文、工作流、凭据、容器和 GitHub 交付。Android 应用通过设备配对连接 Daemon，用于普通对话、生成计划草稿、启动工作流、查看 Kanban、审查结果和确认 Draft PR。

> [!WARNING]
> Shiki 目前处于设计与技术验证阶段，尚未达到生产可用状态。请勿将其用于高敏感代码、关键业务仓库或无法承受凭据泄露与错误修改风险的环境。

## 两种使用模式

### Conversation：普通对话

Conversation 用于讨论想法、分析问题、理解代码和形成计划草稿：

- 可以不选择 Project，进行一般技术对话；
- 也可以选择 Daemon 已注册的 Project，使用专用只读能力分析固定 revision；
- 可以生成版本化 `PlanDraft`；
- 不创建 Git worktree；
- 不修改代码，不运行 shell、构建、测试、依赖安装或仓库脚本；
- 不执行 Git mutation、commit、push 或创建 PR；
- 用户在自然语言中要求“直接修改”不能自动升级权限。

Conversation 可以通过明确的“转为工作流任务”操作进入 Workflow。用户必须确认 Project、Goal、base ref 和验收标准；Daemon 固定 base commit 后生成不可变 `TaskBrief`。发送消息、选择 Project 或生成 PlanDraft 都不会隐式启动执行。

### Workflow Task：固定执行工作流

Workflow Task 用于真正修改、验证和交付代码。MVP 固定为：

```text
PLAN（AI，只读）
  → CODE（AI，受管 worktree 读写）
  → VALIDATE（Host 确定性验证）
  → REVIEW（AI，独立会话、只读快照）
  → HUMAN_REVIEW（用户审核）
      ├──拒绝──▶ 本地完成，不发布
      ├──要求修改──▶ 创建修订任务后原任务完成，不发布
      └──批准──▶ DRAFT_PR（Host 确定性发布）
```

只有 `PLAN`、`CODE`、`REVIEW` 是 AI 节点。每个 AI 节点固定完整 Claude model ID、版本化 prompt、预算、工具和工作区权限。`VALIDATE`、`HUMAN_REVIEW`、`DRAFT_PR` 不使用模型。

## 产品边界

- **本地优先：** Host 上的本地状态是唯一事实来源，使用版本化 JSON、追加 JSONL 和本地文件系统持久化。
- **用户自托管执行：** 对话、Claude Agent SDK、代码、构建和测试均在用户控制的设备上运行，不使用 Shiki 共享执行集群。
- **无中心身份系统：** 不要求 Shiki 中心账号、注册、组织或中心数据库。
- **权限显式升级：** Conversation 永远无代码写入和发布权限；只有用户确认转换后，Workflow 才能进入有副作用节点。
- **移动端是控制端：** Android 不执行仓库代码，也不是任务状态的事实来源。
- **直接连接优先：** 设备配对后通过 LAN 或用户自建 VPN 使用 HTTPS/WSS 连接 Host。
- **Kanban 是投影：** 卡片列由 Workflow、NodeRun、HumanReview 和 Delivery 推导，不能通过直接写列绕过节点。
- **人工交付：** HUMAN_REVIEW 前不向 GitHub push；批准后只创建 Draft PR，不自动 Ready、不 Force Push、不自动 Merge。
- **可选 Relay 是未来方向：** Relay 若实现，只转发端到端加密消息，不看明文、不保存权威状态、不执行任务。

## MVP 范围

| 维度 | MVP 范围 |
| --- | --- |
| Host | Windows 11 + WSL2 + Docker Desktop；原生 Ubuntu + Docker Engine |
| Client | Android 10+，Kotlin + Jetpack Compose |
| 模式 | Conversation + 固定 Workflow Task |
| Agent Runtime | 容器化 Claude Agent SDK TypeScript，固定精确版本 |
| 模型 | AI 节点使用固定完整 Claude model ID，不使用浮动别名或静默 fallback |
| Workflow | `PLAN → CODE → VALIDATE → REVIEW → HUMAN_REVIEW → DRAFT_PR`；Validation outcome failure 仍运行只读 REVIEW，Runner infrastructure failure 暂停推进 |
| 调度 | 每 Host 同时最多一个 AI NodeRun；多个 Conversation/Workflow 可排队 |
| 存储 | 版本化 JSON、追加 JSONL、本地 Artifact 和可重建 Projection |
| 工作区 | 仅 Workflow CODE 使用每任务独立 Git worktree |
| Forge | GitHub |
| 交付 | HUMAN_REVIEW 批准后由 Host 创建 Draft PR |

MVP 不包括通用 DAG、动态节点、无限修复循环、多 Provider、PTY 兼容、单 Host 多 AI 并发、Android 原始 prompt/model 编辑、任意 MCP、完整移动 IDE或自动 Merge。

## 核心能力（规划中）

### Conversation

- 新建、继续、归档普通对话；
- 可选 Project 只读上下文，并显示读取 revision；
- 生成和查看版本化 PlanDraft；
- 将选定消息与 PlanDraft 作为 TaskBrief 草稿来源；
- 显式转换前保持零代码副作用。

### Workflow

- 从 Conversation 转换，或从 Kanban Backlog 直接打开同一 TaskBrief 确认页；两种入口都必须确认 Project、Goal、base ref 和验收标准；
- 固化 TaskBrief、WorkflowDefinition、模型、prompt 和 base commit；
- 自动推进无需人工介入且满足策略的节点；
- 持久化 NodeRun、Artifact、Timeline、usage 和失败证据；
- Android 展示 Plan、Diff、Validation、AI Review 和 Human Review；
- 用户拒绝时不发布；批准后只创建 Draft PR；
- 将 GitHub Checks、Review、Ready/Merged/Closed 状态作为 Delivery 投影回流。

## 系统架构

```text
┌────────────────────────────────────────────────────┐
│ Android Client                                     │
│ Conversations · PlanDraft · Workflow Kanban        │
│ Timeline · Artifacts · HUMAN_REVIEW · Delivery     │
└──────────────────────┬─────────────────────────────┘
                       │ 设备配对 + HTTPS/WSS
                       │ LAN / 用户自建 VPN
                       ▼
┌────────────────────────────────────────────────────────────┐
│ 用户自己的 Host                                            │
│                                                            │
│ Shiki Daemon                                               │
│ ├── Conversation / PlanDraft / Conversion                  │
│ ├── Workflow Engine / Node Scheduler / Artifact Store      │
│ ├── Timeline / Kanban Projection / Policy                  │
│ ├── Project / worktree / Docker                            │
│ └── Human Review / GitHub Delivery                         │
│                                                            │
│ 单 Host AI Scheduler：任意时刻最多一个 AI NodeRun          │
└──────────────┬──────────────────────────────┬──────────────┘
               │                              │ Host 持有凭据
               ▼                              ▼
┌──────────────────────────────┐   ┌────────────────────────┐
│ Agent Runtime Container      │   │ GitHub                 │
│ Claude Agent SDK             │   │ Repo · Issue · PR      │
│ PLAN / CODE / REVIEW profile │   │ Checks · Review       │
│ 无 GitHub Token / Docker API │   └────────────────────────┘
└──────────────────────────────┘

┌──────────────────────────────┐
│ Validation Runner            │
│ 固定 validation profile     │
│ 不调用模型、不修改源代码    │
└──────────────────────────────┘
```

Daemon 是控制面和安全决策点。Claude Agent SDK 运行在按节点 profile 配置的容器中，不运行在拥有 Docker 和 GitHub 权限的 Daemon 进程内。Docker 不可用或策略检查失败时，正式 Workflow 不会静默退化为 Host 原生执行。

## Conversation 与转换流程

```text
新建 Conversation
  → 可选：选择 Project 只读 revision
  → 普通对话 / 代码分析
  → 可选：生成 PlanDraft
  → 用户点击“转为工作流任务”
  → 确认 Project、Goal、base ref、验收标准
  → Daemon 解析 base commit 并生成不可变 TaskBrief
  → 创建固定版本 WorkflowTask
  → 进入 PLAN
```

Conversation 保持独立并可继续查看。Workflow 保存 `sourceConversationId` 和可选 `sourcePlanDraftId`，但不会继承隐藏权限或把整段未筛选对话自动变成执行指令。

## Workflow 与 Kanban

Android Kanban 列：

```text
BACKLOG | PLAN | CODING | REVIEW | NEEDS YOU | DELIVERED
```

- `BACKLOG`：创建命令已成功并立即进入 PLAN 队列，等待 Host AI 槽；不另设 Start 命令；
- `PLAN`：PLAN NodeRun 已开始、已产出 Plan 或正在准备进入 CODE；
- `CODING`：CODE 或 VALIDATE；
- `REVIEW`：REVIEW NodeRun；
- `NEEDS YOU`：HUMAN_REVIEW、可处理失败或中断；
- `DELIVERED`：DRAFT_PR 正在发布、本地未发布完成或 Draft PR 已创建，以 badge 区分。

Kanban 只显示 WorkflowTask，不显示未转换 Conversation。首版不支持跨列自由拖动；任务创建后自动排队，取消、重试和审核使用明确领域命令。

## 状态与并发

```text
Conversation.lifecycle:
ACTIVE / ARCHIVED

WorkflowTask.lifecycle:
ACTIVE / WAITING_FOR_HUMAN /
COMPLETED / FAILED / CANCELLED / INTERRUPTED

WorkflowTask.currentStage:
PLAN / CODE / VALIDATE / REVIEW / HUMAN_REVIEW / DRAFT_PR

NodeRun.status:
BLOCKED / READY / QUEUED / RUNNING /
SUCCEEDED / FAILED / CANCELLED / INTERRUPTED / SKIPPED

HumanReview:
PENDING / APPROVED / REJECTED / CHANGES_REQUESTED
```

Conversation 回复、PlanDraft、PLAN、CODE 和 REVIEW 都是 AI NodeRun，共享 Host 的唯一 AI 执行槽。VALIDATE 和 DRAFT_PR 不占 AI 槽，但仍受资源、Project/worktree 和发布锁限制。HUMAN_REVIEW 批准后，WorkflowTask 原子地回到 `ACTIVE/DRAFT_PR` 并创建确定性发布 NodeRun。

`CHANGES_REQUESTED` 不自动回到 CODE，也不触发无限循环；MVP 进入 `NEEDS YOU`。用户通过统一 TaskBrief 创建事务确认修订 Project、Goal、base 和验收标准；新任务创建成功后，原任务以 `COMPLETED` 收敛并记录替代关系。可重试 NodeRun 失败时 WorkflowTask 统一进入 `WAITING_FOR_HUMAN`；创建新 NodeRun 后回到 `ACTIVE`。`FAILED`/`INTERRUPTED` WorkflowTask 不会被重试 API重新打开，恢复需创建新任务。

## Claude Agent SDK Runtime

MVP 使用官方 TypeScript 包 `@anthropic-ai/claude-agent-sdk`，固定精确版本并提交 lockfile。实现开始时重新核对所选版本，升级前重跑能力探针。

每个 AI NodeRun 固化并记录：

- 完整 model ID；
- prompt ID、version 和 digest；
- WorkflowDefinition version；
- Agent SDK 和 Claude Code runtime 版本；
- 容器镜像 digest；
- workspace access、tools 和 policy version；
- `maxTurns`、`maxBudgetUsd`、Host 墙钟 timeout；
- session ID、result subtype、usage、model usage、估算成本和停止原因。

运行原则：

- Conversation/PlanDraft/PLAN：只读，无 shell、仓库脚本和写工具；
- CODE：仅当前受管 worktree 文件可写，拒绝本地 Git mutation，按策略开放非 Git 工具；
- REVIEW：独立 SDK session，只读 CODE 停止后的固定快照；
- 不动态 `setModel`，不静默 fallback，不接受 Android 任意 model ID；
- 不把 SDK permission、hook 或 `cwd` 当作容器安全边界；
- Provider 凭据若进入容器，仍可能被同容器仓库代码读取，必须最小化、可撤销并明确提示。

## Artifact 与 Human Review

节点通过结构化结果传递信息。TaskBrief、HumanReviewBundle 和 PublishManifest 是由 Conversion/Workflow Engine 生成的领域对象；其余是 NodeRun 产生的 Artifact：

```text
TaskBrief                 # 领域输入，producer: WorkflowConversion
PlanArtifact              # NodeRun Artifact
CodeResultArtifact        # NodeRun Artifact
WorkspaceSnapshot         # NodeRun Artifact
DiffSummary               # NodeRun Artifact
ValidationReport          # NodeRun Artifact
ReviewReport              # NodeRun Artifact
HumanReviewBundle         # 领域门禁对象，producer: WorkflowEngine
PublishManifest           # 确定性领域对象，producer: WorkflowEngine
PullRequestReference      # DRAFT_PR NodeRun Artifact
```

PlanDraft 是 Conversation 聚合中的版本化领域对象，记录 producing AI NodeRun 和 model/prompt/runtime identity，但不重复存入通用 Artifact Store。NodeRun Artifact 必须记录 producer NodeRun 和可选 workspace digest；领域输入/门禁对象记录真实 producer（WorkflowConversion 或 WorkflowEngine），不能伪造 NodeRun。所有对象都有 Schema version、content digest 和来源，内容变化生成新对象。

HUMAN_REVIEW 必须绑定：Project、repository、base SHA、Workflow/version、实际模型、Plan、workspace、Diff、Validation、Review 和 PublishManifest digest。任一内容变化后，旧决定失效。

## 本地持久化

概念布局：

```text
$XDG_STATE_HOME/shiki/
├── conversations/<conversation-id>/
│   ├── state.json
│   ├── events.jsonl
│   ├── messages.jsonl
│   └── plan-drafts/
├── workflow-tasks/<workflow-task-id>/
│   ├── task-brief.json
│   ├── workflow.json
│   ├── state.json
│   ├── events.jsonl
│   ├── node-runs/
│   └── artifacts/
├── deliveries/
└── projections/kanban.json       # 可删除、可重建缓存

$XDG_DATA_HOME/shiki/
├── projects/
├── repositories/
└── worktrees/<workflow-task-id>/ # Conversation 下不得出现
```

Timeline 按聚合分开排序，事件使用 `aggregateType + aggregateId + sequence`。Projection 是读取优化，不是事实来源。GitHub、Claude 和设备秘密不写入普通 JSON/JSONL、Artifact 或日志。

## 安全边界

| 场景 | 模型 | 仓库读取 | 写入 | 命令 | GitHub 写入 |
| --- | --- | --- | --- | --- | --- |
| Conversation 无 Project | 是 | 无 | 否 | 否 | 否 |
| Conversation + Project | 是 | 专用只读 read/search | 否 | 否 | 否 |
| PlanDraft / PLAN | 是 | 固定 revision 只读 | 否 | 否 | 否 |
| CODE | 是 | 是 | 当前受管 worktree 文件；Git 元数据隔离 | 受限非 Git mutation 工具 | 否 |
| VALIDATE | 否 | 是 | 不修改源代码 | 固定 validation profile | 否 |
| REVIEW | 是 | Diff/验证/只读快照 | 否 | 否 | 否 |
| HUMAN_REVIEW | 否 | 固定审核包 | 否 | 否 | 否 |
| DRAFT_PR | 否 | Host 最终检查 | Host 受控 commit | 固定 Git/GitHub 操作 | 仅 Draft PR |

自然语言、仓库指令、prompt 和 Android 参数都不能切换权限 profile、容器参数、Project 路径或 GitHub 目标。

## 计划技术栈

| 层级 | 计划技术 |
| --- | --- |
| Android | Kotlin、Jetpack Compose、Android Keystore |
| Daemon | TypeScript、Node.js、本地 HTTPS API 与任务编排 |
| Agent Runtime | Claude Agent SDK TypeScript + Claude Code runtime，固定容器镜像 |
| Workflow | 版本化固定 WorkflowDefinition、Node Scheduler、Artifact Store |
| Prompt/Model | 完整 model ID、版本化 prompt 和 digest registry |
| 实时通信 | HTTPS、WSS、聚合事件序号与断线补传 |
| 本地存储 | 版本化 JSON、追加 JSONL、本地 Artifact、可重建 Projection |
| 执行 | Git worktree、Docker、按节点权限 profile 的容器 |
| Validation | Host 控制的版本化确定性 validation profile |
| GitHub | Host 侧 Git/GitHub API 或 CLI、Draft PR、Delivery 同步 |
| 契约 | OpenAPI、版本化 JSON Schema、生成 TypeScript/Kotlin 模型 |
| 工程 | pnpm workspace、Gradle、GitHub Actions |

## 第一阶段技术验证

- Claude Agent SDK 精确版本、捆绑 runtime 和目标容器平台；
- 固定完整 model ID、prompt digest 和实际模型记录；
- Conversation/PLAN 只读且无 shell、写入或仓库脚本能力；
- CODE 只能修改当前 worktree 文件，拒绝 `git add`、commit、tag、branch、push 等 Git mutation；
- REVIEW 使用独立 session 且不能修改 snapshot；
- SDK 消息、partial/result 顺序、工具事件和错误分类；
- `maxTurns`、`maxBudgetUsd`、AbortController、`close()` 和完整进程树终止；
- Provider 凭据暴露、独立 `CLAUDE_CONFIG_DIR` 和 setting sources；
- 单 Host AI NodeRun scheduler 覆盖 Conversation 与 Workflow；
- VALIDATE、HUMAN_REVIEW、DRAFT_PR 不产生模型请求；
- Human Review 前无 push，批准后只创建 Draft PR；
- Android Conversation/Workflow 断线补传和 Kanban 重建。

任何未验证能力在 UI 中关闭或标记未知，不能用 Fake Runtime 测试冒充真实 SDK、Docker 或 GitHub 证据。

## 路线图

1. **双模式领域与协议基线**
   - Conversation、PlanDraft、TaskBrief、WorkflowTask、NodeRun、Artifact、HumanReview、Delivery、Kanban Schema。
2. **Claude Agent SDK 容器探针**
   - 固定 SDK、模型、prompt、容器 profiles、认证、取消和错误边界。
3. **Daemon 双模式内核**
   - Conversation、显式转换、Workflow Engine、单 AI NodeRun scheduler、本地状态和恢复。
4. **固定 Workflow 执行**
   - PLAN、CODE、VALIDATE、REVIEW、Artifact、worktree 和失败语义。
5. **设备配对、协议与 Kanban**
   - HTTPS/WSS、两类聚合补传、转换幂等和可重建 Projection。
6. **HUMAN_REVIEW 与 GitHub 交付**
   - 结果摘要门禁、DRAFT_PR 确定性节点和 Delivery 对账。
7. **Android 双模式 MVP**
   - Conversations、PlanDraft、转换确认、Workflow stepper、Kanban 和 Human Review。
8. **跨平台与安全加固**
   - WSL2/Ubuntu、恢复、恶意仓库、凭据、网络和资源验收。
9. **后续评审**
   - 多 Provider、通用 DAG、有界修复、更多 Host 平台、可选 Relay 和更强隔离。

## 计划中的仓库结构

```text
Shiki/
├── apps/
│   ├── android/
│   └── daemon/
├── packages/
│   ├── protocol/
│   ├── conversation-core/
│   ├── workflow-core/
│   ├── node-scheduler/
│   ├── agent-runtime/
│   ├── prompt-registry/
│   ├── kanban-projection/
│   ├── host-platform/
│   ├── github/
│   └── security/
├── workflows/
│   └── default/
├── prompts/
│   ├── conversation/
│   ├── plan/
│   ├── code/
│   └── review/
├── container/
│   ├── agent-runtime/
│   └── validation/
├── tools/
│   ├── agent-sdk-probe/
│   └── protocol-probe/
└── tests/
    ├── protocol/
    ├── integration/
    ├── e2e/
    └── fixtures/
```

该结构是规划，不表示目录和实现已经存在。

## 开发环境

MVP 开发与验收目标仍是：

- Windows 11 + WSL2 Ubuntu + Docker Desktop；
- 原生 Ubuntu + Docker Engine；
- Android Studio、Android 模拟器和至少一台真机；
- 固定 Node.js、pnpm、JDK、Gradle、Agent SDK、模型、prompt 和镜像版本；
- 独立的 Anthropic/GitHub 测试身份与专用仓库。

当前开发环境可以随后配置，不改变上述架构。环境未就绪时只能验证文档、领域模型和 Fake Runtime；不得宣称真实容器、SDK 或 Draft PR 已通过。

## 项目状态

- [x] 确认本地优先、个人自托管定位
- [x] 确认 Conversation + Workflow 双模式
- [x] 确认显式转换和 Conversation 零副作用
- [x] 确认固定 Workflow 与 AI/确定性节点分离
- [x] 确认 Claude Agent SDK 容器化、固定 model/prompt
- [x] 确认 Kanban 是可重建投影、单 Host 单 AI NodeRun
- [x] 确认 HUMAN_REVIEW 后只创建 Draft PR
- [ ] 完成 Claude Agent SDK 与容器能力探针
- [ ] 实现 Daemon 双模式内核和 Fake Runtime
- [ ] 实现固定 Workflow、worktree 和 Validation
- [ ] 实现 Android Conversation 与 Kanban
- [ ] 实现 Human Review 与 GitHub Draft PR
- [ ] 完成 WSL2/Ubuntu 恢复与安全验收

## 文档

- [总体设计文档](docs/DESIGN_v0.2.md)：双模式领域、架构、状态、安全、API 和路线图；
- [开发环境指南](docs/DEVELOPMENT_SETUP_v0.2.md)：工具链、容器 profiles、探针、测试和验收。

三份 v0.2 文档共同采用 Conversation + explicit Workflow + Claude Agent SDK + Kanban + Draft PR-only 基线。后续变更产品或安全边界时，应在同一变更中同步文档并补充 ADR。

## Contributing

项目处于早期设计与验证阶段。涉及 AI 节点的变更必须说明完整 model ID、prompt version/digest、Agent SDK/runtime 和容器 profile；涉及确定性节点的变更必须说明 runner/config version。不能仅依据文档外观或 Fake Runtime 声称真实能力已通过。

## License

许可证尚未确定。在许可证文件正式加入仓库前，不授予复制、分发或再许可本项目代码的权利。
