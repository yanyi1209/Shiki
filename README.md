# Shiki

> 面向 Android 的安全远程 AI 编程 Agent。

Shiki 旨在让开发者通过 Android 手机创建和管理 AI 编码任务。代码与开发工具运行在远程 Ubuntu 工作区中；Agent 在隔离的 Docker 容器内读取、修改和测试 GitHub 仓库，并在用户审查和确认后创建 Draft Pull Request。

> [!WARNING]
> Shiki 目前处于设计与技术验证阶段，尚未达到生产可用状态，不建议用于高敏感代码或关键业务仓库。

## 核心能力（规划中）

- 通过 Android 创建、监控、暂停和取消编码任务
- 支持流式对话、结构化工具事件和只读命令日志
- 在手机锁屏、切换应用或短暂断网后继续执行服务器任务
- 展示代码 Diff、测试结果、模型用量和估算费用
- 对删除文件、扩大网络权限和推送代码等高风险操作进行分级审批
- 使用 GitHub App 实现最小权限仓库授权
- 每个任务运行在独立、受限且可清理的 Docker 容器中
- 用户使用自己的 Anthropic API Key，不共享模型额度
- 经用户最终确认后创建 Draft Pull Request，不自动合并代码

## 系统架构

```text
┌──────────────────────────────┐
│ Android Client               │
│ Kotlin + Jetpack Compose     │
└──────────────┬───────────────┘
               │ HTTPS / WebSocket
               ▼
┌──────────────────────────────┐
│ NestJS API                   │
│ Auth · Tasks · Approvals     │
│ GitHub App · Event Stream    │
└──────────┬───────────┬───────┘
           │           │
           ▼           ▼
┌────────────────┐  ┌────────────────────┐
│ PostgreSQL     │  │ Worker (systemd)   │
│ Prisma         │  │ Git · Docker · SDK │
└────────────────┘  └──────────┬─────────┘
                               │
                               ▼
                    ┌────────────────────┐
                    │ Isolated Container │
                    │ Claude · Node.js   │
                    │ Python · Workspace │
                    └────────────────────┘
```

Android 客户端负责交互、审批和状态展示；API 服务负责身份、权限与任务状态；独立 Worker 管理 Git 操作、Agent 生命周期和 Docker 容器。GitHub 凭证由 Worker 使用，不暴露给任务容器。

## 技术栈

| 层级 | 计划技术 |
| --- | --- |
| Android 客户端 | Kotlin、Jetpack Compose、Android Keystore |
| API 服务 | TypeScript、Node.js、NestJS |
| Agent 执行器 | 独立 Worker、systemd、Docker Engine |
| Agent 接口 | 优先验证 Claude Agent SDK / Claude Code 官方结构化接口 |
| 数据库 | PostgreSQL、Prisma |
| 实时通信 | REST API、WebSocket、事件序号补传 |
| GitHub 集成 | GitHub App、OAuth、Webhook、Draft Pull Request |
| 管理界面 | React、Vite、TypeScript |
| Monorepo | pnpm workspace、Gradle |
| API 契约 | OpenAPI、版本化 JSON Schema |
| 反向代理 | Caddy（待部署方案确认） |
| CI/CD | GitHub Actions、版本化 Docker 镜像、签名 APK |
| 监控与告警 | 结构化日志、基础指标、外部监控、飞书告警 |
| 备份 | 加密 PostgreSQL 备份、S3 类对象存储 |

## 安全设计原则

Shiki 将不可信仓库内容和 Agent 操作视为潜在风险输入，而不是默认可信代码。

