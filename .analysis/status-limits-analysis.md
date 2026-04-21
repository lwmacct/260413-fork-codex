# `/status` 命令中 `Limits` 信息来源分析

## 结论摘要

`/status` 里的 `Limits` 不是在命令执行时即时计算出来的单一值，而是由一组缓存的 `RateLimitSnapshotDisplay` 渲染出来的展示结果。

整体链路是：

1. 用户执行 `/status`
2. TUI 立即用当前缓存渲染一张 status 卡片
3. 如果当前会话满足 ChatGPT 鉴权条件，TUI 同时异步发起一次 `account/rateLimits/read`
4. app-server 通过 `BackendClient::get_rate_limits_many()` 请求后端 usage 接口
5. 后端返回的 `RateLimitStatusPayload` 被转换成一个或多个 `RateLimitSnapshot`
6. TUI 将这些 snapshot 更新到 `rate_limit_snapshots_by_limit_id`
7. `/status` 卡片中的 `Limits` 区域再根据这些 snapshot 生成最终文案

因此，`Limits` 显示的是：

- 本地缓存中的 rate limit 快照
- 或本次 `/status` 触发的异步刷新结果
- 再经过 TUI 的展示层格式化与状态判定后得到的文案

## 1. `/status` 的入口与行为

`/status` 的入口在 `codex-rs/tui/src/chatwidget/slash_dispatch.rs`。

- `SlashCommand::Status` 会先判断 `should_prefetch_rate_limits()` 是否为真。
- 条件成立时，先 `add_status_output(true, Some(request_id))`，再发送 `AppEvent::RefreshRateLimits`。
- 条件不成立时，只渲染当前状态，不发起 limits 刷新。

关键代码：

- `codex-rs/tui/src/chatwidget/slash_dispatch.rs:322-335`
- `codex-rs/tui/src/chatwidget.rs:7825-7827`

其中 `should_prefetch_rate_limits()` 的条件很简单：

```rust
self.config.model_provider.requires_openai_auth && self.has_chatgpt_account
```

也就是说，只有需要 OpenAI 鉴权并且当前确实有 ChatGPT 账户时，`/status` 才会主动刷新 limits。

## 2. `/status` 卡片最初拿到的 limits 数据

`add_status_output()` 会从 `chatwidget.rate_limit_snapshots_by_limit_id` 里取出所有当前缓存的 limits 快照，然后交给 status 渲染模块。

关键代码：

- `codex-rs/tui/src/chatwidget.rs:7504-7558`

这里最关键的是：

```rust
let rate_limit_snapshots: Vec<RateLimitSnapshotDisplay> = self
    .rate_limit_snapshots_by_limit_id
    .values()
    .cloned()
    .collect();
```

也就是说，`/status` 初次渲染使用的是 TUI 内存中的缓存，而不是现场直接请求后端后再渲染。

这也是为什么：

- `/status` 可以“立即显示”
- 刷新是异步的
- 刷新回来后只更新对应的 status 卡片内容

## 3. `/status` 刷新请求如何发出

当 `SlashCommand::Status` 触发刷新时，会发送 `AppEvent::RefreshRateLimits`。

这个事件在 `App::refresh_rate_limits()` 中被处理，TUI 会起一个后台任务调用 `fetch_account_rate_limits(request_handle)`，然后把结果包装成 `AppEvent::RateLimitsLoaded` 发回主循环。

关键代码：

- `codex-rs/tui/src/app/background_requests.rs:25-45`
- `codex-rs/tui/src/app/event_dispatch.rs:487-505`

成功时：

1. 遍历返回的所有 snapshot
2. 逐个调用 `chat_widget.on_rate_limit_snapshot(Some(snapshot))`
3. 如果这是 `/status` 触发的刷新，则调用 `finish_status_rate_limit_refresh(request_id)`，把对应 status 卡片里的 limits 区域更新掉

失败时：

- 不会清空已有缓存
- 但会结束这次 `/status` 卡片的刷新流程

## 4. TUI 如何缓存 rate limit 快照

缓存更新逻辑在 `chatwidget.on_rate_limit_snapshot()`。

关键代码：

- `codex-rs/tui/src/chatwidget.rs:2888-2983`

这里有几个关键点：

### 4.1 bucket key 是 `limit_id`

如果后端没给 `limit_id`，TUI 默认按 `"codex"` 处理：

```rust
let limit_id = snapshot
    .limit_id
    .clone()
    .unwrap_or_else(|| "codex".to_string());
```

