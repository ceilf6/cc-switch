# 手动改 JSON 的日子结束了：CC Switch 账号切换机制剖析

## 一个每天都在发生的场景

你用 Claude Code 写了半天代码，额度快到了，想切到中转站的 key 继续干活。于是你打开 `~/.claude/settings.json`，把 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_AUTH_TOKEN` 改掉，保存，重启终端。

好，这才只是 Claude Code。你还在用 Codex，它的配置在 `~/.codex/config.toml` 里，格式是 TOML；Gemini CLI 的密钥又存在另一个 `.env` 文件里。三个工具，三种配置格式，三套文件路径。如果你手头有 3-5 个供应商（官方直连、中转站 A、中转站 B、公司内部网关……），每天在它们之间来回切换，这个"打开文件 → 改字段 → 保存 → 重启"的循环能让人崩溃。

更烦的是：你好不容易配好了 MCP 服务器、permissions 规则、各种插件，切了个供应商回来，发现它们全丢了——因为你上次改 settings.json 的时候不小心覆盖了。

CC Switch 就是为了终结这种日子而生的。

## CC Switch 是什么

一句话讲：CC Switch 是一个桌面 GUI 应用，让你用可视化界面管理 Claude Code、Codex、Gemini CLI、OpenCode、OpenClaw 和 Hermes Agent 这六款 AI CLI 工具的供应商配置。点一下切换，配置立刻生效，不用手动碰任何文件。

它用 Tauri 2 构建（Rust 后端 + React 前端），跑在 Windows、macOS 和 Linux 上，数据存在本地 SQLite 里。MIT 开源，目前版本 3.16.0，GitHub 上的 star 增长还挺快。

但今天不聊它的 UI 有多好看或者支持多少预设。我想聊一个更有趣的问题：**当你在 CC Switch 里点了"切换"按钮，它背后到底做了什么？**

## 切换的核心：一个 Provider 抽象统一六种格式

CC Switch 最聪明的设计在于它的数据模型。不管你用的是哪个 CLI 工具，每个"供应商"在它的 SQLite 数据库里都是同一种结构：

```
Provider {
    id: "deepseek-claude",
    name: "DeepSeek (Claude Code)",
    settingsConfig: { ... },  // 该应用原生配置格式的 JSON 快照
}
```

关键在 `settingsConfig` 这个字段。对于 Claude Code，它存的是 `settings.json` 应该长什么样；对于 Codex，它把 `auth` JSON 和 `config.toml` 拆开存成一个对象的两个字段；对于 Gemini，它存的是环境变量键值对。

也就是说，CC Switch 并没有发明一种"统一配置格式"——它尊重每个 CLI 工具自己的原生格式，只是把不同格式的快照统一管起来。这个思路很务实：不试图改变工具本身，只做一层外部管理。

## 普通切换：四步走

当你点击切换按钮，并且没有开启代理接管模式时，走的是"普通模式"。它的流程分四步：

**第一步：回填（Backfill）**

在写入新配置之前，CC Switch 会先把当前 live 文件的内容读回来，更新到数据库中对应的旧供应商记录里。为什么要这样做？因为用户可能在 CLI 里直接改了配置（比如 `claude code` 运行时 accept 了某个 permission），这些变更如果不回填，下次切回来就丢了。

这一步还会剥离"通用配置片段"（后面会讲），确保回填的只是该供应商自身的配置。

**第二步：更新数据库状态**

在数据库里把 `is_current` 标记指向新的供应商 ID，同时写入本地设备级别的 settings 文件。这里有两层状态——数据库里的 current 是"默认值"（给新设备用的），本地 settings 里的是"本设备的选择"（优先级更高）。这个设计是为了支持多设备通过云同步数据库时，各设备可以有独立的当前供应商。

**第三步：拼接通用配置片段**

这是解决"切供应商后 MCP 配置丢了"的关键机制。CC Switch 允许用户提取一份"通用配置片段"（common config snippet），里面放的是跨供应商共享的东西——比如 MCP 服务器配置、permissions 列表、插件设置等。

切换时，CC Switch 把目标供应商的 `settingsConfig`（只包含 key 和 endpoint）跟通用配置片段做一次 deep merge，生成完整的配置后再写入 live 文件。这样不管你切到哪个供应商，MCP 和权限配置都不会丢。

对于 Claude Code，合并的是 JSON 对象；对于 Codex，合并的是 TOML document；对于 Gemini，合并的是 env 环境变量。同一套逻辑，针对不同格式各有一个合并实现。

**第四步：原子写入 live 文件**

最后一步是把合并好的配置写到 CLI 工具实际读取的位置。这里用了经典的"写临时文件 → rename 替换"的原子写入模式，避免写到一半断电或崩溃导致配置文件损坏。在 Unix 上 rename 是原子操作；在 Windows 上稍有不同（先删再 rename），但也做了尽量安全的处理。

写完 live 文件后，还会触发一次 MCP 同步，确保所有应用的 MCP 服务器配置是一致的。

## 代理热切换：秒切无感

上面说的是"普通模式"——每次切换都要写磁盘。但 CC Switch 还有一种"代理接管模式"，这时候应用的 live 配置文件被改写成指向本地代理（`http://127.0.0.1:15721`），所有请求都经过 CC Switch 内置的本地代理服务器转发。

