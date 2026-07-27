# Shiki

> 面向个人开发者的本地优先 AI 编程任务系统：任务在你自己的 Host 上执行，Android 负责随身控制。

Shiki 不是托管式 AI 编程 SaaS。用户自己的电脑、开发机、家庭服务器或个人 VPS 是 **Host**；运行在其上的 **Shiki Daemon** 负责项目、任务、凭据、执行和 GitHub 交付。Android 应用通过设备配对连接 Daemon，用于创建任务、查看 Timeline、审查 Diff、确认操作和接收结果。

> [!WARNING]
> Shiki 目前处于设计与技术验证阶段，尚未达到生产可用状态。请勿将其用于高敏感代码、关键业务仓库或无法承受凭据泄露与错误修改风险的环境。

## 产品边界

- **本地优先：** Host 上的本地状态是唯一事实来源，首版使用本地文件和追加写入的 JSONL 事件日志持久化。
- **用户自托管执行：** 代码、Coding CLI、构建和测试均在用户控制的设备上运行，不使用 Shiki 提供的共享执行集群。
- **无中心身份系统：** 不要求 Shiki 中心账号、注册、成员、组织或邀请，也不依赖中心数据库。
- **移动端是控制端：** Android 不承载代码执行，也不是任务状态的唯一保存位置。
- **直接连接优先：** 手机完成设备配对后，通过局域网或用户自建 VPN 访问 Host，并使用 HTTPS/WSS 传输控制请求和事件。
- **可选 Relay 仅是未来方向：** 未来可能提供端到端加密的消息中继，以解决无法直连的问题；Relay 不应看到明文任务、代码或凭据，也不保存权威任务状态、不执行任务。
- **人工交付：** 用户最终确认前不向 GitHub 推送任务改动；确认后只创建 Draft PR，不自动转为 Ready、不自动合并。

## MVP 范围

首版刻意限制为：

- Host：Windows 11 + WSL2 + Docker Desktop，或原生 Ubuntu + Docker Engine；
- 控制端：Android；
- 代码托管：GitHub；
- Agent：接通并验证一个真实 Provider；
- 并发：每个 Host 同时只运行一个任务；
- 存储：本地文件、状态快照和追加 JSONL，不引入中心数据库；
- 交付：用户确认后由 Host 创建 GitHub Draft PR。

首版不以完整移动 IDE、多人协作、公共注册与计费、多 Provider、多任务并发、自动合并或通用云端执行平台为目标。

## 核心能力（规划中）

- 从 GitHub Issue 或用户直接输入创建编程任务；
- 在 Android 上查看流式消息、结构化事件、命令日志、Diff、测试结果和任务状态；
- 为每个任务创建独立 Git worktree 和一次性 Task Container；
- 在容器中运行真实 Coding CLI，修改代码并执行构建、测试和检查；
- 在协议确实支持时提供工具事件、取消、用量信息和执行前审批；
- 对不具备可靠细粒度审批能力的 Provider 显式降级，而不是伪装支持；
- 手机断线后由 Host 独立继续任务，并在重连后按事件序号补传；
- 最终确认前展示完整 Diff、验证结果和交付计划；
- 由 Host 完成 Git/GitHub 写操作，并且只创建 Draft PR；
- 将 GitHub CI 与 Review 状态回流到 Host 和 Android。

## 系统架构

```text
┌──────────────────────────────────────┐
│ Android Client                       │
│ 任务 · 事件 · Diff · 审批 · 最终确认 │
└──────────────────┬───────────────────┘
                   │ 设备配对 + HTTPS/WSS
                   │ LAN / 用户自建 VPN
                   │ （未来：可选 E2EE Relay）
                   ▼
┌──────────────────────────────────────────────────────┐
│ 用户自己的 Host                                      │
│                                                      │
│ Shiki Daemon                                         │
│ 连接与设备认证   任务编排   策略/能力判断             │
│ 本地状态文件     JSONL 事件  Provider Adapter         │
│ Git/worktree     GitHub 集成 凭据与发布控制           │
└───────────────┬───────────────────────┬───────────────┘
                │                       │ Host 持有凭据并执行
                │ 创建/挂载 task        │ clone/fetch/worktree/
                │ worktree              │ commit/push/Draft PR
                ▼                       ▼
┌──────────────────────────────┐   ┌────────────────────┐
│ 一次性 Task Container        │   │ GitHub             │
│ 真实 Coding CLI              │   │ Issue · PR · CI    │
│ 项目工具链 · 构建 · 测试     │   │ Review             │
│ 无 GitHub Token / Docker API │   └────────────────────┘
└──────────────────────────────┘
```