因此默认主 bucket 就是 `codex`。

### 4.2 展示名称优先用 `limit_name`

如果有 `limit_name` 就用它，没有就退回 `limit_id`：

```rust
let limit_label = snapshot
    .limit_name
    .clone()
    .unwrap_or_else(|| limit_id.clone());
```

这个 label 会进入 `RateLimitSnapshotDisplay.limit_name`，后面决定 `/status` 中展示成 `5h limit`、`Weekly limit`、`Monthly limit`，还是 `codex_other 30m limit` 这类文字。

### 4.3 新快照没带 `credits` 时会沿用旧值

```rust
if snapshot.credits.is_none() {
    snapshot.credits = self
        .rate_limit_snapshots_by_limit_id
        .get(&limit_id)
        .and_then(|display| display.credits.as_ref())
        .map(...)
}
```

这意味着 `/status` 中的 `Credits` 有可能来自更早一次快照，而不是这次返回本身。

### 4.4 最终缓存结构

TUI 最终存的是：

- `BTreeMap<String, RateLimitSnapshotDisplay> rate_limit_snapshots_by_limit_id`

也就是“按 limit_id 分桶后的展示态快照”。

## 5. app-server 返回给 TUI 的 limits 结构

TUI 通过 app-server 的 `account/rateLimits/read` RPC 获取数据。

协议定义在：

- `codex-rs/app-server-protocol/src/protocol/v2.rs:1928-1932`

结构是：

```rust
pub struct GetAccountRateLimitsResponse {
    pub rate_limits: RateLimitSnapshot,
    pub rate_limits_by_limit_id: Option<HashMap<String, RateLimitSnapshot>>,
}
```

注意这里有两层：

- `rate_limits`
  兼容历史逻辑的单 bucket 视图
- `rate_limits_by_limit_id`
  新的多 bucket 视图

TUI 在 `app_server_rate_limit_snapshots_to_core()` 里会把这两部分都转成 core 层的 `RateLimitSnapshot`。

关键代码：

- `codex-rs/tui/src/app_server_session.rs:1261-1287`

这一步之后，TUI 已经不关心 app-server 返回的是“单桶兼容字段”还是“多桶 map”了，统一变成多个 `RateLimitSnapshot` 来处理。

## 6. app-server 的 limits 数据从哪里来

app-server 处理 `account/rateLimits/read` 的逻辑在：

- `codex-rs/app-server/src/codex_message_processor.rs:2005-2087`

流程如下：

1. 检查当前是否已登录
2. 检查是否是 ChatGPT auth
3. 基于当前 auth 构造 `BackendClient`
4. 调用 `client.get_rate_limits_many().await`
5. 将返回的多个 snapshot 整理成：
   - 一个 primary bucket
   - 一个 `rate_limits_by_limit_id`

这里 primary bucket 的选择规则是：

- 优先取 `limit_id == "codex"` 的快照
- 如果没有，就取第一个 snapshot

对应代码：

```rust
let primary = snapshots
    .iter()
    .find(|snapshot| snapshot.limit_id.as_deref() == Some("codex"))
    .cloned()
    .unwrap_or_else(|| snapshots[0].clone());
```

所以 `/status` 最终能看到多 bucket 信息，但 app-server 仍然维护了一个兼容的主 bucket 返回值。

## 7. 后端 usage 接口的真实来源

真正请求后端的地方在：

- `codex-rs/backend-client/src/client.rs:291-299`

`get_rate_limits_many()` 会请求：

- `PathStyle::CodexApi` 时：`/api/codex/usage`
- `PathStyle::ChatGptApi` 时：`/wham/usage`

然后把返回 JSON 反序列化为 `RateLimitStatusPayload`。

## 8. 后端 JSON 是怎样映射成 snapshot 的

后端 payload 结构在 openapi models 中：

- `codex-rs/codex-backend-openapi-models/src/models/rate_limit_status_payload.rs:15-47`
- `codex-rs/codex-backend-openapi-models/src/models/rate_limit_status_details.rs:15-35`
- `codex-rs/codex-backend-openapi-models/src/models/additional_rate_limit_details.rs:15-28`
- `codex-rs/codex-backend-openapi-models/src/models/credit_status_details.rs:13-40`

关键信息：

- `plan_type`
- `rate_limit`
- `credits`
- `additional_rate_limits`
- `rate_limit_reached_type`

在 `backend-client` 中，这个 payload 被转换为多个 `RateLimitSnapshot`：

