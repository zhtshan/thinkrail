# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在本仓库中工作时提供指引。

## 从这里开始

[`AGENTS.md`](AGENTS.md) 是本仓库权威的代理契约——模块边界、仓库级不变量、验证策略和交接规则。
它会在本项目中自动加载；修改任何内容之前请先阅读。本文件是速查地图：命令、架构，以及持久知识存放的位置。

顶层规范（另有就近放置的 `SPEC.md` 文件，每个模块及每个有意义的子模块各一份）：

- [`goal-and-requirements.md`](goal-and-requirements.md) — 产品目标、原则、能力、非目标
- [`architecture.md`](architecture.md) — 拓扑、编号的跨模块决策、依赖边
- [`pi-compatibility.md`](pi-compatibility.md) — 针对已安装 `pi` SDK 的宿主适配及其移除条件
- [`central-integration.md`](central-integration.md) — JetBrains Central 集成

## 命令

Bun 1.4.4 是锁定的包管理器/运行时；进程内 `pi` 引擎要求 Node >= 22.19。

```bash
bun install
bun run dev          # 同时启动宿主和 Web 客户端，并打开浏览器
bun run dev:server   # 仅宿主（packages/server）
bun run dev:web      # 仅 Web 客户端（apps/web）
```

构建与启动器：

```bash
bun run build              # 跨 workspace 的 turbo 构建
bun run build:web          # 仅 Web 包（先运行 token 检查）
bun run build:binary       # 独立 CLI 产物
bun run desktop:dev        # 打包并打开 Electrobun 应用
bun run desktop:build      # 打包但不打开
bun run smoke:desktop      # 桌面构建后的原生冒烟测试
bun run smoke:desktop:installer <path> <stable|canary>
```

### 快速门禁（pre-commit 钩子会运行这些）

```bash
bun run lint              # biome check .
bun run typecheck         # 跨 packages + root + e2e 的 tsc
bun run test              # 根脚本测试 + turbo run test --continue
bun run check:deps        # 精确版本 / catalog 一致性
bun run check:boundaries  # 模块依赖边界
bun run check:seams       # 编译二进制的打包器接缝金丝雀检查
bun run check:spec-surface # 已登记的 SPEC 公共接口面
bun run check:compiler    # React Compiler 普查与已提交基线对比
bun run check:i18n        # 禁止硬编码 UI 文本，校验 catalog 完整性
```

`bun run check:pi-packages` 还会打包可发布的 `pi-*` 包并在原生 Node 下运行——当 `pi-extensions/*`
或 `packages/pi-*` 包发生变更时必须执行。

### 运行单个测试

```bash
# 单个单元测试文件（bun 按路径过滤）
bun test packages/contracts/src/match.test.ts
bun test packages/server/src/host/dialog.test.ts

# 单个包的完整测试套件
bun run --filter @thinkrail/contracts test
bun run --filter @thinkrail/server test
```

### 浏览器 E2E

先执行一次 `bunx playwright install chromium`，然后：

```bash
bun run e2e                                  # 完整的无代理门禁（分片）
bun run e2e -- e2e/changes.spec.ts           # 聚焦迭代
bun run e2e -- --last-failed                 # 修复循环
bun run e2e:serial                           # 单通道调试兜底
bun run e2e -- --shards=12                   # 显式指定 1-16 个通道
bun run e2e:binary                           # 打包后的 CLI 宿主
bun run e2e:desktop                          # 打包后的桌面宿主
bun run e2e:agent                            # 仅 @agent 规格，真实 provider，串行
bun run e2e:full                             # 全部；会消耗真实 provider token
bun run e2e:full e2e/agent-naming.live.spec.ts   # 单个聚焦的 live 规格
```

在整体实现完成后、交接之前完整运行一次 `bun run e2e`，而不是每次改动都运行。`bun run e2e:full` 和
`bun run test:workflows` 会消耗真实 provider token——需先询问。`bun run test:workflows` 是按需使用的测试工具，
不是提交门禁。

## 架构

三个环；线协议（wire）是它们之间唯一的耦合。

```
apps/cli, apps/desktop     宿主库之上的薄启动器
packages/server            引擎宿主：Bun.serve(HTTP+WS) + 进程内 pi（createAgentSession）
packages/shared            服务端 shell 环境
packages/contracts         类型化、带版本的线协议——WS 方法/频道、协议版本、守卫
apps/web                   移动优先的 React UI，由传输层驱动，以静态资源发布
packages/locales           UI 字符串目录 + 唯一的 i18next 实例
packages/ui                自有的 shadcn/Radix 原语、cn()、onThemeSwap
packages/resource-renderers 文件/diff 渲染器 + 审阅界面
packages/extension-api     扩展两半的类型 + define* 辅助函数
thinkrail-extensions/*     pi 能力 + ./server 和 ./web 两半，每侧由一个注册表组合
pi-extensions/*            可移植的 pi 包，以 @thinkrail/pi-<name> 发布到 npm
packages/pi-*              可移植的纯 pi 包，由宿主打包进每个会话
```

