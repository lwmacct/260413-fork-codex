# API Key 模式下 `/status` 仍显示 `Limits` 的原因分析

## 结论

在这套代码里，`Limits` 被渲染出来有两条数据来源：

1. 主动查询：`/status` 触发 `account/rateLimits/read`，然后 app-server 再去请求 usage 接口
2. 被动提取：模型主请求的响应头或流式事件里直接携带 rate limits，客户端收到后直接缓存并渲染

你的场景里，如果客户端“当前实际是 API key 模式”，那**更可能是第 2 条**，不是又额外主动请求了一次 `https://chatgpt.com/backend-api/wham/usage`。

原因是：

- 主动查询路径明确要求当前 auth 是 ChatGPT auth，否则直接拒绝
- 但被动提取路径对 auth mode 没有任何限制
- 只要中转代理在主响应里返回了 `x-codex-*` 限额头，或者返回了 `type = "codex.rate_limits"` 的事件，Codex 就会照单全收并渲染

换句话说，**API key 模式并不阻止 `Limits` 被渲染**；它只阻止“主动去查 ChatGPT usage 接口”。

## 一、两条来源要严格区分

### 路径 A：`/status` 主动查询

入口在：

- `codex-rs/tui/src/chatwidget/slash_dispatch.rs:322-330`
- `codex-rs/tui/src/chatwidget.rs:7825-7827`

`/status` 只有在下面条件成立时才会主动刷新 limits：

```rust
self.config.model_provider.requires_openai_auth && self.has_chatgpt_account
```

而 app-server 真正拉 limits 时还会再做一次 auth 校验：

- `codex-rs/app-server/src/codex_message_processor.rs:2005-2028`

关键判断：

```rust
if !auth.is_chatgpt_auth() {
    return Err(JSONRPCErrorError {
        message: "chatgpt authentication required to read rate limits".to_string(),
        ...
    });
}
```

所以：

- 如果当前 session 在 app-server 看来是纯 `ApiKey`
- 那么这条主动查询链路按代码是不该成功的

### 路径 B：主响应流被动携带 rate limits

这条链路完全独立于 `/status`。

只要主模型请求的响应里出现 rate limit 信息，Codex 就会：

1. 解析
2. 缓存
3. 转成 app-server 通知
4. 最终渲染到 `/status`

这条链路**不依赖 `has_chatgpt_account`**，也**不校验当前是否是 ChatGPT auth**。

## 二、被动链路的完整调用路径

## 2.1 `codex-api` 从主响应里提取 limits

### HTTP/SSE 路径：从响应头解析

文件：

- `codex-rs/codex-api/src/sse/responses.rs:57-105`

关键逻辑：

```rust
let rate_limit_snapshots = parse_all_rate_limits(&stream_response.headers);
...
for snapshot in rate_limit_snapshots {
    let _ = tx_event.send(Ok(ResponseEvent::RateLimits(snapshot))).await;
}
```

也就是说：

- 对模型主请求的 SSE 响应
- 只要响应头里有符合规则的 `x-codex-*` 头
- 就会立刻生成 `ResponseEvent::RateLimits`

### WebSocket 路径：从流式事件解析

文件：

- `codex-rs/codex-api/src/endpoint/responses_websocket.rs:600-612`

关键逻辑：

```rust
if event.kind() == "codex.rate_limits" {
    if let Some(snapshot) = parse_rate_limit_event(&text) {
        let _ = tx_event.send(Ok(ResponseEvent::RateLimits(snapshot))).await;
    }
    continue;
}
```

也就是说：

- 如果 websocket 主流里发来一条 `type = "codex.rate_limits"` 事件
- 也会直接变成 `ResponseEvent::RateLimits`

## 2.2 core 把它写入 session 状态

文件：

- `codex-rs/core/src/session/turn.rs:2088-2092`
- `codex-rs/core/src/session/mod.rs:2554-2600`
- `codex-rs/core/src/state/session.rs:130-140`

调用顺序：

1. `ResponseEvent::RateLimits(snapshot)`
2. `sess.update_rate_limits(...)`
3. `state.set_rate_limits(snapshot)`
4. `send_token_count_event(...)`

核心代码：