Host 是控制面和安全决策点。它创建 worktree、启动和销毁容器、保存 Timeline、管理 Provider 会话，并负责所有需要 GitHub 凭据的操作。任务容器只获得任务工作目录、必要工具和受控配置；它可以修改工作树，但不负责 `commit`、`push` 或创建 PR。Docker 是 MVP 容器任务的必需能力；不可用或未通过策略检查时，Daemon 应拒绝任务，不静默改为直接在 Host 运行仓库代码。

## 主流程

```text
GitHub Issue / 用户任务
  → Host 获取仓库并创建独立 worktree
  → Host 启动一次性 Task Container
  → Shiki Daemon 通过 Provider Adapter 驱动容器中的真实 Coding CLI
  → Agent 修改代码，Host 收集事件、Diff 与测试结果
  → Android 展示结果，用户作最终确认
  → Host 创建提交、推送任务分支并创建 Draft PR
  → GitHub CI / Review 状态回流 Host 与 Android
```

用户拒绝最终确认时，Host 不推送任务改动，也不创建 PR；Task 以“本地结果、未发布”完成。后续是否保留 worktree、基于该结果创建关联的新 Task 或清理，由用户在 Host 策略允许的范围内决定。

## Provider Adapter 与能力降级

不同 Coding CLI 的公开接口、事件质量和恢复能力差异很大。Shiki 按以下顺序选择集成方式：

1. **官方结构化协议：** 优先使用 Provider 明确支持的 SDK、ACP、JSON-RPC 或等价协议；
2. **CLI JSON/JSONL：** 在官方 CLI 提供有文档、可版本约束的机器可读输出时使用；
3. **PTY 兼容：** 最后才以伪终端驱动交互式 CLI，作为兼容路径。

每个 Adapter 必须声明经过验证的能力，例如结构化流、工具调用事件、执行前审批、取消、会话恢复和用量统计。Host 只启用实际可用的功能：

- 不声称所有 CLI 都提供完整 API；
- 不把终端文本解析包装成“可靠审批”；
- PTY 或事件不完整时，只提供任务启动、容器权限、网络策略和最终发布等外层控制；
- 如果无法在工具执行前确定性拦截，就禁用对应细粒度审批界面，并明确告知用户；
- Adapter 失去协议兼容性时应快速失败，不能静默放宽权限。

无论使用哪一层，安全边界都不能只依赖 Provider 自报事件或 Agent 自觉遵守提示词。

## 任务生命周期与恢复

```text
CREATED
  → QUEUED
  → PREPARING
  → RUNNING
      ↔ WAITING_FOR_APPROVAL（仅在可靠支持时）
  → VALIDATING
  → WAITING_FOR_FINAL_CONFIRMATION
      ├──确认──▶ PUBLISHING → COMPLETED
      └──拒绝──▶ COMPLETED（本地结果，不发布）

任意非终态 → FAILED / CANCELLED / INTERRUPTED
```

GitHub Draft PR 创建并持久化链接后，Task 可以进入 `COMPLETED`；用户拒绝发布时也以“本地结果、未发布”完成。后续 Checks、Review、Merged 或 Closed 作为独立 Delivery 投影继续同步，不长期占用 Host 的唯一执行名额。

- **手机断线：** 不会中止 Host 上已运行的任务；不需要用户输入时任务继续，等待确认时保持等待状态。Android 重连后按任务事件序号补取遗漏事件。
- **Host 关机、休眠或 Daemon 停止：** 任务不能继续运行，并进入中断或待恢复状态。
- **Host 恢复后：** Shiki 可以从本地状态、JSONL 日志、worktree 和产物重建任务视图，但 Provider 会话能否原地恢复取决于其协议和 CLI。无法恢复时只能启动新会话继续现有工作树或由用户重试，不能承诺无缝续跑。
- **任务取消：** Host 可以停止容器并保留已有日志与 Diff；Provider 是否支持优雅取消由 Adapter 能力决定。

## 本地持久化

首版不使用 PostgreSQL 或其他中心数据库。计划中的 Host 数据布局概念如下，实际路径和格式会在实现时版本化：

```text
~/.shiki/
├── config/                 # Host 与已配对设备配置
├── projects/               # 本地项目元数据
├── tasks/<task-id>/
│   ├── state.json          # 可重建的任务状态快照
│   ├── events.jsonl        # 只追加事件流
│   └── artifacts/          # Diff、测试摘要等任务产物
└── worktrees/<task-id>/    # 每任务独立 worktree
```

这是便于说明的数据概念图；Linux/WSL2 的实际实现将按 XDG 将配置、持久数据、运行状态、缓存和临时锁分开，具体路径以[总体设计文档](docs/DESIGN.md)与平台路径模块为准。事件写入需要递增序号、格式版本和崩溃恢复策略。快照是读取优化，JSONL 事件和工作区共同用于恢复与审计。GitHub Token、Provider 凭据等秘密不得写入任务 JSONL 或普通状态文件。