- `pi` 是唯一的引擎，且**在进程内**运行——绝不作为子进程，绝不引入第二个运行时。
  `pi` 拥有模型、技能、压缩、成本和会话状态；宿主只暴露 `pi` 报告的内容。
- **先水合再流式（Hydrate-then-stream）**：每个客户端先*读取*当前状态（`session.list`、`session.getMessages`、
  `project.list`、`workspace.list`），然后订阅实时增量。客户端是领域状态的无状态投影，绝不是第二个事实来源。
- **Workspace 即 git worktree**：project（仓库）→ workspace（独立分支 + 位于 `~/.thinkrail/worktrees` 下的 cwd）
  → {chats, files, terminals}。两个用户自有的例外在线协议上用 `kind` 标记：内置的 `kind: "default"`
  workspace（项目文件夹本身）和已挂载的 `kind: "external"` worktree。
- **自动工作结束于 `agent_settled`，绝不是 `agent_end`**——`agent_end` 只是一次尝试的边界；
  其后可能还有重试、压缩和排队的后续任务。
- 流式语义：`text_delta` / `thinking_delta` 为追加；`tool_execution_update.partialResult` 为替换。
  会话流式进行中 `prompt()` 会抛错——应使用 `steer()` / `followUp()`。
- 服务端能力与其 UI 渲染器通过 `registerToolRenderer`（`apps/web/src/chat/toolRegistry.tsx`）按工具名关联；
  未注册的工具回退到默认渲染器。

## 违反即导致构建失败的规则

- `apps/web` 只能依赖 `contracts`、`locales`、`ui`、`extension-api/web`、`resource-renderers` 和
  `thinkrail-extensions/*/web`——绝不能依赖 `server` 或 `shared`。浏览器代码绝不能值导入 `pi`；
  只能从 `pi-ai` / `pi-agent-core` 包根导入类型。
- 子模块以 `index.ts` barrel 作为唯一公共接口；同级模块通过 barrel 导入。仅 `apps/web/src/panels`、
  `apps/web/src/chat` 和 `packages/ui` 有意使用按文件导入（代码分割 / 库惯例）。
- 规范先行于代码：凡是移动或模糊了边界、契约、不变量或决策的改动，先更新所属的 `SPEC.md`，再改代码。
- 面向用户的 UI 文本必须是 `packages/locales/en.json` 中的 key，并通过 `@thinkrail/locales/react` 渲染。
  不允许硬编码 JSX 文本或文本属性（由 `check:i18n` 强制）。
- 样式使用基于生成的语义 CSS 变量 token 的 Tailwind v4 工具类。禁止内联 style 对象、原始十六进制色值、
  调色板名称或未知的 token 工具类。token *源文件* 是 `apps/web/src/styles/{typography,colors,spacing}.json`；
  使用 `bun run typography:generate` / `colors:generate` / `spacing:generate` 重新生成，并用对应的 `:check`
  变体校验。`styles/generated/` 是生成产物——编辑源 JSON，绝不编辑产物。做颜色/间距相关工作时请阅读
  `apps/web/src/styles/COLOR.md` 和 `SPACING.md`。
- 图标使用 `@remixicon/react`（默认 Line，激活态用 Fill）；原语来自 `@thinkrail/ui/<primitive>`；
  `cn()` 位于 `@thinkrail/ui/utils`。
- 错误仅以分类信息（类型、`code`、scope/method、仅 ThinkRail 的栈帧）上报到 PostHog Error Tracking——
  绝不上报 `message` 文本。宿主诊断使用 `logger(scope)`，绝不使用 `console.*`。
- 绝不为了让门禁通过而添加 `biome-ignore`、`@ts-expect-error`、`@ts-ignore`、`eslint-disable` 或 `as any`。
  确实需要的抑制须先获得批准。
- 依赖固定精确版本；跨领域的依赖在根 `workspaces.catalog` 中只声明一次，并以 `catalog:` 引用。
  只允许 `@earendil-works/*` scope。

## 接下来看哪里

| 问题 | 文件 |
| --- | --- |
| 这个模块负责什么？ | 该模块的 `SPEC.md`（frontmatter：`id`、`type`、`parent`、`depends-on`） |
| 包之间允许哪些依赖边？ | `architecture.md` 拓扑 + 父模块的 `SPEC.md` |
| 线协议如何工作？ | `packages/contracts/src/wsProtocol.ts`（`WS_METHODS`、`WS_CHANNELS`） |
| UI 如何组合？ | `apps/web/src/main.tsx` → `shell/Shell.tsx`；面板与布局无关 |
| 扩展如何接入？ | `packages/server/src/extensions/registry.ts` 和 `apps/web/src/extensions/registry.ts` |
| 工具卡片如何渲染？ | `apps/web/src/chat/toolRegistry.tsx` + `apps/web/src/chat/tools/register.ts` |
| 会话中的 `pi` 上运行了什么？ | `packages/server/src/agent/extensions.ts`（打包的 pi 扩展 + 技能） |