```rust
ResponseEvent::RateLimits(snapshot) => {
    sess.update_rate_limits(&turn_context, snapshot).await;
}
```

和：

```rust
state.set_rate_limits(new_rate_limits);
self.send_token_count_event(turn_context).await;
```

以及：

```rust
(self.token_info(), self.latest_rate_limits.clone())
```

这说明：

- rate limits 被作为 session 状态的一部分保留下来
- 然后和 token 使用量一起往上游发送

## 2.3 app-server 把它转成 `AccountRateLimitsUpdated`

文件：

- `codex-rs/app-server/src/bespoke_event_handling.rs:2306-2328`

关键逻辑：

```rust
let TokenCountEvent { info, rate_limits } = token_count_event;
...
if let Some(rate_limits) = rate_limits {
    outgoing
        .send_server_notification(ServerNotification::AccountRateLimitsUpdated(
            AccountRateLimitsUpdatedNotification {
                rate_limits: rate_limits.into(),
            },
        ))
        .await;
}
```

注意这里没有任何“只有 ChatGPT auth 才允许发通知”的判断。

也就是说：

- 只要底层主响应里带了 rate limits
- app-server 就会把它广播成 `AccountRateLimitsUpdated`

## 2.4 TUI 收到通知后直接更新缓存

文件：

- `codex-rs/tui/src/app/app_server_adapter.rs:172-176`

关键逻辑：

```rust
ServerNotification::AccountRateLimitsUpdated(notification) => {
    self.chat_widget.on_rate_limit_snapshot(Some(
        app_server_rate_limit_snapshot_to_core(notification.rate_limits.clone()),
    ));
    return;
}
```

最终，chatwidget 在处理 `TokenCount` 时也会接受 `rate_limits`：

- `codex-rs/tui/src/chatwidget.rs:7026-7029`

```rust
EventMsg::TokenCount(ev) => {
    self.set_token_info(ev.info);
    self.on_rate_limit_snapshot(ev.rate_limits);
}
```

所以 TUI 最终根本不关心这份 limits 是怎么来的：

- 是 `/status` 主动查来的
- 还是主模型响应里顺手带来的

只要到了 `on_rate_limit_snapshot()`，就会进入同一份缓存并被 `/status` 渲染。

## 三、为什么 API key 模式下依然可能显示 `Limits`

这是这次分析里最关键的结论。

### 3.1 `has_chatgpt_account` 只影响主动刷新，不影响被动渲染

`has_chatgpt_account` 由账户状态更新：

- `codex-rs/tui/src/chatwidget.rs:9829-9837`
- `codex-rs/tui/src/app/app_server_adapter.rs:178-188`
- `codex-rs/tui/src/app_server_session.rs:239-267`

当 `account/read` 返回 `Account::ApiKey` 时，TUI 启动阶段会把：

- `status_account_display = ApiKey`
- `has_chatgpt_account = false`

但这只会影响：

- `/status` 是否主动触发 `RefreshRateLimits`
- fast mode 等一些依赖 ChatGPT account 的 UI 能力

它**不会禁止** `on_rate_limit_snapshot()` 接受从主响应链路来的 rate limits。

### 3.2 TUI 渲染 `Limits` 时不会检查 auth mode

`on_rate_limit_snapshot()` 本身只管更新缓存：

- `codex-rs/tui/src/chatwidget.rs:2888-2983`

然后 `/status` 在渲染时只读取缓存：

- `codex-rs/tui/src/chatwidget.rs:7504-7558`
- `codex-rs/tui/src/status/card.rs:394-442`

这里没有“如果当前是 ApiKey 就丢弃 limits”的逻辑。

所以只要缓存里有 snapshot，`Limits` 就可能显示。

## 四、中转代理要注入什么内容，客户端才会显示 `Limits`

这部分非常重要，因为它直接回答“是不是代理在主响应里带回来的”。

## 4.1 代理在 HTTP/SSE 主响应头里注入这些头

解析逻辑在：

- `codex-rs/codex-api/src/rate_limits.rs:21-98`
- `codex-rs/codex-api/src/rate_limits.rs:181-216`

默认 `codex` bucket 会查这些头：

