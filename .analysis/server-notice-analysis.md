# Server Notice 分析

## 范围

这份说明文档讨论两个相关问题：

1. 已观察到的 usage-limit UI 文案里，哪些部分是服务端提供的，哪些部分是客户端生成的？
2. 目前服务端可以通过哪些“非阻塞”提示/通知渠道向客户端展示内容？

重点关注本仓库中的 Codex TUI 和 app-server 行为。

## 简短结论

- usage-limit 错误文本不是原始的服务端字符串。
- 后端返回的是结构化错误数据，加上一段可选的 promo 文本，最终句子由客户端在本地拼装。
- `Heads up, you have less than 25%...` 这条 rate-limit 阈值提示完全是客户端生成的。

对于“服务端到客户端的非阻塞通知”，当前相关的展示面包括：

- `configWarning`：服务端定义的自由文本，在历史记录中以 warning 展示。
- `deprecationNotice`：服务端定义的自由文本，以专门的 deprecation notice cell 展示。
- `mcpServer/startupStatus/updated.error`：服务端定义的失败文本，作为 MCP 启动警告展示。
- `model/list` 中的 `availability_nux.message`：服务端定义的启动 tooltip 文本。
- `model/list` 中的 `upgrade_info.upgrade_copy` / `upgrade_info.migration_markdown`：服务端定义的迁移提示文案。

还有一些结构化通道，当前在 TUI 中并不支持或不展示任意服务端文案：

- `account/rateLimits/updated`：只包含结构化数字；warning 文案由本地生成。
- `model/rerouted`：只包含结构化 reroute 元数据；当前 TUI 会忽略它。
- `experimentalFeature/list.announcement`：虽然 API 会返回，但当前 TUI 的 tooltip 流程并不会从 app-server 读取它。

此外，还有一些正常聊天 notice 之外的专用服务端文本通道：

- `thread/realtime/error.message`：自由文本，但只用于 realtime session。
- `account/login/completed.error`：自由文本，但只被登录/引导流程消费。

## 第 1 部分：Usage-limit 文案

### 观察到的字符串 1

示例：

```text
■ You've hit your usage limit. To continue using Codex, start a free trial of Plus today, or try again at Apr 19th, 2026 4:54 PM
```

### 结论

这是一个“混合产物”：

- 服务端提供结构化错误字段，例如 `plan_type` 和 `resets_at`
- 服务端还可能通过 header `x-codex-promo-message` 提供 promo 文案
- 客户端负责拼装最终句子和本地重试时间戳
- TUI 再补上前缀 `■` 的错误表现形式

### 它会终止整个会话吗？

它会终止当前 turn，但不会终止整个 session/thread。

证据：

- TUI 会通过通用错误路径处理 usage-limit error，该路径会调用 `finalize_turn()`：
  - [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2805)
- rate-limit 错误会被显式路由到 `on_error(...)`，而不是 `on_warning(...)`：
  - [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2817)