- `codex-rs/backend-client/src/client.rs:445-560`

映射规则是：

### 8.1 默认主 bucket

主 bucket 总是先构造一个 `limit_id = "codex"` 的 snapshot：

```rust
Self::make_rate_limit_snapshot(
    Some("codex".to_string()),
    None,
    payload.rate_limit.flatten().map(|details| *details),
    payload.credits.flatten().map(|details| *details),
    plan_type,
    rate_limit_reached_type,
)
```

### 8.2 额外 bucket

`additional_rate_limits` 里的每个条目会转换成一个单独 snapshot：

- `metered_feature` -> `limit_id`
- `limit_name` -> `limit_name`
- `rate_limit` -> primary/secondary window
- `credits` 不跟随额外 bucket，一律为 `None`

所以多 bucket 情况下：

- credits 只来自主 `codex` bucket
- 非 `codex` bucket 主要只提供 usage window 信息

### 8.3 window 字段映射

`RateLimitWindowSnapshot` 会被转换成：

- `used_percent`
- `window_minutes`
- `resets_at`

`window_minutes` 是由秒数换算来的。

## 9. `/status` 中 `Limits` 文案是怎么决定的

真正决定 `/status` 文案的是 `codex-rs/tui/src/status/`。

关键文件：

- `codex-rs/tui/src/status/rate_limits.rs`
- `codex-rs/tui/src/status/card.rs`

### 9.1 先把 snapshot 转成展示态

`rate_limit_snapshot_display_for_limit()` 会把协议层 snapshot 转成 `RateLimitSnapshotDisplay`：

- `codex-rs/tui/src/status/rate_limits.rs:126-143`

这里会做两件事：

1. 把 `resets_at` 时间戳转成本地时间显示字符串
2. 把 `credits` 转成专门的展示结构

### 9.2 再把展示态归类成 4 种状态

`compose_rate_limit_data_many()` 会输出：

- `Available`
- `Stale`
- `Unavailable`
- `Missing`

关键代码：

- `codex-rs/tui/src/status/rate_limits.rs:170-279`

状态判定规则：

#### `Missing`

没有任何 snapshot：

```rust
if snapshots.is_empty() {
    return StatusRateLimitData::Missing;
}
```

#### `Unavailable`

有 snapshot，但拼不出任何可展示 row。

row 的来源是：

- primary window
- secondary window
- credits

如果三者都没有，就会变成 `Unavailable`。

#### `Stale`

只要任意 snapshot 的 `captured_at` 距当前时间超过 15 分钟，就标记为 stale：

- `RATE_LIMIT_STALE_THRESHOLD_MINUTES = 15`
- `codex-rs/tui/src/status/rate_limits.rs:59-60`
- `codex-rs/tui/src/status/rate_limits.rs:181-183`

#### `Available`

有可展示 row，且不 stale。

### 9.3 每一行如何组成

row 的标签和内容也不是固定写死的。

关键规则：

- `codex` bucket 通常直接显示 `5h limit`、`Weekly limit`
- 非 `codex` bucket 如果只有一个 window，会显示成 `<limit_name> <duration> limit`
- 非 `codex` bucket 如果有多个 window，会先插一行 `<limit_name> limit`，再跟具体窗口行

对应代码：

- `codex-rs/tui/src/status/rate_limits.rs:185-270`

这也是为什么 snapshot 里如果出现额外 metered feature，`/status` 中可能看到额外 limit 行，而不只是一组传统的 5h/weekly。

## 10. `/status` 中最终显示成什么文字

最后由 `StatusHistoryCell::rate_limit_lines()` 决定文案：

- `codex-rs/tui/src/status/card.rs:394-442`

具体是：

### `Available`

显示具体的 limit 行，例如：

- `5h limit: [████...] 55% left`
- `Weekly limit: [████...] 70% left`
- `Credits: 38 credits`

具体渲染逻辑在：

- `codex-rs/tui/src/status/card.rs:445-520`

其中注意：

- 进度条显示的是 `remaining = 100 - used_percent`
- 文案是 `xx% left`
- `resets_at` 会尝试内联显示，不够宽时换行

### `Stale`

先显示具体 limit 行，再加一行 warning：

- 刷新中时：`limits may be stale - run /status again shortly.`
- 非刷新中时：`limits may be stale - start new turn to refresh.`

### `Unavailable`

显示：

- `Limits: not available for this account`

### `Missing`

显示：

