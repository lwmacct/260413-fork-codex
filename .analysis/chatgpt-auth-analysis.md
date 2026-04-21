# ChatGPT 聊天认证分析

## 范围

这份说明文档记录了：当当前 provider 由 ChatGPT 驱动、应用正在发起普通 chat/model 请求时，本项目实际走的是哪条认证链路。

它主要回答三个问题：

1. 聊天交互请求实际使用的 `BASE_URL` 是什么？
2. 实际发送出去的 `Authorization: Bearer ...` 值是什么？
3. 最终与认证相关的请求头是如何组装出来的？

## 简短结论

- 聊天登录/认证端点使用的是 `https://auth.openai.com`。
- 聊天交互请求并不会使用这个认证域名。
- 默认的 ChatGPT 后端基础 URL 是 `https://chatgpt.com/backend-api/`。
- 当当前认证模式是 ChatGPT 时，model/API provider 的 base URL 会变成 `https://chatgpt.com/backend-api/codex`。
- 在 ChatGPT 认证模式下，对外发出的 bearer token 是 ChatGPT 的 `access_token`。
- 在 API key 模式下，对外发出的 bearer token 则是 API key。

因此，对于聊天交互来说：

- `BASE_URL`：`https://chatgpt.com/backend-api/codex`
- `Authorization: Bearer ...`：使用的是 ChatGPT `access_token`，而不是 `OPENAI_API_KEY`

## 关键发现

### 1. 默认 ChatGPT 请求基础 URL

运行时配置中 `chatgpt_base_url` 的默认值是：

```text
https://chatgpt.com/backend-api/
```

来源：

- [core/src/config/mod.rs](/app/data/workspace/codex-rs/core/src/config/mod.rs#L2078)

当当前认证模式是 `Chatgpt` 时，provider 层会切换到这个默认 API base：

```text
https://chatgpt.com/backend-api/codex
```

来源：

- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)

### 2. 请求认证头的值

从登录状态桥接到 API auth 时，会构造 `CoreAuthProvider`，其 token 选择逻辑是：

- 如果存在 provider 级别的 API key，则 `token = provider.api_key()`
- 否则，如果配置了 `experimental_bearer_token`，则 `token = experimental_bearer_token`
- 否则，`token = auth.get_token()`

来源：

- [login/src/api_bridge.rs](/app/data/workspace/codex-rs/login/src/api_bridge.rs#L6)

`auth.get_token()` 的返回值是：

- API key 模式下返回 API key
- ChatGPT 模式下返回 `tokens.access_token`

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L279)

随后，请求层会发送：

```text
Authorization: Bearer {token}
```

并且，对于 ChatGPT session，还会额外发送：

```text
ChatGPT-Account-ID: {account_id}
```

来源：

- [codex-api/src/auth.rs](/app/data/workspace/codex-rs/codex-api/src/auth.rs#L17)
- [codex-api/src/api_bridge.rs](/app/data/workspace/codex-rs/codex-api/src/api_bridge.rs#L202)

### 3. `OPENAI_API_KEY` 虽然存在，但在 ChatGPT 模式下不是聊天 bearer

在浏览器登录流程中，login server 还可能会把 ID token 换成一个类似 API key 的 token，并把它存成 `auth.json` 中的 `OPENAI_API_KEY`。

来源：

- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L1059)
- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L779)

但是，当认证状态被加载时：

- 如果 `auth_mode == ApiKey`，`openai_api_key` 会成为活动 token 来源
- 否则，ChatGPT 认证仍然使用已存储的 ChatGPT tokens

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L192)

这意味着：即使 `auth.json` 同时含有 `OPENAI_API_KEY`，一个通过 ChatGPT 认证的 session 在发起 chat/model 请求时，仍然会使用 `tokens.access_token` 作为实际 bearer。

## 调用栈

## A. 聊天请求 base URL 的解析

1. 配置加载时会带上默认值：
   `chatgpt_base_url = "https://chatgpt.com/backend-api/"`