- `x-codex-primary-used-percent`
- `x-codex-primary-window-minutes`
- `x-codex-primary-reset-at`
- `x-codex-secondary-used-percent`
- `x-codex-secondary-window-minutes`
- `x-codex-secondary-reset-at`
- `x-codex-credits-has-credits`
- `x-codex-credits-unlimited`
- `x-codex-credits-balance`
- `x-codex-limit-name`

额外 bucket 则是：

- `x-<limit>-primary-used-percent`
- `x-<limit>-primary-window-minutes`
- `x-<limit>-primary-reset-at`
- `x-<limit>-secondary-used-percent`
- `x-<limit>-secondary-window-minutes`
- `x-<limit>-secondary-reset-at`
- `x-<limit>-limit-name`

只要这些头出现在**主模型请求的响应头**里，Codex 就会渲染 limits。

### 4.1.1 `x-codex-*` 头的精确解析规则

这些规则都来自：

- `codex-rs/codex-api/src/rate_limits.rs:21-258`

客户端的解析规则非常“宽松”，本质上是“只要格式能 parse，就信任并展示”。

#### 默认 bucket

默认 bucket 固定看 `x-codex-*`：

- `x-codex-primary-used-percent`
- `x-codex-primary-window-minutes`
- `x-codex-primary-reset-at`
- `x-codex-secondary-used-percent`
- `x-codex-secondary-window-minutes`
- `x-codex-secondary-reset-at`
- `x-codex-limit-name`
- `x-codex-credits-has-credits`
- `x-codex-credits-unlimited`
- `x-codex-credits-balance`

如果调用的是 `parse_all_rate_limits()`，默认 bucket 总会先尝试按 `codex` 解析一次。

#### 额外 bucket

额外 bucket 的发现规则不是“扫描所有可能的前缀”，而是：

- 遍历响应头名
- 找出所有以 `-primary-used-percent` 结尾的头
- 再从头名中反推出 bucket id

对应代码：

```rust
fn header_name_to_limit_id(header_name: &str) -> Option<String> {
    let suffix = "-primary-used-percent";
    let prefix = header_name.strip_suffix(suffix)?;
    let limit = prefix.strip_prefix("x-")?;
    Some(normalize_limit_id(limit.to_string()))
}
```

这意味着：

- 如果一个自定义 bucket 根本没有 `x-<bucket>-primary-used-percent`
- 即使你给了 `secondary-*` 或 `limit-name`
- `parse_all_rate_limits()` 也**不会自动发现这个 bucket**

这是被动解析里一个很重要的限制。

#### `limit_id` 归一化规则

bucket id 会做下面处理：

- `trim()`
- 转小写
- `-` 替换为 `_`

也就是：

```rust
name.into().trim().to_ascii_lowercase().replace('-', "_")
```

所以：

- `x-Codex-Secondary-primary-used-percent`
- `x-codex-secondary-primary-used-percent`

最后都会被归一成 `codex_secondary`。

#### `limit_name` 规则

`x-<bucket>-limit-name` 是可选的，只要是非空字符串就会被接受。

客户端不会校验这个名字是不是“官方套餐名”或“官方模型名”。

因此：

- 服务端/代理完全可以自定义 `limit_name`
- TUI 会直接把它拿来显示

这也是为什么你可能看到不同套餐下显示的 label 完全不同。

#### window 值规则

window 解析要求：

- `used_percent` 必须能 parse 成有限 `f64`
- `window_minutes` 如果有，必须能 parse 成 `i64`
- `reset_at` 如果有，必须能 parse 成 `i64`

但一个 window 是否“存在”，还要满足：

```rust
used_percent != 0.0
    || window_minutes.is_some_and(|minutes| minutes != 0)
    || resets_at.is_some()
```

注意两个边界：

1. **没有 `used_percent` 就不会生成 window**
   即使你给了 reset 时间和窗口时长，也不行。
2. `used_percent = 0` 并不一定消失
   只要你还给了非 0 的 `window_minutes` 或 `reset_at`，这个 window 仍然会被保留。

#### credits 规则

credits 不是按 bucket 前缀解析的，而是固定读：

- `x-codex-credits-has-credits`
- `x-codex-credits-unlimited`
- `x-codex-credits-balance`

对应代码：

