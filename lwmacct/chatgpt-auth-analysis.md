# ChatGPT Chat Auth Analysis

## Scope

This note documents the auth path used by this project when the active provider is ChatGPT-backed and the app is making normal chat/model requests.

It focuses on two questions:

1. What `BASE_URL` is actually used for chat interaction requests?
2. What value is actually sent in `Authorization: Bearer ...`?

## Short Answer

- Chat login/auth endpoints use `https://auth.openai.com`.
- Chat interaction requests do not use that auth domain.
- The default ChatGPT backend base URL is `https://chatgpt.com/backend-api/`.
- When the current auth mode is ChatGPT, the model/API provider base URL becomes `https://chatgpt.com/backend-api/codex`.
- In ChatGPT auth mode, the outgoing bearer token is the ChatGPT `access_token`.
- In API key mode, the outgoing bearer token is the API key.

So for chat interaction:

- `BASE_URL`: `https://chatgpt.com/backend-api/codex`
- `Authorization: Bearer ...`: ChatGPT `access_token`, not `OPENAI_API_KEY`

## Key Findings

### 1. Default ChatGPT request base URL

The runtime config default for `chatgpt_base_url` is:

```text
https://chatgpt.com/backend-api/
```

Source:

- [core/src/config/mod.rs](/app/data/workspace/codex-rs/core/src/config/mod.rs#L2078)

When the active auth mode is `Chatgpt`, the provider layer switches to this default API base:

```text
https://chatgpt.com/backend-api/codex
```

Source:

- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)

### 2. Request auth header value

The bridge from login state into API auth constructs `CoreAuthProvider` with:

- `token = provider.api_key()` if a provider-level API key exists
- otherwise `token = experimental_bearer_token` if configured
- otherwise `token = auth.get_token()`

Source:

- [login/src/api_bridge.rs](/app/data/workspace/codex-rs/login/src/api_bridge.rs#L6)

`auth.get_token()` returns:

- API key in API key mode
- `tokens.access_token` in ChatGPT mode

Source:

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L279)

The request layer then sends:

```text
Authorization: Bearer {token}
```

and, for ChatGPT sessions, also:

```text
ChatGPT-Account-ID: {account_id}
```

Sources:

- [codex-api/src/auth.rs](/app/data/workspace/codex-rs/codex-api/src/auth.rs#L17)
- [codex-api/src/api_bridge.rs](/app/data/workspace/codex-rs/codex-api/src/api_bridge.rs#L202)

### 3. `OPENAI_API_KEY` exists, but is not the chat bearer in ChatGPT mode

During browser login, the login server may also exchange the ID token for an API key-like token and store it as `OPENAI_API_KEY` in `auth.json`.

Sources:

- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L1059)
- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L779)

However, when auth state is loaded:

- if `auth_mode == ApiKey`, `openai_api_key` becomes the active token source
- otherwise ChatGPT auth uses the stored ChatGPT tokens

Source:

- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L192)

That means a ChatGPT-authenticated session still uses `tokens.access_token` as the outgoing bearer for chat/model requests, even if `auth.json` also contains `OPENAI_API_KEY`.

## Call Stack

## A. Chat request base URL resolution

1. Config is loaded with default:
   `chatgpt_base_url = "https://chatgpt.com/backend-api/"`
2. Client setup asks the provider to resolve an API provider using current auth mode.
3. If auth mode is `Chatgpt`, provider base URL becomes:
   `https://chatgpt.com/backend-api/codex`

Relevant path:

1. [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L662)
2. [models-manager/src/manager.rs](/app/data/workspace/codex-rs/models-manager/src/manager.rs#L435)
3. [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)
4. [core/src/config/mod.rs](/app/data/workspace/codex-rs/core/src/config/mod.rs#L2078)

## B. Chat request bearer token resolution

1. Current auth is loaded from `AuthManager`.
2. `auth_provider_from_auth(...)` converts it to `CoreAuthProvider`.
3. If using normal ChatGPT login, `auth.get_token()` returns `tokens.access_token`.
4. API layer writes `Authorization: Bearer {token}`.

Relevant path:

1. [core/src/client.rs](/app/data/workspace/codex-rs/core/src/client.rs#L662)
2. [login/src/api_bridge.rs](/app/data/workspace/codex-rs/login/src/api_bridge.rs#L6)
3. [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L279)
4. [codex-api/src/auth.rs](/app/data/workspace/codex-rs/codex-api/src/auth.rs#L17)

## C. App-server path using the same auth material

Some app-server flows also build backend clients directly from auth and do the same thing:

1. read token from `auth.get_token()`
2. set `Authorization: Bearer {token}`
3. if available, set `ChatGPT-Account-ID`

Relevant path:

1. [app-server/src/codex_message_processor.rs](/app/data/workspace/codex-rs/app-server/src/codex_message_processor.rs#L1741)
2. [backend-client/src/client.rs](/app/data/workspace/codex-rs/backend-client/src/client.rs#L136)
3. [backend-client/src/client.rs](/app/data/workspace/codex-rs/backend-client/src/client.rs#L169)

## Auth Domain vs Chat Domain

These are easy to confuse, but they are different:

### Auth domain

Used for login, code exchange, and token refresh:

- `https://auth.openai.com/oauth/authorize`
- `https://auth.openai.com/oauth/token`
- `https://auth.openai.com/api/accounts/deviceauth/usercode`
- `https://auth.openai.com/api/accounts/deviceauth/token`

Sources:

- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L51)
- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L503)
- [login/src/server.rs](/app/data/workspace/codex-rs/login/src/server.rs#L705)
- [login/src/auth/manager.rs](/app/data/workspace/codex-rs/login/src/auth/manager.rs#L85)
- [login/src/device_code_auth.rs](/app/data/workspace/codex-rs/login/src/device_code_auth.rs#L67)
- [login/src/device_code_auth.rs](/app/data/workspace/codex-rs/login/src/device_code_auth.rs#L106)

### Chat backend domain

Used for actual chat/model/backend requests:

- `https://chatgpt.com/backend-api/`
- `https://chatgpt.com/backend-api/codex`

Sources:

- [core/src/config/mod.rs](/app/data/workspace/codex-rs/core/src/config/mod.rs#L2078)
- [model-provider-info/src/lib.rs](/app/data/workspace/codex-rs/model-provider-info/src/lib.rs#L184)

## Practical Conclusion

If you are tracing a real chat turn in this project and the session is logged in with ChatGPT:

- the request target is based on `https://chatgpt.com/backend-api/codex`
- the bearer credential is the ChatGPT `access_token`
- it is not using `OPENAI_API_KEY` as the request bearer for that path

If the session instead uses API key auth:

- the bearer token is the API key
- the provider base URL may be OpenAI-style instead of the ChatGPT backend