2. 客户端初始化时，会根据当前 auth mode 让 provider 解析 API provider。
3. 如果 auth mode 是 `Chatgpt`，provider base URL 就会变成：
   `https://chatgpt.com/backend-api/codex`

相关路径：

1. [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L662)
2. [models-manager/src/manager.rs](/app/data/workspace/codex-rs/models-manager/src/manager.rs#L435)
3. [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)
4. [core/src/config/mod.rs](/app/data/workspace/codex-rs/core/src/config/mod.rs#L2078)

## B. 聊天请求 bearer token 的解析

1. 从 `AuthManager` 读取当前 auth。
2. `auth_provider_from_auth(...)` 把它转换成 `CoreAuthProvider`。
3. 在普通 ChatGPT 登录场景下，`auth.get_token()` 返回 `tokens.access_token`。
4. API 层写入 `Authorization: Bearer {token}`。

相关路径：

1. [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L662)
2. [login/src/api_bridge.rs](/app/data/workspace/codex-rs/login/src/api_bridge.rs#L6)
3. [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L279)
4. [codex-api/src/auth.rs](/app/data/workspace/codex-rs/codex-api/src/auth.rs#L17)

## C. app-server 路径复用同一份认证材料

有些 app-server 流程也会直接基于 auth 构造 backend client，逻辑也是一样的：

1. 从 `auth.get_token()` 读取 token
2. 设置 `Authorization: Bearer {token}`
3. 如果有的话，再设置 `ChatGPT-Account-ID`

相关路径：

1. [app-server/src/codex_message_processor.rs](/app/data/workspace/codex-rs/app-server/src/codex_message_processor.rs#L1741)
2. [backend-client/src/client.rs](/app/data/workspace/codex-rs/backend-client/src/client.rs#L136)
3. [backend-client/src/client.rs](/app/data/workspace/codex-rs/backend-client/src/client.rs#L169)

## 认证信息是如何组装出来的

这一部分重点说明：最终对外请求头是怎么拼出来的。

## 第 1 层：默认 client headers

共享的 login/default HTTP client 会安装：

- `User-Agent`
- `originator`
- 可选的 residency headers

来源：

- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L131)
- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L189)
- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L226)

重要细节：

- `User-Agent` 是由 originator、构建版本、OS 信息和 terminal detection 信息组合出来的。
- `originator` 会作为底层 reqwest client 的默认 HTTP header 插入。

## 第 2 层：provider 级别 headers

model provider 会提供：

- `base_url`
- 静态 `http_headers`
- 从环境变量派生的 `env_http_headers`
- query params

来源：

- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L157)
- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)

这些内容在 API 层会变成 `provider.headers`。

来源：

- [codex-api/src/provider.rs](/app/data/workspace/codex-rs/codex-api/src/provider.rs#L43)

## 第 3 层：每次请求的额外 headers

对于普通的 `responses` 调用，请求路径还会再加上一些 per-turn headers，例如：

- `x-client-request-id`
- `session_id`
- `x-openai-subagent`

来源：

- [codex-api/src/endpoint/responses.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses.rs#L88)
- [codex-api/src/requests/headers.rs](/app/data/workspace/codex-rs/codex-api/src/requests/headers.rs#L5)
- [codex-api/src/requests/headers.rs](/app/data/workspace/codex-rs/codex-api/src/requests/headers.rs#L13)

## 第 4 层：认证 headers

认证层最后再追加：

- `Authorization: Bearer {token}`
- 如果有的话，再加 `ChatGPT-Account-ID: {account_id}`

来源：

- [codex-api/src/auth.rs](/app/data/workspace/codex-rs/codex-api/src/auth.rs#L17)

这里的 token 和 account id 都来自 `CoreAuthProvider`，而 `CoreAuthProvider` 是根据当前 auth state 构造出来的：

- API key 模式：bearer token 是 API key
- ChatGPT 模式：bearer token 是 ChatGPT `access_token`
- ChatGPT 模式：account id 会作为单独的 `ChatGPT-Account-ID` header 附带出去

来源：

- [login/src/api_bridge.rs](/app/data/workspace/codex-rs/login/src/api_bridge.rs#L6)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L279)

## Header 合并顺序

对于普通 HTTP endpoint 请求，合并顺序是：

1. 先从 `provider.headers` 开始
2. 再扩展 `extra_headers`
3. 最后附加 auth headers

来源：

- [codex-api/src/endpoint/session.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/session.rs#L47)

这意味着：

- per-request headers 可以覆盖 provider headers
- auth headers 总是在这些层之后再附加

对于 websocket 风格请求，还会多一层：

1. 从 `provider.headers` 开始
2. 扩展 `extra_headers`
3. 用 `default_headers` 填补缺失项
4. 最后附加 auth headers

来源：

- [codex-api/src/endpoint/responses_websocket.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses_websocket.rs#L311)
- [codex-api/src/endpoint/responses_websocket.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses_websocket.rs#L328)
- [codex-api/src/endpoint/realtime_websocket/methods.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/realtime_websocket/methods.rs#L591)

一个很重要的实际结论是：auth headers 是在非认证层合并完成之后再追加的，所以最终的请求身份是在最后根据解析出的登录状态拼出来的，而不是提前固化在 provider 配置里。

## 最终聊天请求的具体形状

对于一个普通的、已通过 ChatGPT 认证的 `responses` 请求，最终对外请求大致是：

```text
POST https://chatgpt.com/backend-api/codex/responses
Authorization: Bearer <chatgpt access_token>
ChatGPT-Account-ID: <chatgpt account id>
originator: <originator>
User-Agent: <codex user agent>
session_id: <conversation id, when available>
x-client-request-id: <conversation id, when available>
x-openai-subagent: <subagent label, when applicable>
...provider headers...
...extra per-request headers...
```

关于 auth header 的精确预期，测试里也有覆盖：

- [codex-api/tests/clients.rs](/app/data/workspace/codex-rs/codex-api/tests/clients.rs#L241)

## 认证域名 与 聊天域名

这两者很容易混淆，但它们是不同的：

### 认证域名

用于登录、code exchange 和 token refresh：

- `https://auth.openai.com/oauth/authorize`
- `https://auth.openai.com/oauth/token`
- `https://auth.openai.com/api/accounts/deviceauth/usercode`
- `https://auth.openai.com/api/accounts/deviceauth/token`

来源：

- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L51)
- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L503)
- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L705)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L85)
- [login/src/device_code_auth.rs](/app/data/workspace/codex-rs/login/src/device_code_auth.rs#L67)
- [login/src/device_code_auth.rs](/app/data/workspace/codex-rs/login/src/device_code_auth.rs#L106)

### 聊天后端域名

用于实际的 chat/model/backend 请求：

- `https://chatgpt.com/backend-api/`
- `https://chatgpt.com/backend-api/codex`

来源：

- [core/src/config/mod.rs](/app/data/workspace/codex-rs/core/src/config/mod.rs#L2078)
- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)

## 实际结论

如果你正在追踪本项目中的一个真实 chat turn，并且该 session 是通过 ChatGPT 登录的：

- 请求目标是基于 `https://chatgpt.com/backend-api/codex`
- bearer 凭证是 ChatGPT 的 `access_token`
- `ChatGPT-Account-ID` 会作为单独的 header 发送
- 最终的请求身份是在 provider headers、request headers 合并完之后，再由 auth headers 拼出来
- 这条路径并不是用 `OPENAI_API_KEY` 作为请求 bearer

如果该 session 使用的是 API key auth：

- bearer token 就是 API key
- provider base URL 也可能是 OpenAI 风格，而不是 ChatGPT backend

## `access_token` 是如何刷新的

这一部分专门讨论受管的 ChatGPT 认证，也就是存储状态里 `auth_mode = "chatgpt"` 的情况。

这个区分很重要，因为代码里其实存在两种“看起来都像 ChatGPT”的认证模式：

- `Chatgpt`：受管认证，持有已存储的 `refresh_token`，会通过 `auth.openai.com` 的 OAuth refresh 流程刷新
- `ChatgptAuthTokens`：外部注入的 tokens，由外部 provider 刷新，不走同一条基于磁盘 refresh-token 的网络路径

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L31)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1531)

### 受管刷新路径的范围

对于 `auth_mode = "chatgpt"`，运行时不会永远使用同一份固定的 `access_token`。它会把下面这些内容存进 `AuthDotJson`：

- `tokens.access_token`
- `tokens.refresh_token`
- `tokens.id_token`
- `last_refresh`

来源：

- [login/src/auth/storage.rs](/app/data/workspace/codex-rs/login/src/auth/storage.rs#L28)
- [login/src/token_data.rs](/app/data/workspace/codex-rs/login/src/token_data.rs#L10)

### 触发条件

主要有两个入口。

#### 1. 返回 auth 之前的主动刷新

`AuthManager::auth()` 会先检查缓存中的 ChatGPT auth 是否已经 stale。如果 stale，就会在把 auth 返回给调用方之前先尝试执行 `refresh_token()`。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1244)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1482)

#### 2. 收到 401 之后的被动刷新

当 API 请求返回 `401 Unauthorized` 时，`UnauthorizedRecovery` 可以启动一个恢复状态机。在受管 ChatGPT 模式下，它会做：

1. 从存储做一次 guarded reload
2. 向 token authority 发起刷新

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L867)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1027)

### Staleness 规则

受管 ChatGPT auth 满足以下任一条件时会被视为 stale：

- `tokens.access_token` 是一个 JWT，并且它的 `exp` 已经过期
- `last_refresh` 早于配置的 refresh interval

当前 refresh interval 常量是 8 天。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L68)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1578)
- [login/src/token_data.rs](/app/data/workspace/codex-rs/login/src/token_data.rs#L122)

### 发网前的 guarded reload

`refresh_token()` 不会立刻去调用认证服务器。它会先从存储做一次 guarded reload。

这样做的目的是为了避免在以下场景里拿着过期的进程内状态去刷新：

- 另一个 Codex 实例已经替你刷新过 token
- 用户已经切换过账号

逻辑如下：

1. 读取缓存 auth
2. 提取当前 `account_id`
3. 只有在磁盘上的 account id 仍然匹配时才 reload auth
4. 如果 reload 后的 auth 已经不同了，就直接停止，并使用新的磁盘状态
5. 只有在 reload 后 auth 仍然相同的情况下，才真正去调用 token authority

如果 account id 不匹配，则刷新会以 account-mismatch 错误被拒绝。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1482)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1321)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L159)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L215)

### 刷新调用链

当受管刷新真正发生时，调用路径是：

1. `AuthManager::refresh_token()` 或 `AuthManager::refresh_token_from_authority()`
2. `refresh_token_from_authority_impl()`
3. `refresh_and_persist_chatgpt_token(...)`
4. `request_chatgpt_token_refresh(...)`
5. `POST https://auth.openai.com/oauth/token`

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1514)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1519)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1653)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L665)

