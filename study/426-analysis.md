# 426 Upgrade Required 分析

## 范围

这份说明文档描述了在本项目中 HTTP `426 Upgrade Required` 的含义、它是在什么位置被处理的，以及它为什么会影响 ChatGPT 驱动的 `Responses` 传输行为。

它主要回答四个问题：

1. `426` 会在请求链路的什么位置出现？
2. 它会如何映射到客户端内部的错误模型？
3. 为什么它会触发 `FallbackToHttp`？
4. 这会对后续 turn 产生什么实际影响？

## 简短结论

- 在本项目中，`426 Upgrade Required` 会在 Responses WebSocket 建连路径上被当作一个特殊信号处理。
- 如果 WebSocket 握手因为 HTTP `426` 失败，客户端不会把它当成普通的传输错误。
- 相反，它会返回 `FallbackToHttp`。
- 随后该 session 会禁用 WebSocket，并通过标准的 HTTP Responses API（`POST /responses` + SSE）重放请求。

因此，这里的 `426` 实际含义是：

- “这个 session 不要继续使用 Responses-over-WebSocket；切换到 HTTP/SSE。”

## 核心处理路径

### 1. WebSocket 连接尝试

对于正常的、由 ChatGPT 驱动的 `Responses` 流式请求，只要 WebSocket 传输可用，turn client 会优先选择 WebSocket。

来源：

- [client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1431)

这条路径会调用 `stream_responses_websocket(...)`，它会先尝试建立 WebSocket 连接。

来源：

- [client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1447)

### 2. 握手 HTTP 错误映射

如果 WebSocket 握手因为一个 HTTP 响应而失败，`map_ws_error(...)` 会把这次握手失败转换成：

```text
ApiError::Transport(TransportError::Http { status, ... })
```

来源：

- [responses_websocket.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses_websocket.rs#L422)

这意味着 `426` 会作为一个普通的 HTTP 状态码被保留在 transport error 内部。

### 3. `426` 的专门分支

随后 WebSocket 流处理路径会检查一个特定状态码：

```text
status == StatusCode::UPGRADE_REQUIRED
```

如果命中，它会返回：

```text
WebsocketStreamOutcome::FallbackToHttp
```

来源：

- [client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1302)

这是最关键的特殊处理。`426` 不只是被记录下来；它会直接改变传输策略。

## 回退之后会发生什么

一旦返回 `FallbackToHttp`，调用方就会对当前 session 启用“粘性”的 HTTP 回退。

这会做两件事：

1. 为当前 session 禁用 WebSocket
2. 重置 WebSocket session 状态

来源：

- [client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1484)

之后，该 turn 会通过 HTTP Responses API 路径执行：

- `POST /responses`
- `Accept: text/event-stream`

来源：

- [responses.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses.rs#L104)

## 为什么 `426` 很重要

在普通 HTTP 语义里，`426 Upgrade Required` 表示服务器要求客户端切换协议。

在这个代码库里，这个通用 HTTP 语义被进一步转成了一个非常具体的产品行为：

- 如果 WebSocket upgrade 因为 `426` 被拒绝，客户端就会认为应该改用 HTTP 传输

这样可以避免在服务端或中间代理已经明确表示 upgrade 路径不可用的情况下，客户端还不断重复无效的 WebSocket 尝试。

## 与其他错误的区别

`426` 在这里是特殊的，因为它有专门的分支。

相比之下：

- `401 Unauthorized` 会进入认证恢复逻辑
- 普通网络错误会作为传输失败上抛
- 其他任意 HTTP 错误并不会自动意味着“切换到 HTTP/SSE”

来源：

- [client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1302)

因此，从客户端视角看：

- `426` 意味着传输回退
- `401` 意味着凭证恢复
- 其他错误意味着常规的失败/重试处理

## 测试证据

这里有一个显式测试验证了这一行为。

测试挂载了：

- 一个返回 `426` 的 WebSocket 握手 `GET .../responses`
- 一个成功的 SSE `POST /responses` 响应

随后断言：

- 进行过一次 WebSocket 尝试
- 进行过一次 HTTP 尝试
- 请求最终通过 HTTP 成功完成

来源：

- [websocket_fallback.rs](/app/data/workspace/codex-rs/core/tests/suite/websocket_fallback.rs#L27)

测试里的内联注释也说明了预期行为：

- 如果没有对 `426` 的特殊处理，sampling loop 会在切换传输方式之前不断重试 WebSocket 握手

来源：

- [websocket_fallback.rs](/app/data/workspace/codex-rs/core/tests/suite/websocket_fallback.rs#L49)

还有一个“粘性回退”测试表明，一旦回退被激活，后续 turn 会一直走 HTTP，不会重新尝试 WebSocket。

来源：

- [websocket_fallback.rs](/app/data/workspace/codex-rs/core/tests/suite/websocket_fallback.rs#L197)

## 对 MITM / 代理 的实际含义

如果你正在用反向代理或 MITM 拦截流量，并且想强制这个客户端退回到标准 HTTP/SSE，那么在当前代码里，最干净、最受支持的信号就是：在 WebSocket 握手上返回 HTTP `426 Upgrade Required`。

这比依赖随机网络错误更可靠，因为代码里明确把 `426` 识别为“应切换传输方式”的原因。

对于一个正常的 ChatGPT 驱动 provider，这意味着：

- 拦截发往 `/responses` 的 WebSocket 握手
- 返回 `426 Upgrade Required`
- 允许后续的 HTTP `POST /responses` 请求正常继续

## 最终结论

在本项目里，`426 Upgrade Required` 不只是一个偶然出现的 HTTP 错误。

它实际上是一个刻意使用的传输控制信号，效果如下：

- WebSocket 握手返回 `426`
- 客户端把它映射成 `FallbackToHttp`
- session 禁用 WebSocket 传输
- 请求随后通过标准 HTTP/SSE 成功继续
