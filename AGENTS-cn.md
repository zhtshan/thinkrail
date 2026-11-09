# ThinkRail

会随使用不断变好的代理式 IDE，围绕 `pi` 编码代理构建。应用本身是一个轻量宿主：运行 `pi` 并将其桥接到丰富的 UI；
`pi` 负责模型、技能、压缩、成本和会话状态。

## 按需阅读上下文

- 产品目标、原则、能力和非目标，参阅 `goal-and-requirements.md`。
- 系统拓扑、跨模块决策和仓库级不变量，参阅 `architecture.md`。
- 升级 pi 或在 pi 周围做宿主变通时，阅读 `pi-compatibility.md`：根据已安装的 SDK 核对每一个适用的移除条件，
  更新注册表，并报告移除或保留了什么。
- 当工作受某个模块边界、契约、不变量、已记录的行为或架构决策约束，或可能改变它们时，阅读所属的 `SPEC.md`。
- 局部性工作不需要阅读无关的规范，也不需要完整的仓库地图。

## 模块结构与边界

清晰、分形的模块边界是最高优先级要求：

- 每个包和有意义的目录级子模块都有一份 `SPEC.md`，说明其职责、公共接口、允许的依赖和禁止的越界访问。
- 子模块只通过 `index.ts` barrel 暴露公共接口；同级模块通过 barrel 导入，绝不导入内部实现。仅当 barrel 会破坏
  代码分割或库惯例时才允许按文件导入，例如 `apps/web/src/panels` 和 `packages/ui`。
- 同级子模块之间的依赖边记录在父模块的 `SPEC.md` 中，而不是每个叶子模块里。
- 凡是移动或模糊了边界的改动，先更新所属规范。在可行时用测试覆盖公共接口和边界规则，但不要为局部改动人为凑覆盖率。

## 引擎与架构

- 通过 `@earendil-works/pi-coding-agent`（`createAgentSession`）在进程内运行 `pi`，绝不作为子进程运行，也绝不通过第二个
  代理运行时运行。致命的 provider/代理故障可能会拖垮宿主；这种缺乏崩溃隔离的状况是可接受的。
- 三个环分别是：引擎宿主（`packages/server` + `packages/shared`）、类型化的线协议（`packages/contracts`）、
  可独立发布的 UI（`apps/web`）。`apps/cli` 和 `apps/desktop` 是同一宿主之上的薄启动器。
- 只使用 `@earendil-works/*` 包 scope。`@mariozechner/*` 已弃用。

## 仓库级不变量

- `apps/web` 只能依赖 `contracts`、`locales`、`ui`、`extension-api/web`、`resource-renderers` 和
  `thinkrail-extensions/*/web`，绝不能依赖 `server` 或 `shared`。
- 面向用户的 UI 文本是 `packages/locales/en.json` 中的 key，并通过 `@thinkrail/locales/react` 渲染；
  lint 门禁会拒绝硬编码的 JSX 文本和文本属性。
- 浏览器打包的代码中绝不值导入 `pi`。只能从 `pi-ai` / `pi-agent-core` 包根导入类型；
  `@earendil-works/pi-coding-agent` 仅限服务端。
- 一套 id 模型：UI 标签页 id 对应 `session.sessionId`；不存在单独的 pi UUID。
- `pi` 拥有状态。宿主只暴露 pi 报告的内容，不重新计算成本、统计数据或状态。
- 流式语义：`text_delta` / `thinking_delta` 为追加；`tool_execution_update.partialResult` 为替换。
- 会话流式进行中 `prompt()` 会抛错；应使用 `steer()` / `followUp()`。错误应通过事件流和方法抛出的结果向上传递，
  而不是当作崩溃信号处理。
- 自动工作结束于 `agent_settled`，绝不是 `agent_end`；在尝试级别的 `agent_end` 之后，可能还会有重试、压缩、恢复
  或排队中的后续任务。
- UI 面板与布局无关；由外壳（shell）负责排布。
- 错误无条件上报到 PostHog Error Tracking，但仅作为分类信息上报——类型、`code`、
  来源/scope/请求方法以及仅 ThinkRail 的栈帧——绝不上报 `message` 文本。值得区分的失败应使用 `CodedError` 的 code，
  而不是依赖其 message。携带错误的宿主诊断通过 `logger(scope)` 输出，而不是 `console.*`。允许列表位于
  `packages/server/src/analytics/SPEC.md`。
- 传输层的宿主端点是一个参数，默认同源；`server.welcome` 携带协议版本。

### Web UI 上下文