### 刷新端点

真正的刷新端点常量是：

```text
https://auth.openai.com/oauth/token
```

代码也支持通过下面这个环境变量覆盖它，以便测试或定制环境使用：

```text
CODEX_REFRESH_TOKEN_URL_OVERRIDE
```

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L85)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L782)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L878)

### 刷新请求体

刷新请求是以 JSON 发送的，而不是 form-urlencoded。

精确 payload 形状如下：

```json
{
  "client_id": "app_EMoamEEZ73f0CkXaXp7hrann",
  "grant_type": "refresh_token",
  "refresh_token": "<stored refresh token>"
}
```

它来自 `RefreshRequest` 结构体以及硬编码的 `CLIENT_ID`。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L765)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L780)

### 刷新请求头与 UA

刷新请求使用的是共享的 `create_client()` HTTP client，而不是临时拼一个裸 reqwest client。也就是说，这个请求会继承标准的 Codex 默认 headers。

请求本身会显式添加：

- `Content-Type: application/json`

共享 client 会提供：

- `User-Agent: <computed Codex UA>`
- `originator: <originator value>`
- 可选的 `x-openai-internal-codex-residency: us`

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L175)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L679)
- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L189)
- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L226)

默认的 `originator` 是：

```text
codex_cli_rs
```

除非它被 `CODEX_INTERNAL_ORIGINATOR_OVERRIDE` 覆盖。