## 安全模型与限制

Shiki 将仓库内容、依赖安装脚本、Agent 输出和任务容器内进程都视为不可信输入。

- **设备身份：** Android 通过一次性配对建立设备密钥；私钥保存在 Android Keystore，Host 维护可撤销的配对设备列表。Shiki 不依赖中心账号判断设备权限。
- **传输边界：** 优先在可信 LAN 或用户自建 VPN 内连接，并使用 HTTPS/WSS。不要将未加固的 Daemon 直接暴露到公网。
- **Host 权威：** 容器和 Agent 不能自行批准权限、发布代码或修改 Host 策略。最终确认必须由 Host 验证已配对设备的请求。
- **GitHub 凭据：** GitHub Token、OAuth 凭据或 `gh` 登录信息只保留在 Host，绝不注入任务容器。clone/fetch、分支、提交、push 和 Draft PR 操作均由 Host 侧受控代码完成。
- **Provider 凭据：** 真实 Coding CLI 可能要求 API Key、OAuth Token 或会话文件进入同一个任务容器。即使使用短时注入、只读挂载或最小权限，恶意仓库代码或同容器进程仍可能窃取这些凭据。容器不能隔离“同容器内的凭据与代码”；应优先使用可撤销、短期、低权限凭据，并避免复用高价值账号会话。
- **容器隔离：** 每任务使用一次性、非 root、受资源限制的容器，不挂载 Docker Socket、Host 凭据目录或无关路径，并尽量限制网络出口。
- **隔离声明：** Task Container 用于缩小错误或恶意操作的影响范围，不是虚拟机或硬件级绝对安全边界；Docker、内核、挂载和配置漏洞仍可能影响 Host。
- **审批边界：** 只有 Provider 协议提供可靠的执行前钩子时，才能把细粒度工具审批作为控制措施。否则以容器权限、Host 操作边界、网络策略和最终确认兜底。
- **交付边界：** 用户最终确认前不向 GitHub 推送任务修改；确认后只创建 Draft PR，永不自动合并。
- **日志与脱敏：** 日志应避免记录秘密，并对常见 Token 和私钥模式做尽力脱敏；脱敏不能保证识别所有敏感数据。

运行 Shiki 意味着信任自己的 Host 操作系统、Docker/WSL2 环境、所选 Provider 及其凭据机制。对高风险仓库，应使用单独设备、虚拟机或更强隔离，而不是仅依赖任务容器。

## 计划技术栈

| 层级 | 计划技术 |
| --- | --- |
| Android | Kotlin、Jetpack Compose、Android Keystore |
| Shiki Daemon | TypeScript、Node.js、本地 HTTPS API 与任务编排 |
| 实时通信 | HTTPS、WSS、递增事件序号与断线补传 |
| 本地存储 | 版本化 JSON、追加 JSONL、本地文件系统 |
| 任务执行 | Git worktree、Docker Engine / Docker Desktop、一次性容器 |
| Provider 集成 | 官方 SDK/ACP/JSON-RPC → CLI JSON/JSONL → PTY |
| GitHub 集成 | Host 侧 Git、GitHub API/CLI、Draft PR、CI/Review 状态同步 |
| API 契约 | OpenAPI、版本化 JSON Schema |
| 工程组织 | pnpm workspace、Gradle |
| CI | GitHub Actions、Host/Android 测试、任务镜像构建 |

技术选型仍需通过探针确认；表中内容是实现方向，不代表对应能力已经完成。

## 第一阶段技术验证

完整产品开发前，项目将使用一个真实 Provider 完成端到端探针，重点验证：

- 在任务容器中启动、交互和终止真实 Coding CLI；
- 官方结构化协议是否稳定提供消息、工具事件、退出状态和错误；
- 执行前审批是否确实发生在工具运行之前，而不是事后通知；
- CLI JSON/JSONL 的版本、错误流、部分行和异常退出行为；
- PTY 模式对提示、确认、终端尺寸、颜色和断线的兼容性上限；
- 取消、超时、Daemon 重启和 Provider 会话恢复的实际语义；
- worktree、一次性容器、资源限制和清理流程；
- GitHub Token 不进入容器，Provider 凭据暴露范围符合明确声明；
- Android 断线重连、事件补传和重复请求幂等性；
- 最终确认之前无法 push，确认后只能创建 Draft PR；
- 在 Windows 11 + WSL2/Docker Desktop 与原生 Ubuntu 上行为一致。

探针结果将形成 Adapter 能力矩阵。结构化接口不足时依次降级到 JSON/JSONL 和 PTY；任何无法可靠实现的审批、恢复或用量能力都应在 MVP 中关闭并标注，而不是通过脆弱解析制造保证。

## 路线图