在这种模式下，切换供应商根本不需要碰磁盘。CC Switch 只是在内存里更新代理的路由目标——把请求从供应商 A 的 endpoint 转到供应商 B 的 endpoint。代码里叫 `hot_switch_provider`，整个过程几乎是瞬时的，CLI 工具完全无感知。

而且这种模式还有个附带好处：因为请求都经过本地代理，CC Switch 可以做故障转移（一个挂了自动切到下一个）、熔断、延迟监控这些事情。

不过代理模式也有一个硬限制：**不允许切换到官方供应商**。代码里有明确的检查，如果你在代理模式下试图切到 `category == "official"` 的供应商，会直接报错。原因很实际——用代理转发官方 API 可能导致你的账号被封。

## 代理的隐藏技能：协议翻译

读到这里你可能会想：代理模式不就是个转发器吗？把请求从本地转发到上游，有什么特别的？

特别的地方在于：上游不一定说同一种"语言"。

Claude Code 只会说 Anthropic Messages API 的协议——它发出的请求是 `{"model": "...", "messages": [...], "system": "...", "max_tokens": 1024}` 这种 Anthropic 格式，它期望收到的响应也是 `{"type": "message", "content": [...], "stop_reason": "end_turn"}` 这种结构。它不认识 OpenAI 的 `{"choices": [{"message": {...}}]}` 格式。

但你想用的上游供应商可能只提供 OpenAI Chat Completions 兼容接口。市面上大量的模型供应商——通义千问、DeepSeek、Moonshot/Kimi、Azure OpenAI、各种中转站——它们对外暴露的是 OpenAI 格式的 API，而不是 Anthropic 格式的。

这就出现了一个断层：客户端只说 Anthropic 语言，上游只听 OpenAI 语言。

CC Switch 的代理在中间做了一件精巧的事：**协议翻译**。它拦截 Claude Code 发出的 Anthropic 格式请求，实时转换成上游能理解的格式，再把上游返回的响应翻译回 Anthropic 格式，让 Claude Code 完全不知道背后对接的其实不是 Anthropic 的服务器。

具体来说，代理支持三种目标格式的转换：

一种是 **OpenAI Chat Completions** 格式（`/v1/chat/completions`），这是最通用的。Anthropic 的 `system` 字段变成 messages 数组里 `role: "system"` 的消息；`messages` 里每条消息的 content 从 Anthropic 的 block 数组（`[{"type": "text", "text": "..."}]`）展平成 OpenAI 的纯字符串或 parts 格式；工具定义从 Anthropic 的 `{"name": ..., "input_schema": {...}}` 转成 OpenAI 的 `{"type": "function", "function": {"name": ..., "parameters": {...}}}` 格式。