```rust
let has_credits = parse_header_bool(headers, "x-codex-credits-has-credits")?;
let unlimited = parse_header_bool(headers, "x-codex-credits-unlimited")?;
```

这说明：

- credits 解析是“全局”的，不是每个 bucket 单独一套
- 只要这两个 bool 头缺一个或格式不合法，credits 整体就不会生成
- `balance` 只是可选字符串

也就是说，**被动 header 路径下没有 per-bucket credits 的概念**。

#### bool 规则

bool 只接受：

- `true`
- `false`
- `1`
- `0`

大小写 `true/false` 也接受，因为用了 `eq_ignore_ascii_case`。

除了这些值，都会被判定为无效，从而导致 credits 不生成。

#### 客户端不会做的校验

对 header 路径来说，客户端**不会**校验：

- header 是否来自官方服务端
- `limit_name` 是否真实存在
- `used_percent` 是否符合套餐规则
- 这个 bucket 是否真的属于当前账户
- 当前 auth mode 是否应该拥有这些 limits

因此，从实现上说：

- 服务端可以自定义内容
- 中转代理也可以自定义内容
- 客户端只做“语法级 parse”，不做“语义级验证”

### 4.1.2 哪些字段不能靠 `x-codex-*` 头传下来

有两个字段在 header 路径里不会被设置：

- `plan_type`
- `rate_limit_reached_type`

因为 `parse_rate_limit_for_limit()` 里固定写的是：

```rust
plan_type: None,
rate_limit_reached_type: None,
```

所以如果你看到：

- 不同套餐对应不同 plan 文案
- 或不同“额度耗尽原因”

这些信息更可能来自：

- `codex.rate_limits` 事件
- `account/rateLimits/read` 返回的 payload
- `account/updated` / `account/read` 返回的账户状态

而不是单纯的 `x-codex-*` 响应头。

## 4.2 代理在 WebSocket 主流里注入 `codex.rate_limits`

解析逻辑在：

- `codex-rs/codex-api/src/rate_limits.rs:120-170`

事件最核心的字段是：

```json
{
  "type": "codex.rate_limits",
  "plan_type": "...",
  "metered_limit_name": "codex",
  "rate_limits": {
    "primary": {
      "used_percent": 42,
      "window_minutes": 60,
      "reset_at": 1730000000
    },
    "secondary": {
      "used_percent": 5,
      "window_minutes": 1440,
      "reset_at": 1730003600
    }
  },
  "credits": {
    "has_credits": true,
    "unlimited": false,
    "balance": "123"
  }
}
```

关键点是 `type == "codex.rate_limits"`。

只要代理在主 websocket 流里发出这样的事件，Codex 也会显示 limits。

## 五、所以这次更可能是谁发起了请求

这里要区分两种“请求”：

### 情况 A：客户端主动新发了 usage 请求

这条请求是：

- `GET <chatgpt_base_url>/wham/usage`
- 或 `GET <chatgpt_base_url>/api/codex/usage`

代码：

- `codex-rs/backend-client/src/client.rs:291-299`

但按代码，它需要：

- `/status` 触发主动刷新
- 且当前 auth 被识别为 ChatGPT auth

所以如果你已经确认当前客户端在 app-server 里是 `ApiKey`，那么这条路径**不应该成功**。

### 情况 B：主模型请求的响应里就自带了 limits

这是更符合你描述的情况：

- 你说中转代理“代替你使用了 chatgpt 格式的请求/响应”
- 你看到的是“主请求内容/响应内容像 ChatGPT”
- 而不是看到一条单独的 `GET /wham/usage`

从代码推断，这更像是：

- 你的代理在 `/responses` 主响应上附带了 `x-codex-*` 头
- 或在 websocket 流里附带了 `codex.rate_limits` 事件
- 客户端于是把这些内容当成 limits 正常渲染

这是**基于源码行为的推断**，不是运行时抓包的直接证明。

## 六、还有一个次要可能：显示的是旧缓存

即使当前是 API key，`Limits` 也可能来自旧缓存。

因为：

- `update_account_state()` 只更新账户状态，不清空 rate limit 缓存
  - `codex-rs/tui/src/chatwidget.rs:9829-9839`