- core 测试把这种行为描述为：
  - “submission should succeed while emitting usage limit error events”
  - [client.rs](/app/data/workspace/codex-rs/core/tests/suite/client.rs#L2516)

更广义地说，一个 error 结束 turn 之后，客户端会释放该 turn，并允许下一次 prompt 继续：

- [stream_error_allows_next_turn.rs](/app/data/workspace/codex-rs/core/tests/suite/stream_error_allows_next_turn.rs#L107)

所以，如果目标是承载广告或 promo，`usage limit` 并不是一个好载体，因为它本质上仍然是一个会结束当前 turn 的错误面。

### 证据链

HTTP `429` 路径会解析 `usage_limit_reached` 错误，并读取可选的 promo header：

- [api_bridge.rs](/app/data/workspace/codex-rs/codex-api/src/api_bridge.rs#L70)
- [rate_limits.rs](/app/data/workspace/codex-rs/codex-api/src/rate_limits.rs#L172)

最终的人类可读消息是在 `Display for UsageLimitReachedError` 里组装的：

- [error.rs](/app/data/workspace/codex-rs/protocol/src/error.rs#L449)

关键细节：

- 如果存在 `promo_message`，客户端使用：
  - `You've hit your usage limit. {promo_message},{}`
- 重试后缀文本是客户端生成的：
  - `or try again later.`
  - `or try again at {formatted}.`
- 时间戳格式化也是在本地/客户端完成：
  - [error.rs](/app/data/workspace/codex-rs/protocol/src/error.rs#L524)
  - [error.rs](/app/data/workspace/codex-rs/protocol/src/error.rs#L533)

随后，TUI 再把最终错误消息渲染为带本地 `■` 前缀的形式：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2800)
- [history_cell.rs](/app/data/workspace/codex-rs/tui/src/history_cell.rs#L2180)

### 哪些由服务端控制，哪些由客户端控制

服务端控制的输入：

- 错误类型：`usage_limit_reached`
- `plan_type`
- `resets_at`
- 可选的 `x-codex-promo-message`

客户端控制的格式化：

- `You've hit your usage limit. ...`
- `or try again at ...`
- 本地时间戳格式化
- 最终 TUI 错误气泡/前缀

### 观察到的字符串 2

示例：

```text
Heads up, you have less than 25% of your 5h limit left. Run /status for a breakdown.
```

### 结论

这条字符串完全是客户端生成的。

它被硬编码在 TUI 中，由结构化的 rate-limit snapshot 触发。服务端只提供使用量数字和时间窗口大小。

来源：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L463)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L503)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2675)

服务端的 rate-limit 更新路径只传结构化数据：

- [app_server_adapter.rs](/app/data/workspace/codex-rs/tui/src/app/app_server_adapter.rs#L167)
- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L6356)

## 第 2 部分：非阻塞 notice 展示面

这一部分关注的是：哪些来自服务端的内容可以在不结束 turn、也不让 turn 失败的情况下通知用户。

## 适合“广告/宣传文案”的总结

如果目标是“能承载 promo 文案，同时不打断当前对话”，这些通道大致可按下面排序：

最合适：

1. 启动 tooltip / announcement 类展示面
2. model availability NUX

可以用，但产品语义上比较别扭：

1. `configWarning`
2. `deprecationNotice`

不适合：

1. `usage limit`
2. `mcpServer/startupStatus/updated.error`
3. 当前实现下的 `model/rerouted`

原因：

- startup tooltip 和 NUX 展示面本身就更接近广告/提示语气
- `configWarning` / `deprecationNotice` 技术上是非阻塞文本通道，但它们的语义更偏运维/兼容性，而不是营销
- usage-limit 和 MCP failure 展示面本质上是 error/warning 通道

## 1. `configWarning`

### 服务端能发送什么

`ConfigWarningNotification` 包含：

- `summary: String`
- `details: Option<String>`
- `path: Option<String>`
- `range: Option<TextRange>`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L6441)

### TUI 如何展示

TUI 会把它转成历史记录中的 warning 行，并且可见文本只使用 `summary` / `details`：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L6155)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2839)
- [history_cell.rs](/app/data/workspace/codex-rs/tui/src/history_cell.rs#L1771)

当前 TUI 不会在 warning 展示里单独渲染 `path` / `range`。

### 这些 warning 目前来自哪里

app-server 会在启动/配置场景下发出这类 warning，例如：

- 无效配置回退
- exec policy 解析 warning
- project config 被禁用的 notice
- 启动 warning
- 与 sandbox 相关的 warning

来源：

- [app-server/src/lib.rs](/app/data/workspace/codex-rs/app-server/src/lib.rs#L431)
- [app-server/src/lib.rs](/app/data/workspace/codex-rs/app-server/src/lib.rs#L453)

它们会在 initialize 阶段作为非阻塞通知发出：

- [message_processor.rs](/app/data/workspace/codex-rs/app-server/src/message_processor.rs#L453)

### 结论

`configWarning` 是当前最明确的、由服务端定义文本且不会阻塞的通知通道。

## 2. `deprecationNotice`

### 服务端能发送什么

`DeprecationNoticeNotification` 包含：

- `summary: String`
- `details: Option<String>`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L6410)

### TUI 如何展示

TUI 会渲染一个专门的 deprecation notice cell：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L3997)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L6149)
- [history_cell.rs](/app/data/workspace/codex-rs/tui/src/history_cell.rs#L1776)

### 这些 notice 目前来自哪里

core 会在使用已弃用功能或已弃用配置项时发出 deprecation event，随后 app-server 转发它们：

- [core/src/codex.rs](/app/data/workspace/codex-rs/core/src/codex.rs#L1754)
- [bespoke_event_handling.rs](/app/data/workspace/codex-rs/app-server/src/bespoke_event_handling.rs#L1314)

### 结论

`deprecationNotice` 是另一个已经在线上使用的、服务端到客户端的自由文本通知通道，并且不会打断当前对话。

## 3. `account/rateLimits/updated`

### 服务端发送什么

服务端发送的是结构化 snapshot：

- limit id / name
- 使用百分比
- reset 时间戳
- credits / plan type

### 客户端展示什么

用户可见的 warning 文案由 TUI 阈值逻辑在本地生成：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2675)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2701)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2757)

### 结论

这是一个非阻塞通知路径，但不是一个“服务端可定制文案”的路径。当前服务端无法在这里自定义 warning 句子。

## 4. `mcpServer/startupStatus/updated`

### 服务端发送什么

协议中包含：

- `name`
- `status`
- `error: Option<String>`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L5575)

### TUI 如何使用

当状态是 `Failed` 时，如果存在 `notification.error`，TUI 会直接把它作为 warning 展示；否则会退回到本地文案：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L3069)
- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L2900)
- [history_cell.rs](/app/data/workspace/codex-rs/tui/src/history_cell.rs#L1771)

TUI 还会额外生成一些汇总性的 follow-up warning，例如 `MCP startup incomplete (...)`，但这些 summary 行是客户端生成的。

### 结论

这是一个已经在使用的、由服务端定义文本的非阻塞通道，不过它的作用域限定在 MCP server 启动/更新失败。

## 5. `model/list` -> `availability_nux.message`

### 服务端发送什么

模型元数据里可以带：

- `availability_nux.message: String`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L1778)