来源：

- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L34)
- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L35)

计算出来的 UA 格式是：

```text
{originator}/{build_version} ({os_type} {os_version}; {arch}) {terminal_token}
```

其中 `terminal_token` 来自 terminal detection 逻辑，例如 `TERM_PROGRAM`、terminal version 或 `TERM`。

来源：

- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L131)
- [terminal-detection/src/lib.rs](/app/data/workspace/codex-rs/terminal-detection/src/lib.rs#L173)
- [terminal-detection/src/lib.rs](/app/data/workspace/codex-rs/terminal-detection/src/lib.rs#L262)

还有一个重要的反向结论：

- 这个刷新请求不会附加 `Authorization: Bearer ...`
- 它也不会附加 `ChatGPT-Account-ID`

这些 headers 是给 ChatGPT backend 请求用的，而不是给 OAuth refresh 调用本身用的。

### 刷新响应如何处理与持久化

刷新响应会被解码成：

- `id_token: Option<String>`
- `access_token: Option<String>`
- `refresh_token: Option<String>`

随后，持久化逻辑只会更新那些确实返回了的字段：

- 如果存在 `id_token`，就解析它并替换 `tokens.id_token`
- 如果存在 `access_token`，就替换 `tokens.access_token`
- 如果存在 `refresh_token`，就替换 `tokens.refresh_token`
- 总是把 `last_refresh` 更新成 `now`

保存完成后，manager 会调用 `reload()`，让后续调用方看到更新后的 auth snapshot。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L772)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L644)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1653)
- [login/src/token_data.rs](/app/data/workspace/codex-rs/login/src/token_data.rs#L129)

### 错误分类

如果刷新端点返回 `401`，代码会解析 error payload，并识别几个已知的 refresh 失败原因：

- `refresh_token_expired`
- `refresh_token_reused`
- `refresh_token_invalidated`

这些都会被映射成 permanent failure。其他非 401 失败则被视为 transient transport/server failure。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L688)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L710)
- [login/src/auth/util.rs](/app/data/workspace/codex-rs/login/src/auth/util.rs#L1)

如果某次 permanent failure 发生在当前缓存 auth snapshot 上，该失败还会被缓存下来，避免运行时对同一份注定失败的 refresh 请求反复打网。

来源：

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L1526)