- 真正清空缓存只有 `on_rate_limit_snapshot(None)`
  - `codex-rs/tui/src/chatwidget.rs:2979-2983`

所以如果你：

1. 之前有过 ChatGPT auth
2. 当时缓存过 limits
3. 后来切到 API key

那 `/status` 仍然可能显示旧的 limits。

但如果你观察到用量数值随着当前请求继续变化，那就更像是“主响应流正在持续提供 limits”，而不是旧缓存。

## 七、如何判断到底是哪一种

最有效的判断方式不是看 `/status`，而是看网络和流式事件。

### 7.1 如果出现单独的 usage 请求

看有没有这类请求：

- `GET .../wham/usage`
- `GET .../api/codex/usage`

如果有，说明走了主动查询路径。

### 7.2 如果没有 usage 请求，但主响应里有这些内容

看模型主请求的响应：

- HTTP/SSE 首响应头里是否存在 `x-codex-primary-used-percent` 等头
- WebSocket 文本流里是否存在 `type = "codex.rate_limits"` 事件

如果存在，说明 limits 来自主响应，被动提取，不是另发 usage 请求。

### 7.3 如果都没有，但 `/status` 还显示

那就要怀疑是旧缓存未清空。

## 八、最终回答你的问题

针对你这句话：

> 我要分析的是为什么用量依然被渲染出来了，是对 `https://chatgpt.com/backend-api/wham/usage` 发起了新的请求还是，中转代理提取出来了默认，或者说由中转代理进行了请求

基于这份代码，更合理的回答是：

- **如果当前客户端真的是 API key 模式，那么不太像是客户端又主动新发了一次 `https://chatgpt.com/backend-api/wham/usage`**
- **更像是中转代理在主模型响应里返回了 ChatGPT/Codex 风格的 limits 信息**
- **客户端并不会校验这些 limits 是否只该出现在 ChatGPT auth 模式下，它会直接解析并渲染**

更具体地说：

1. 代理可能在主 SSE 响应头里注入了 `x-codex-*` 限额头
2. 代理可能在主 websocket 流里注入了 `codex.rate_limits` 事件
3. 代理也可能自己向上游查过 usage，再把结果并入主响应

第 3 点客户端代码本身看不出来；对客户端来说，结果都一样：

- 收到 rate limits
- 生成 `ResponseEvent::RateLimits`
- 缓存
- 渲染到 `/status`

## 九、关键源码位置

- `/status` 主动刷新入口：
  `codex-rs/tui/src/chatwidget/slash_dispatch.rs:322-330`
- 主动刷新条件：
  `codex-rs/tui/src/chatwidget.rs:7825-7827`
- app-server 主动查 limits 时要求 ChatGPT auth：
  `codex-rs/app-server/src/codex_message_processor.rs:2005-2028`
- usage 请求地址拼接：
  `codex-rs/backend-client/src/client.rs:291-299`
- SSE 主响应头解析 limits：
  `codex-rs/codex-api/src/sse/responses.rs:57-105`
- WebSocket `codex.rate_limits` 事件解析：
  `codex-rs/codex-api/src/endpoint/responses_websocket.rs:600-612`
- rate limit 头和事件的解析规则：
  `codex-rs/codex-api/src/rate_limits.rs:21-246`
- core 收到 `ResponseEvent::RateLimits`：
  `codex-rs/core/src/session/turn.rs:2088-2092`
- core 写入 session rate limits：
  `codex-rs/core/src/session/mod.rs:2554-2600`
- state 保存 latest_rate_limits：
  `codex-rs/core/src/state/session.rs:130-140`
- app-server 广播 `AccountRateLimitsUpdated`：
  `codex-rs/app-server/src/bespoke_event_handling.rs:2306-2328`
- TUI 接收通知后更新缓存：
  `codex-rs/tui/src/app/app_server_adapter.rs:172-176`
- TUI 处理 token count 时接受 rate limits：
  `codex-rs/tui/src/chatwidget.rs:7026-7029`
- 切换账户状态不清空缓存：
  `codex-rs/tui/src/chatwidget.rs:9829-9839`
- 真正清空缓存的地方：
  `codex-rs/tui/src/chatwidget.rs:2979-2983`
