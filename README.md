# dsh-plugin-codex-annotate · 划词引用回复

[![license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

给 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的 Web UI 加一个 Codex 风格的「划词追问」：选中助手回复里的任意文字，右键选择 **在此回答**，写下你的问题或补充；可以连续攒下多条注释，最后一次性发出。发出的消息自带精确出处（消息 id、轮次、段号、小节、字符区间或代码行号）和原文，并在气泡里折叠成一条 `📎 N 条注释` 的附件条。

> 面向 DSH **0.2.1-rc.2** 的 Web 客户端构建，依赖若干尚未承诺稳定的 DOM / React 内部细节（见[兼容性](#兼容性与已知限制)）。

## 特性

- **划词右键即用**：右键菜单里新增「在此回答」，与「复制」并列；没有选中文字时不拦截系统菜单。
- **精确定位**：消息 id、轮次、段号、所在小节标题、字符区间；代码块记录语言与行号。
- **先确认、后统一发送**：每次确认只暂存一条注释，输入框工具行显示 `📎 N 条注释`，点开可逐条删除或清空，最后一次性发送。
- **气泡内折叠**：发出的消息把引用部分收成 `📎 N 条注释 ▾`，点击展开原文，展开后仍可正常划选复制。
- **纯本地**：没有任何网络请求或遥测；引用内容就是一条普通用户消息，fork / 回放 / 搜索 / 上下文压缩都照常工作。

## 环境

- DeepSeek Harness **0.2.1-rc.2**（Web UI 或桌面版）
- 无构建步骤：仓库里的 `client.js` 就是浏览器直接加载的模块

## 安装

要求 DSH 已安装且 `dsh` 在 PATH 中。从 GitHub 直接安装：

```bash
dsh plugin --profile desktop add github:mar1259/dsh-plugin-codex-annotate
```

`desktop` 是桌面版的 profile 名；Web 版通常用 `web`，换成你实际启动的 profile 即可。
固定到某个 commit，避免仓库后续改动影响现有环境：

```bash
dsh plugin --profile desktop add github:mar1259/dsh-plugin-codex-annotate#<commit-sha>
```

本地 clone 安装：

```bash
git clone https://github.com/mar1259/dsh-plugin-codex-annotate.git
dsh plugin --profile desktop add ./dsh-plugin-codex-annotate
```

卸载：

```bash
dsh plugin --profile desktop remove dsh-plugin-codex-annotate
```

本仓库是纯 JS，没有 `prepare` / `build` 脚本，git 安装不需要 pnpm 的 `allowBuilds` 许可。

> 安装或更新客户端 bundle 后，刷新页面（或重启 DSH）才会加载新的 JS 代际。

## 使用

1. 在助手回复中选中一段文字 → **右键** → `❝ 在此回答`。
2. 在弹出的输入框里写下回复 → 点 **确认**（或按 Enter）。
3. 继续划词、继续确认；输入框工具行左侧出现 `📎 N 条注释` 胶囊。
4. 点胶囊 → 浮层里可逐条删除、清空，或直接 **发送**。
5. 发送后，所有注释与回复合成**一条**用户消息；气泡里显示为 `📎 N 条注释 ▾`，点击展开。

## 发出的消息长这样

段落选区：

```text
> **回复引用** · 助手消息 #4f2a1c · 第 3 轮 · 第 2 段「方案 A」· 字符 12–58
> 「建议把鉴权放到网关层，避免每个服务重复实现。」

那网关自己挂了怎么办？
```

代码选区：

```text
> **回复引用** · 助手消息 #4f2a1c · 第 3 轮 · 代码块 ts · 第 12–18 行
> ```
> for (const x of xs) { ... }
> ```
```

多条注释会按顺序拼接在同一个消息里，每条一个引用块 + 一段回复。

## 工作原理

| 部件 | 挂载点 | 作用 |
| --- | --- | --- |
| 右键菜单 + 引用编辑器 + 注释列表 | `shell.overlay` | 全帧浮层：拦截助手选区上的 `contextmenu`，渲染菜单/编辑器/列表 |
| 待发送胶囊 | `conversation.input.left` | 输入框工具行左侧，显示 `📎 N 条注释` |
| 发送桥 | `conversation.input.overlay` | 从会话作用域拿到 `inputActions`，把文本写入输入框并提交 |
| 气泡折叠 | 注入 CSS + DOM 标记 | 给用户气泡内的引用 `<blockquote>` 打标记，折叠成附件条 |

- 消息身份不依赖 CSS 类名：从选区节点沿 React fiber 链读取 `sessionId` / `messageId` / 轮次，读不到时退回 `data-chat-flow-kind` 属性。
- 发送走宿主自己的输入管线：`insertText`（追加到草稿末尾）或 `setDraft`，随后 `submit()`，因此排队、steer、附件等行为与手动发送一致。
- 引用内容始终是消息正文的一部分，没有被藏进自定义事件；`<blockquote>` 折叠只是视觉呈现。

## 兼容性与已知限制

- 依赖 DSH 客户端内部结构：`__reactFiber$*`、`data-chat-flow-kind` / `data-chat-turn`、用户消息里 `<blockquote>` 的渲染方式。**升级 DSH 后可能需要适配**。
- 目前只支持助手回复正文；跨消息选择、工具调用卡片、流式输出中的消息不会出现菜单项。
- 待发送注释只存在内存里，刷新页面会清空。
- 在助手选区的右键菜单会**替换**系统菜单，当前只提供「复制」与「在此回答」两项；其他区域仍使用系统菜单。
- 长引用会截断：正文 300 字、代码 12 行，超出会标注「已截断」。
- 若输入框在确认到发送之间被改动，发送会中止并提示重试，避免覆盖你的草稿。

## 隐私

插件不做任何网络请求，不收集遥测，也不读写工作区文件。引用的原文与你的回复会作为普通消息文本，经 DSH 原有的发送链路交给模型。

## 开发

仓库即包目录，没有构建产物：

```text
client.js          # 浏览器半边：菜单、编辑器、注释列表、气泡折叠
index.js           # Host 半边：空 apply，仅用于打包
cordis.patch.yml   # Bundle 补丁：插入一行 codex-annotate
package.json       # dsh.bundle / dsh.client 清单
locale/{zh,en}.json# 插件管理器显示的标题与描述
icon.svg           # 插件图标
```

语法自检：

```bash
node --check client.js
node --check index.js
```

## 贡献者

- [mar1259](https://github.com/mar1259) — 作者、产品方向与测试
- **DeepSeek Harness Agent** — AI 结对实现（架构、客户端代码、文档）

提交记录中使用 `Co-authored-by: DeepSeek Harness Agent <agent@dsh.invalid>` 标注 AI 协作；这是保留域名的占位邮箱，不对应任何公开账号。

## License

[MIT](LICENSE) © 2026 mar1259

---

## English

Codex-style selection annotations for the DeepSeek Harness Web UI. Select text in an assistant reply, right-click, choose **Reply here**, write your note, and confirm — accumulate several notes as a `📎 N notes` chip inside the composer tool row, then send them all at once. The outgoing message quotes the exact location (message id, turn, block, section, character range or code lines) and is folded into an expandable attachment row inside the chat bubble.

Install:

```bash
dsh plugin --profile desktop add github:mar1259/dsh-plugin-codex-annotate
```

Tested against DSH 0.2.1-rc.2. The plugin relies on client internals (`__reactFiber$`, `data-chat-flow-kind`, rendered blockquotes), so DSH upgrades may require adaptation. No network access, no telemetry. MIT licensed.