app-server 会把 preset 中的 model metadata 向前转发：

- [app-server/src/models.rs](/app/data/workspace/codex-rs/app-server/src/models.rs#L35)

### TUI 如何使用

TUI 可能把它作为 startup tooltip override 展示，并在本地对每个模型维护 shown-count：

- [app.rs](/app/data/workspace/codex-rs/tui/src/app.rs#L784)
- [app.rs](/app/data/workspace/codex-rs/tui/src/app.rs#L801)

### 结论

这是服务端定义的文案，也是非阻塞的，但它更像是一个启动/session 级别的提示，而不是对话线程中的实时 notice。

如果目的是做广告或 promo，这已经是当前最干净的现有挂点之一。

## 6. `model/list` -> `upgrade_info.upgrade_copy` / `migration_markdown`

### 服务端发送什么

模型元数据里可以带 upgrade info，包括：

- `upgrade_copy`
- `migration_markdown`
- `model_link`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L1817)

### TUI 如何使用

TUI 可能用服务端提供的文案展示 model migration prompt：

- [app.rs](/app/data/workspace/codex-rs/tui/src/app.rs#L842)
- [model_migration.rs](/app/data/workspace/codex-rs/tui/src/model_migration.rs#L66)

如果存在 `migration_markdown`，它会优先于本地合成文案。

### 结论

这是服务端定义的文本，但它比轻量级的 inline notice 更偏 modal。

它可以承载 promo 文案，但更接近一个 migration/interstitial，而不是一个被动广告位。

## 7. `model/rerouted`

### 服务端发送什么

`ModelReroutedNotification` 只包含结构化字段：

- `thread_id`
- `turn_id`
- `from_model`
- `to_model`
- `reason`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L6402)

### 当前 TUI 行为

TUI 当前会忽略它：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L6148)

### 结论

这目前不是一个可用于服务端自定义文案的 TUI 通道。

## 8. `experimentalFeature/list.announcement`

### API 暴露了什么

experimental feature list 响应里包含一个可选 `announcement` 字段：

- [codex_message_processor.rs](/app/data/workspace/codex-rs/app-server/src/codex_message_processor.rs#L4912)
- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L1924)

### 当前 TUI 行为

当前 tooltip 逻辑不会从 app-server 获取它。相反，TUI 使用的是：

- 内置在本地 `FEATURES` 里的 feature metadata
- 一个托管在 GitHub 上的 `announcement_tip.toml`

来源：

- [tooltips.rs](/app/data/workspace/codex-rs/tui/src/tooltips.rs#L42)
- [tooltips.rs](/app/data/workspace/codex-rs/tui/src/tooltips.rs#L54)

### 结论

协议形状里虽然有 `announcement` 字段，但当前 TUI 并没有把 app-server 当成这个 notice 的来源。

另外，当前 TUI 本身已经有一个轻量级 promo 展示面：startup tooltip 和远程 announcement 文件：

- startup tooltip 渲染：
  - [history_cell.rs](/app/data/workspace/codex-rs/tui/src/history_cell.rs#L1207)
- 远程 announcement tip 来源：
  - [tooltips.rs](/app/data/workspace/codex-rs/tui/src/tooltips.rs#L7)
- 如果远程 tip 获取成功，它会优先于常规 tooltip 选择：
  - [tooltips.rs](/app/data/workspace/codex-rs/tui/src/tooltips.rs#L48)

这条路径当前不是 app-server 驱动的，但它是 TUI 中现有最接近广告位的非阻塞展示槽。

## 9. 专用文本通道

这些字段确实是服务端定义的文本，但它们不是通用的、线程内 notice 展示面。

### `thread/realtime/error`

协议中包含：

- `thread_id`
- `message`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L4061)

TUI 只会把它路由到 realtime conversation 处理逻辑：

- [chatwidget.rs](/app/data/workspace/codex-rs/tui/src/chatwidget.rs#L6219)

所以这是一个只服务 realtime 的 error/status 通道，而不是通用的被动提示系统。

### `account/login/completed`

协议中包含：

- `success`
- `error`

定义：

- [v2.rs](/app/data/workspace/codex-rs/app-server-protocol/src/protocol/v2.rs#L6391)

TUI 会在 onboarding/login UI 中消费它，而不是主聊天历史：

- [onboarding/auth.rs](/app/data/workspace/codex-rs/tui/src/onboarding/auth.rs#L865)

所以这是一条登录流程反馈通道，而不是通用的对话 notice 通道。

## 最终结论

如果问题是：

> 现在服务端到底能定制什么内容，并让客户端以“非阻塞文本 notice”的形式展示出来？

那么实际答案是：

1. `configWarning`
2. `deprecationNotice`
3. `mcpServer/startupStatus/updated.error`
4. 启动阶段的 `availability_nux.message`
5. model migration 的 `upgrade_copy` / `migration_markdown`

如果要求更严格：

> 想要一个“实时出现在对话里、非错误、非模态、由服务端定义文本”的 notice 通道

那么当前已经在使用的通道主要是：

1. `configWarning`
2. `deprecationNotice`
3. `mcpServer/startupStatus/updated.error`

相比之下：

- rate-limit warning 文案是客户端写死的
- usage-limit promo 文案出现在 error 路径里，而不是被动提示路径
- `model/rerouted` 没有消息文本，而且 TUI 会忽略它
- `experimentalFeature/list.announcement` 虽然存在于 API 形状里，但当前没有接到 TUI 展示
- realtime/login 流程有各自独立的文本通道，但它们只服务对应的专用 UI