- 刷新中时：`Limits: refresh requested; run /status again shortly.`
- 非刷新中时：`Limits: data not available yet`

## 11. 实时更新路径：不是只有 `/status` 才能更新 limits

limits 还有一条“推送式”更新路径。

当 core 侧收到 token count 事件，且事件中带有 rate limits 时，app-server 会转发一条 `AccountRateLimitsUpdated` 通知：

- `codex-rs/app-server/src/bespoke_event_handling.rs:2321-2328`

TUI 收到该通知后，会立刻调用：

```rust
self.chat_widget.on_rate_limit_snapshot(Some(...))
```

对应代码：

- `codex-rs/tui/src/app/app_server_adapter.rs:172-176`

所以 `/status` 看到的 limits，很多时候并不是只靠 `/status` 那次刷新得到的，而是平时运行过程中已经通过通知更新进缓存了。

## 12. 更上游的另一条来源：响应头和 SSE 事件

在更底层的 API 层，Codex 也会从 HTTP 响应头或 websocket/SSE 事件中提取 rate limits。

代码在：

- `codex-rs/codex-api/src/rate_limits.rs:21-246`

这里有两条解析路径：

### 12.1 从响应头解析

`parse_all_rate_limits()` 会遍历 header，识别不同 limit family，例如：

- `x-codex-primary-used-percent`
- `x-codex-secondary-used-percent`
- `x-<other-limit>-primary-used-percent`
- `x-codex-credits-*`

然后拼成一个或多个 `RateLimitSnapshot`。

### 12.2 从事件 payload 解析

`parse_rate_limit_event()` 会解析 `type == "codex.rate_limits"` 的事件。

这条路径主要用于实时更新。

不过对 `/status` 来说，最直接的读取路径仍然是：

- TUI -> app-server `account/rateLimits/read` -> backend `/wham/usage`

## 13. 回答题目：`/status` 的 `Limits` 信息“如何得到”

可以把答案压缩成一句话：

`/status` 的 `Limits` 信息来自 TUI 内存中的 `rate_limit_snapshots_by_limit_id` 缓存；这个缓存由 app-server 的 `account/rateLimits/read` 结果和 `account/rateLimits/updated` 通知持续更新，而 app-server 的数据最终来自后端 usage 接口 `/wham/usage`（或 `/api/codex/usage`）返回的 `RateLimitStatusPayload`。

再展开一点：

1. `/status` 执行时先读取本地缓存并立即渲染
2. 如有 ChatGPT 鉴权，再异步请求 `account/rateLimits/read`
3. app-server 调 `BackendClient::get_rate_limits_many()`
4. 后端 `RateLimitStatusPayload` 被映射为多个 `RateLimitSnapshot`
5. TUI 按 `limit_id` 更新缓存
6. status 模块把缓存转换成：
   - 进度条
   - `% left`
   - reset 时间
   - credits 行
   - missing/unavailable/stale 文案

## 14. 最关键的源码位置

- `/status` 入口：
  `codex-rs/tui/src/chatwidget/slash_dispatch.rs:322-335`
- `/status` 读取缓存并创建卡片：
  `codex-rs/tui/src/chatwidget.rs:7504-7558`
- `/status` 刷新完成后更新卡片：
  `codex-rs/tui/src/chatwidget.rs:7561-7585`
- TUI 缓存 limits：
  `codex-rs/tui/src/chatwidget.rs:2888-2983`
- TUI 展示层状态判定：
  `codex-rs/tui/src/status/rate_limits.rs:170-279`
- `/status` 最终文案：
  `codex-rs/tui/src/status/card.rs:394-442`
- app-server 协议：
  `codex-rs/app-server-protocol/src/protocol/v2.rs:1928-1932`
- TUI 把 app-server 返回转成 core snapshot：
  `codex-rs/tui/src/app_server_session.rs:1261-1287`
- app-server 拉取 limits：
  `codex-rs/app-server/src/codex_message_processor.rs:2005-2087`
- backend 调后端 usage：
  `codex-rs/backend-client/src/client.rs:291-299`
- backend payload -> snapshot：
  `codex-rs/backend-client/src/client.rs:445-560`
- usage payload 结构：
  `codex-rs/codex-backend-openapi-models/src/models/rate_limit_status_payload.rs:15-47`
- 实时通知转发：
  `codex-rs/app-server/src/bespoke_event_handling.rs:2321-2328`
- TUI 接收实时通知：
  `codex-rs/tui/src/app/app_server_adapter.rs:172-176`