- **最小权限：** GitHub App 只申请读取代码、写入任务分支和创建 Draft PR 所需权限。
- **任务隔离：** 每个任务使用独立容器、工作区和非 root Linux 身份。
- **凭证隔离：** GitHub Token 留在 Worker；Anthropic API Key 加密保存，并尽量只提供给 Claude 进程。
- **确定性审批：** 后端策略引擎决定允许、拒绝或要求审批，Agent 不能覆盖策略。
- **资源限制：** 限制 CPU、内存、进程数、磁盘和任务运行时间。
- **网络控制：** 原型阶段允许联网；团队试用前切换为必要服务白名单和临时域名审批。
- **审计与脱敏：** 保存任务事件和审批记录，对已知 Token、私钥和常见敏感字段自动脱敏。
- **安全交付：** 推送前执行 Diff 审查、测试结果展示、依赖检查和密钥扫描。
- **人工合并：** Shiki 只创建 Draft PR，不自动转为 Ready，也不自动合并。

Docker 能降低错误命令和恶意代码的影响范围，但不等同于虚拟机级别的绝对安全边界。

## 任务生命周期

```text
Queued
  → Preparing
  → Running
  → Waiting for Approval
  → Running
  → Validating
  → Waiting for Final Confirmation
  → Draft PR Created
  → Completed / Failed / Cancelled / Interrupted
```

任务具有持久状态。Android 断线后，服务端继续执行；客户端重连时使用每任务递增事件序号补取遗漏消息。审批等待最多保留 24 小时，执行时间默认限制为 60 分钟。

## 第一阶段技术验证

正式开发完整产品前，项目将优先验证 Claude 官方结构化接口是否能够可靠提供：

- 流式 Agent 消息
- 工具调用开始和结束事件
- 工具执行前的审批拦截
- 批准与拒绝操作
- 任务取消
- 工作目录约束
- Token 与费用用量
- 受限 Docker 容器内运行

该探针设置 20 个开发工时的决策门槛。如果官方接口无法可靠实现执行前审批，将停止当前内核路线，转入基于 Anthropic API 的受控 Agent 循环方案评审，而不是解析脆弱的终端文本输出。

## 路线图

1. **SDK 探针与执行内核**
   - Claude 结构化接口、Worker、Docker、PostgreSQL、测试 CLI、核心审批和 Diff
2. **GitHub 与身份链路**
   - GitHub App、邀请登录、仓库权限、任务分支和 Draft PR
3. **Android 核心流程**
   - 任务创建、聊天、状态、通知、审批、Diff 和最终确认
4. **安全运维与管理**
   - 网络策略、凭证保护、备份、监控、审计、Web 管理页和紧急停止
5. **邀请制小团队试用**
   - 完成安全回归和恢复演练后开放内部测试
6. **轻量代码编辑器**
   - 文件树、搜索、小文件编辑、草稿、哈希检测和冲突处理

## 计划中的仓库结构

```text
shiki/
├── apps/
│   ├── android/
│   ├── api/
│   ├── worker/
│   └── admin-web/
├── packages/
│   ├── database/
│   ├── observability/
│   ├── policy/
│   └── protocol/
├── tools/
│   ├── shiki-admin/
│   └── test-cli/
├── images/
│   └── task-runtime/
├── fixtures/
│   └── sample-repository/
└── deploy/
```

此结构是规划方案，目录会在对应开发阶段逐步建立。

## 开发环境

计划使用以下环境进行开发和验收：

- Windows 11 + WSL2 Ubuntu
- Docker Desktop（WSL2 Backend）
- Node.js + pnpm
- PostgreSQL
- Android Studio
- 真实 Ubuntu Server 作为最终部署和安全验收环境

开发环境和正式服务器必须使用不同的数据库、API Key、加密主密钥和管理入口。

## 项目状态

- [x] 产品目标与总体架构设计
- [x] 初步安全模型与分阶段路线
- [ ] Claude SDK 能力探针
- [ ] 执行内核原型
- [ ] GitHub App 与身份链路
- [ ] Android MVP
- [ ] 安全运维与邀请制试用

Shiki 是当前内部代号，正式产品名称和 Android `applicationId` 尚未确定。

## 文档

后续计划补充：

- 产品需求文档（PRD）
- 架构决策记录（ADR）
- 威胁模型
- 本地开发指南
- 部署与恢复手册
- 安全披露流程

## Contributing

项目目前处于私有原型阶段，暂未开放外部贡献流程。

## License

MIT