另一种是 **OpenAI Responses API** 格式（`/v1/responses`），这是 OpenAI 较新的 API，结构跟 Chat Completions 不太一样。Codex CLI 和 ChatGPT Plus/Pro 的代理走的是这条路。

第三种是 **Gemini Native** 格式（`generateContent`），给 Google 的原生 API 用的。

整个转换不只是简单的字段重命名。它处理了大量边界情况：Anthropic 的 `thinking` block 需要映射到 OpenAI 的 `reasoning_effort` 参数（并且根据 budget_tokens 的大小映射到 low/medium/high）；`tool_use` block 里的结构化 `input` 要序列化成 OpenAI 的字符串格式 `arguments`；Anthropic 的 `stop_sequences` 要变成 OpenAI 的 `stop`；响应里 OpenAI 的 `finish_reason: "stop"` 要翻译回 Anthropic 的 `stop_reason: "end_turn"`。

更复杂的是流式响应。Claude Code 大量使用 streaming，代理需要把 OpenAI 的 SSE chunk 格式（每个 chunk 是一个 delta 对象）实时翻译成 Anthropic 的 SSE 事件格式（`message_start`、`content_block_start`、`content_block_delta`、`message_delta` 这套事件序列）。这不是攒完再转——而是每收到一个 chunk 就立即翻译并推送，保持 streaming 的实时性。

这个设计是怎么启用的呢？每个供应商的配置里有个 `meta.apiFormat` 字段，值可以是 `"anthropic"`（默认，直接透传）、`"openai_chat"`、`"openai_responses"` 或 `"gemini_native"`。代理在处理请求时先检查这个字段，如果不是 `"anthropic"`，就走对应的转换路径。对用户来说，这是透明的——你在 CC Switch 界面上添加一个 OpenAI 格式的供应商时，它会自动帮你设好这个字段。

这意味着什么？意味着你可以把 **任何** OpenAI 兼容的供应商接入 Claude Code。DeepSeek、Moonshot、通义千问、Azure OpenAI、甚至本地跑的 Ollama——只要它对外提供 OpenAI Chat Completions 接口，CC Switch 就能让 Claude Code 无缝使用它。Claude Code 以为自己在跟 Anthropic 的服务器对话，实际上请求早就被翻译成 OpenAI 格式发给了别的模型。

某种意义上，代理模式把 CC Switch 从一个"配置切换器"升级成了一个"协议网关"。它不只是帮你管配置文件——它还帮你打破了 AI CLI 工具只能对接特定格式 API 的限制。

## 设计哲学：最小侵入性

读完切换逻辑，能感受到整个项目有一条贯穿始终的设计原则：**最小侵入性**。

首先，数据库里永远保留至少一个活跃供应商，确保你卸载 CC Switch 后 CLI 工具还能正常工作——因为 live 配置文件里的内容是完整的、可独立使用的。

其次，backfill 机制意味着 CC Switch 不会"霸占"你的配置。你在 CLI 里手动改了什么，下次切换时它会先帮你存好，不会丢。

第三，通用配置片段是"可选的"。如果你不提取，切换就是纯粹的配置替换；提取了，切换时才会做合并。控制权在用户手上。

最后，原子写入保护了配置文件不会因为写入失败而变成一个半成品——这在日常使用中可能不明显，但如果你经历过一次"settings.json 被写坏导致 CLI 启动失败"的事故，就知道这个细节多有价值。

## 适合谁

如果你只用一个 CLI 工具、一个供应商，CC Switch 对你来说是多余的。但如果你符合以下任一情况，它能省不少事：

你在多个 API 供应商之间频繁切换（比如额度轮换）；你同时使用 2-3 个 AI CLI 工具且希望它们共享同一套 MCP 和 Skills 配置；你受够了每次切供应商后要重新配一遍插件。

作为一个开源项目，它的代码质量和工程化程度（完善的测试、原子写入、多平台兼容处理）都值得学习。如果你对 Tauri 桌面应用开发或者 Rust + React 的实践感兴趣，源码本身也是不错的参考。