做 Web UI 工作时，阅读 `apps/web/SPEC.md` 和所属的子模块规范。样式使用映射到生成的语义 CSS 变量 token 的
Tailwind v4 工具类：绝不使用内联 style 对象、原始十六进制色值、内部调色板名称或未知的 token 工具类。颜色工作阅读
`apps/web/src/styles/COLOR.md`，间距工作阅读 `apps/web/src/styles/SPACING.md`。使用 `@remixicon/react` 图标
（默认 Line，激活时用 Fill），以及来自 `@thinkrail/ui/<primitive>` 的自有 shadcn/Radix 原语；`cn()` 位于
`@thinkrail/ui/utils`。这些规则同样适用于 `packages/ui`、`packages/resource-renderers` 和扩展的 web 部分。

涉及对话渲染或工具展示时，阅读 `apps/web/src/chat/SPEC.md`。展示型渲染器保持由 props 驱动；只有 `ChatView` 集成
store 和传输层。服务端能力与其 UI 渲染器通过 `registerToolRenderer` 按工具名关联；未注册的工具使用默认渲染器。

## 规范与注释

规范是意图、决策、不变量、权衡和复盘的持久载体。保持简洁，避免重复代码或另一份规范的内容。注释应极少：
lint/类型指令以及偶尔一行的风险提示可以接受；理由和叙述属于所属的规范。

## 验证

本地测试使用一次性夹具，不能访问生产环境。运行受影响的测试，修复由所请求改动引起的失败，并在每一步无需请求批准即可重新运行。

- 以最小的相关单元测试或聚焦的 E2E 目标进行迭代。
- 对于已发布应用的行为或集成类改动，在整体实现完成后、最终交接或提交 PR 之前，运行一次完整的无代理浏览器套件
  （`bun run e2e`）——而不是每个 TODO 项或每次提交都运行。
- 当改动涉及真实代理行为时，还需运行覆盖该行为的聚焦 live 规格，例如
  `bun run e2e:full e2e/agent-naming.live.spec.ts`；这无需批准。运行完整的 `bun run e2e:full` 套件前须先询问用户：
  它会消耗真实的 provider token，并且耗时长得多。
- 仅文档/规范的改动和仅测试工具的改动使用定向检查，除非它们可能影响已发布的运行时。
- 快速门禁：`bun run check:deps`、`bun run check:boundaries`、`bun run check:seams`、`bun run lint` 和
  `bun run typecheck`。单元测试为 `bun run test`；`bun run check:spec-surface` 校验已登记的规范/barrel 公共接口；
  `bun run check:compiler` 在某个 web 函数相对于已提交的普查基线新增退出 React Compiler（或开始被编译）时失败。
- `bun run test:workflows` 按需执行：它会消耗真实 provider token，不是提交/CI 门禁。
- 二进制和桌面产物模式有各自独立的门禁；改动这些产物时使用它们。

所有运行器模式、隔离保证、凭据处理、取消行为和调试命令位于 `e2e/SPEC.md`；工作流工具的细节位于
`e2e/workflows/SPEC.md`。

## 交接卫生

绿色门禁是必要的，但不充分：

- 本地交接前，审查与任务相关的工作树和提交。打开或更新 PR 前，审查相对于基础分支的完整分支 diff 加上工作树。
- 对于非平凡的实现，做一次减法审查：移除可避免的抽象、状态所有者、依赖、兼容层和回退逻辑。报告重大的移除，
  并论证保留的各层；不要为局部编辑增加仪式感。
- 绝不为了让门禁通过而仅仅添加 `biome-ignore`、`@ts-expect-error`、`@ts-ignore`、`eslint-disable` 或 `as any`。
  应把报错视为设计信号。若确实需要抑制，先获得用户的明确批准。
- 审计正在审查的 diff 范围内新增的注释和抑制项。不要让之前无关的分支历史污染与任务相关的本地交接。
- 集中处理重复的非平凡推导。在 web 应用中，派生的 store 状态属于 selectors，总是一起写入的数据应放在同一个原子 action 中。
- 替换某种模式或状态模型时，搜索所有旧的出现位置并迁移，或说明有意保留的部分。
- `README.md` 是面向用户的首页。对于新增、移除或重命名用户可见能力，或改变安装或分析行为的改动，在同一改动中审查其描述并
  更新受影响的内容；当所描绘的 UI 发生实质变化时，刷新 `.github/readme-assets/` 下的截图。
- 对批准范围内的安全清理予以执行。只有破坏性的、超出范围的或产品层面的决策才需要询问。
- PR 生命周期相关工作（包括截图要求和 PR 模板）使用 `shipping-a-pr` 技能。以编程方式创建 issue 时，复现所选的 issue
  模板，并传入其 frontmatter 中的标签。

## 技术栈

Bun + Turbo monorepo · TypeScript strict · React 19 + Zustand + Tailwind v4 · 进程内 `pi`
（Node >= 22.19）。应用状态位于 `~/.thinkrail` 下。

依赖固定精确版本。跨领域的依赖在根 `workspaces.catalog` 中统一固定，并通过 `catalog:` 引用；对等依赖（peer dependencies）
和本地协议是仅有的例外。决策理由见 `architecture.md` 的 Decision #10。