### 验证这些行为的测试覆盖

login 集成测试明确覆盖了上面这套 refresh 行为：

- 刷新成功后会更新存储中的 tokens
- access token 过期时会触发主动刷新
- 如果磁盘上的 auth 已经变化，guarded reload 会跳过网络请求
- account mismatch 会阻止刷新
- `401` 错误会被分类成 expired / reused 等 permanent reason
- `UnauthorizedRecovery` 会先 reload，再 refresh

来源：

- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L33)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L159)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L271)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L375)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L456)
- [login/tests/suite/auth_refresh.rs](/app/data/workspace/codex-rs/login/tests/suite/auth_refresh.rs#L615)

### 重要含义

对于普通的 ChatGPT 驱动聊天请求，实际 bearer 仍然是 `tokens.access_token`。

但对于受管的 `auth_mode = "chatgpt"`，这份 bearer 的续命方式通常是：

1. 维护一个已存储的 `refresh_token`
2. 调用 `POST https://auth.openai.com/oauth/token`
3. 把返回的 tokens 持久化回 auth storage

因此，chat bearer 是通过 auth service 刷新的，而不是通过复用 `OPENAI_API_KEY`。

## TLS、mTLS 与 ECH

这一部分回答的是另一个与 bearer auth 不同的问题：

- bearer auth 关心的是“HTTP headers 里发送了什么 token”
- TLS/mTLS 关心的是“传输连接本身是如何完成认证的”

一个重要区分是：

- mTLS 表示客户端也会向服务端出示自己的证书
- ECH 不是客户端证书认证；它是对 ClientHello/SNI 部分进行加密的 TLS 隐私特性

### 主聊天传输：启用了服务端证书校验

共享的、用于 websocket 传输的 TLS helper 会构造一个 rustls `ClientConfig`，其中包含：

- 本机/平台根证书
- 来自 `CODEX_CA_CERTIFICATE` 或 `SSL_CERT_FILE` 的可选自定义 CA bundle
- 基于 root store 的标准 server-auth 校验

配置构造方式如下：

```text
ClientConfig::builder()
  .with_root_certificates(root_store)
  .with_no_client_auth()
```

来源：

- [codex-client/src/custom_ca.rs](/app/data/workspace/codex-rs/codex-client/src/custom_ca.rs#L185)
- [codex-client/src/custom_ca.rs](/app/data/workspace/codex-rs/codex-client/src/custom_ca.rs#L215)

ChatGPT websocket client 在 responses websocket 和 realtime websocket 两条路径上都直接使用了这个 helper。

来源：

- [codex-api/src/endpoint/responses_websocket.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses_websocket.rs#L357)
- [codex-api/src/endpoint/realtime_websocket/methods.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/realtime_websocket/methods.rs#L557)

对于普通 HTTPS client，共享 builder 在这条聊天路径里只会在有配置时增加自定义 root certificates；它不会附加客户端身份材料。

来源：

- [codex-client/src/custom_ca.rs](/app/data/workspace/codex-rs/codex-client/src/custom_ca.rs#L179)

### 主聊天传输：没有 mTLS / 没有客户端证书认证

对于主聊天 HTTP/WebSocket 栈，最直接的证据就是：rustls websocket 配置最后调用的是：

```text
with_no_client_auth()
```

这意味着该路径不会向服务端出示客户端证书。

来源：

- [codex-client/src/custom_ca.rs](/app/data/workspace/codex-rs/codex-client/src/custom_ca.rs#L257)

我对仓库做过范围检查，在 `codex-client`、`codex-api`、`core`、`login` 的请求构造代码里，也没有发现主聊天传输使用：

- `reqwest::Identity`
- `with_client_auth_cert`
- `with_client_cert_resolver`

### OTEL exporter 支持 mTLS，但它是单独的一条路径

OTEL exporter 配置中确实有 mTLS 支持：

- `ca_certificate`
- `client_certificate`
- `client_private_key`

这些字段属于 `OtelTlsConfig`，而 core wiring 只会把它们传给 OTEL exporter 初始化逻辑。

来源：

- [config/src/types.rs](/app/data/workspace/codex-rs/config/src/types.rs#L371)
- [core/src/otel_init.rs](/app/data/workspace/codex-rs/core/src/otel_init.rs#L22)

在 OTEL transport 实现里：

- gRPC OTLP 使用 `TonicIdentity::from_pem(...)`
- HTTP OTLP 使用 `ReqwestIdentity::from_pem(...)`

前提是 `client_certificate` 和 `client_private_key` 都存在。

来源：

- [otel/src/otlp.rs](/app/data/workspace/codex-rs/otel/src/otlp.rs#L34)
- [otel/src/otlp.rs](/app/data/workspace/codex-rs/otel/src/otlp.rs#L101)
- [otel/src/otlp.rs](/app/data/workspace/codex-rs/otel/src/otlp.rs#L148)

所以：

- OTEL exporter 路径：可以做 mTLS
- 主 ChatGPT chat/backend 路径：看起来并不会做 mTLS

### ECH：本项目中没有发现显式支持或配置

我没有在本项目里找到任何为聊天传输配置 ECH 相关 API 或选项的代码。

具体来说，仓库范围检查没有发现使用下面这些 ECH 相关标识符：

- `with_ech`
- `Encrypted Client Hello`
- `encrypted client hello`

而主 websocket 聊天路径里唯一显式出现的 rustls client config，就是前面那个“root store + `with_no_client_auth()`”的最小构造方式。

因此，对这个代码库最稳妥的结论是：

- 存在标准的 TLS 服务端证书校验
- 主聊天路径不存在客户端证书认证
- 本项目中没有显式 ECH 配置

## 最终判断

对于本项目里的 ChatGPT 驱动聊天路径：

- `BASE_URL` 实际上是 `https://chatgpt.com/backend-api/codex`
- `Authorization: Bearer ...` 使用的是 ChatGPT `access_token`
- 普通 ChatGPT 聊天请求并不会把 `OPENAI_API_KEY` 当成 bearer
- `access_token` 会通过 `refresh_token` 调用 `https://auth.openai.com/oauth/token` 刷新
- 主聊天传输会校验服务端证书
- 主聊天传输看起来并不会使用 mTLS / 客户端证书
- 项目代码里没有配置 ECH

## 默认传输方式：优先 WebSocket，HTTP/SSE 作为回退

对于 ChatGPT 驱动的 `Responses` 路径，运行时并不是默认先走 plain HTTP/SSE。

相反，turn-scoped client logic 会这样做：

1. 先检查 Responses-over-WebSocket 是否启用
2. 如果启用，就先尝试 Responses WebSocket 传输
3. 如果这条路径发生 fallback，就把 session 切回 HTTP，并使用 HTTP Responses API

core 里的注释写得很明确：

- “prefer the Responses WebSocket transport ... and fall back to the HTTP Responses API transport otherwise”

来源：

- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1431)
- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1444)

### “启用”是什么意思

`responses_websocket_enabled()` 在以下任一条件满足时会返回 false：

- `provider.supports_websockets == false`
- 当前 session 因为 fallback 已经禁用了 websocket
- SSE fixture override 处于激活状态

来源：

- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L644)

### HTTP/SSE 路径的形状

HTTP 流式路径就是标准的 Responses streaming：

- `POST /responses`
- `Accept: text/event-stream`

来源：

- [codex-api/src/endpoint/responses.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses.rs#L104)

### WebSocket 路径的形状

WebSocket 路径使用的是同一个逻辑上的 `responses` endpoint，只不过会把 provider URL 转成 websocket URL，并连接到 `ws(s)://.../responses`。

来源：

- [codex-api/src/endpoint/responses_websocket.rs](/app/data/workspace/codex-rs/codex-api/src/endpoint/responses_websocket.rs#L299)

## HTTP/SSE 回退是如何触发的

当前代码里最重要、也最显式的 fallback trigger 是：

- 如果 WebSocket connect 返回 HTTP `426 Upgrade Required`

那么运行时就会返回 `FallbackToHttp`。

来源：

- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1302)

回退一旦被激活，客户端就会永久禁用当前 Codex session 的 WebSocket，并重置 websocket state，因此后续 turn 都会使用 HTTP 传输。

来源：

- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1484)

### 对 MITM / 代理 的实际含义

如果你想有意把本项目从 Responses WebSocket 强制切到标准 HTTP/SSE，那么当前 fallback 路径认可的、最干净的服务端信号就是：在 websocket handshake 上返回 `426 Upgrade Required`。

相比之下，随机的网络错误或无关的 HTTP 状态码，并不是这条分支里显式定义的“切换到 HTTP”的信号。

## 如何通过配置强制标准 HTTP/SSE

本项目并没有对普通聊天 turn 暴露一个全局的 `transport = "http"` 配置项。

它实际暴露的是一个 provider capability flag：

- `supports_websockets`

如果当前激活的 provider 把 `supports_websockets = false`，那么普通 turn 路径就会跳过 websocket 分支，直接使用 HTTP Responses API 传输。

来源：

- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L121)
- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1447)

### 一个重要的配置限制

`model_provider` 只能从 `model_providers` map 里选择一个 provider ID，而用户自定义的 `model_providers` 不能覆盖内置保留 ID，比如 `openai`。

因此，如果你想仅通过配置得到一个 HTTP/SSE 变体，受支持的做法是：

1. 定义一个新的自定义 provider entry
2. 设置 `wire_api = "responses"`
3. 设置 `supports_websockets = false`
4. 让 `model_provider` 指向这个自定义 provider ID

来源：

- [config/src/config_toml.rs](/app/data/workspace/codex-rs/config/src/config_toml.rs#L74)
- [config/src/config_toml.rs](/app/data/workspace/codex-rs/config/src/config_toml.rs#L191)
- [config/src/config_toml.rs](/app/data/workspace/codex-rs/config/src/config_toml.rs#L752)

## SOCKS5 代理相关发现

这个话题里，“SOCKS5” 实际上分成两种不同含义：

1. Codex 自己的 managed/local proxy 可以暴露一个 SOCKS5 listener
2. ChatGPT 聊天传输本身是否会直接通过上游 SOCKS5 proxy 发送

### 1. 受管 Codex proxy 支持 SOCKS5

本地的 `codex-network-proxy` 明确支持 SOCKS5 listener，并且相关配置面包括：

- `enable_socks5`
- `socks_url`
- `enable_socks5_udp`

来源：

- [network-proxy/README.md](/app/data/workspace/codex-rs/network-proxy/README.md#L1)
- [config/src/permissions_toml.rs](/app/data/workspace/codex-rs/config/src/permissions_toml.rs#L141)

这意味着：受管的 network-proxy 子系统可以被配置为对外暴露一个本地 SOCKS5 端点。

### 2. 聊天传输没有暴露专门的 SOCKS5 配置项

对于普通聊天 HTTP 流量，默认 client 是从 `reqwest::Client::builder()` 构造出来的，在这条路径上并没有额外增加显式的 `reqwest::Proxy` 或项目级 SOCKS5 配置。

来源：

- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L212)

在 sandbox 模式下，它甚至会显式用 `.no_proxy()` 关闭 proxy autodetection。

来源：

- [login/src/auth/default_client.rs](/app/data/workspace/codex-rs/login/src/auth/default_client.rs#L219)

因此，一个站得住脚的仓库级结论是：

- 没有专门的普通聊天配置字段，比如 `"chat.socks5_proxy"`
- 在非 sandbox 环境里，HTTP 行为仍然可能受 reqwest 默认的环境变量/系统代理发现机制影响
- 在 sandbox 环境里，默认的 HTTP 聊天 client 会显式避开代理

对于 websocket 聊天路径，这个仓库在依赖层面启用了 tungstenite 的 proxy features，但我没有找到一个项目级配置面，能够把某个明确的上游 SOCKS5 proxy 接到普通 ChatGPT websocket connect 上。

## `wire_api = "responses"`：含义与作用域

`wire_api` 是一个 provider 级别的协议选择器，而不是一个 request 级别的参数。

这个字段存在于 `ModelProviderInfo` 上，也就是 provider 定义本身。

来源：

- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L72)

在当前代码库中：

- `WireApi` 只有一个受支持值：`Responses`
- `wire_api = "chat"` 已经被显式拒绝，因为它已被移除

来源：

- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L36)
- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L40)

### 这个参数实际表示什么

它告诉客户端：这个 provider 使用的是哪一种 application-layer API 协议。

在本项目里，这实际上意味着：

- 构造并发送 `ResponsesApiRequest`
- 使用 `responses` endpoint 的形状
- 按 Responses API 事件流格式来解析返回流

来源：

- [codex-api/src/common.rs](/app/data/workspace/codex-rs/codex-api/src/common.rs#L155)
- [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L1444)

### 它不表示什么

`wire_api` 本身并不决定：

- bearer auth 到底用 API key 还是 ChatGPT tokens
- 传输到底走 WebSocket 还是 HTTP/SSE
- 是否会使用代理

这些关注点由其他配置或运行时逻辑控制。

### 作用域

这个参数只作用于 `provider` 定义。

也就是说：

- 它应该放在 `model_providers.<id>` 下面
- 当 `model_provider = "<id>"` 选中某个 active provider 时，它才会被间接生效

来源：

- [config/src/config_toml.rs](/app/data/workspace/codex-rs/config/src/config_toml.rs#L74)
- [config/src/config_toml.rs](/app/data/workspace/codex-rs/config/src/config_toml.rs#L191)