1. **设计与工程基线**
   - 固化文档、Monorepo 骨架、固定工具链、Schema 和测试入口。
2. **Provider、Git 与容器探针**
   - 验证一个真实 Provider、三层 Adapter、worktree、容器、凭据与恢复边界。
3. **Daemon 单任务内核**
   - 本地 JSON/JSONL、状态机、worktree、一次性容器、Diff/测试和清理。
4. **设备配对与本地协议**
   - Host 身份、设备撤销、HTTPS/WSS、Timeline 补传、幂等和离线语义。
5. **GitHub 交付闭环**
   - Issue 输入、Host 凭据隔离、最终确认、Draft PR、CI/Review 回流。
6. **Android MVP**
   - 多 Host、任务创建、Timeline、Diff、能力提示、最终确认和通知。
7. **跨平台与安全加固**
   - Windows 11 + WSL2/Docker Desktop 与原生 Ubuntu 的安装、升级、崩溃恢复和安全测试。
8. **后续能力评审**
   - MVP 稳定后评估更多 Provider、多任务并发、可选 E2EE Relay 和轻量编辑器。

## 计划中的仓库结构

```text
shiki/
├── apps/
│   ├── android/                 # Android 移动控制端
│   └── daemon/                  # Host 上的 Shiki Daemon
├── packages/
│   ├── protocol/                # OpenAPI、JSON Schema 与生成模型
│   ├── daemon-core/             # 任务、Timeline、确认和恢复状态机
│   ├── host-platform/           # 路径、进程、文件、Git 与 Docker 适配
│   ├── provider-adapters/       # 结构化、JSON/JSONL、PTY Adapter
│   ├── github/                  # Host 侧 GitHub 集成
│   └── security/                # 配对、策略、路径、脱敏和密钥扫描
├── tools/
│   ├── provider-probe/          # 真实 Provider 能力探针
│   └── protocol-probe/          # 配对、WSS 与 Timeline 测试客户端
├── container/
│   └── task/                    # 一次性 Task Container 镜像
└── tests/
    └── fixtures/
        ├── sample-repository/   # 正常执行与交付样本
        └── hostile-repositories/ # 恶意仓库安全回归样本
```

该结构是规划，不表示目录和实现已经存在。项目不计划增加中心账号服务、中心业务数据库或共享任务 Worker 集群。

## 开发环境

MVP 的开发与验收矩阵：

### Windows Host

- Windows 11；
- WSL2 Ubuntu；
- Docker Desktop，启用 WSL2 Backend；
- Shiki Daemon 运行在 WSL2 Linux 环境中；
- Android Studio 与 Android 模拟器或实体设备。

### Ubuntu Host

- 受支持的 Ubuntu LTS；
- Docker Engine；
- Git、Node.js LTS 与 pnpm；
- Android 客户端通过 LAN、VPN 和 WSS 连接。

### 通用要求

- Android Studio、JDK 与 Android SDK；
- 独立的 GitHub 测试仓库和最小权限凭据；
- 用于探针的真实 Provider 账号与可撤销凭据；
- 测试环境与个人日常凭据、生产仓库分离。

当前仓库仍处于设计阶段，尚无可运行的安装或启动命令。实现建立后将补充可复现的环境检查、构建、测试和卸载流程。

## 项目状态

- [x] 确认本地优先、个人自托管的产品定位
- [x] 确认 Android 控制端、Host 执行端与 GitHub Draft PR 主流程
- [x] 确认本地 JSON/JSONL、每任务 worktree 与一次性容器方向
- [x] 明确 Provider Adapter 降级策略和安全声明边界
- [ ] 完成一个真实 Provider 的协议与容器探针
- [ ] 实现 Host 单任务执行内核
- [ ] 实现 GitHub 最终确认与 Draft PR 闭环
- [ ] 实现 Android 配对与核心任务流程
- [ ] 完成 Windows/Ubuntu 恢复与安全验收

## 文档

- [总体设计文档](docs/DESIGN.md)：产品边界、架构、安全模型、状态与路线图；
- [开发工具与环境配置清单](docs/DEVELOPMENT_SETUP.md)：Windows/WSL2、Ubuntu、Android、Provider 与容器开发基线。

三份文档共同采用本地优先的 Personal daemon + Container runner + GitHub loop 基线；若后续决策改变产品或安全边界，应在同一变更中同步文档并补充 ADR。

## Contributing

项目处于早期设计与验证阶段，外部贡献流程尚未定型。在提交实现前，请先通过 Issue 说明目标、威胁边界和验证方式；涉及 Provider 能力时必须附带真实探针结果，不能仅依据 CLI 外观或未验证假设声明支持。

## License

许可证尚未确定。在许可证文件正式加入仓库前，不授予复制、分发或再许可本项目代码的权利。
