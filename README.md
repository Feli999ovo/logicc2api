# logicc2api

**English** · [简体中文](README.zh-CN.md)

A local OpenAI-compatible adapter for [Logicc](https://app.logicc.com), with account pool management and a web console. It converts Logicc's AI SDK v6 event stream into OpenAI Chat Completions responses and shows account health, traffic, latency, and request history in one place.

Built with Node.js and plain HTML/CSS/JavaScript. **No runtime dependencies, database setup, or frontend build step.** The web interface is currently in Simplified Chinese; documentation is available in English and Chinese.

## Features

| Area | What it provides |
| --- | --- |
| OpenAI-compatible API | Model listing and streaming/non-streaming Chat Completions |
| Account pool | Add, edit, remove, enable/disable, search, and bulk-import accounts |
| Credential refresh | On-demand Clerk JWT refresh, shared concurrent refreshes, and persistence of rotated client cookies |
| Scheduling | Prefer the least-busy accounts, round-robin between ties, and cool down failing accounts |
| Dashboard | Request trends, success rate, time to first text, and account availability |
| Models and logs | Live model discovery, observed model latency, request details, filters, and CSV export |
| Playground | Send a real streaming request, choose an upstream tool, and stop the local request |
| Local storage | Keep configuration and request metadata across restarts; credentials are omitted from management responses |

The console polls local state every five seconds. Account checks and model synchronization fetch the upstream model list; the playground sends real generation requests and uses your upstream quota.

## Quick start

Requires **Node.js 22 or newer** and a Logicc account/session you can use.

```bash
git clone https://github.com/JamesZhaoY/logicc2api.git
cd logicc2api
npm start
```

There is no dependency installation step. You can also run `node server.mjs` directly.

1. Open **<http://127.0.0.1:18789/>**.
2. Go to **号池管理 → 添加账户** (Account pool → Add account), enter your credentials, and click **检测** (Check).
3. Point your client at **`http://127.0.0.1:18789/v1`**. Unless you configure `LOGICC_API_KEY`, use any non-empty API key, such as `local`.

To select a different port:

```bash
node server.mjs 18790
```

The server binds only to `127.0.0.1`. Command-line port selection takes precedence over `PORT`.

## Account credentials

The account form accepts any of the following:

| Input | Behavior |
| --- | --- |
| `__client` Cookie + Session ID | Obtain fresh session JWTs on demand |
| `__client` Cookie + Session JWT | Read the Session ID from the JWT's `sid` claim, then refresh on demand |
| Session JWT only | Use it until it expires; replace it manually or push a new JWT through `/__set_token` |

For your logged-in account, open browser DevTools → Application → Cookies → `clerk.logicc.com` and copy the **value** of `__client`. A Session ID starts with `sess_`. A Session JWT can be obtained in the Logicc page's console:

```js
await window.Clerk.session.getToken({ skipCache: true })
```

The `__session` cookie contains a short-lived session JWT; it is not the same credential as `__client`. A session can be revoked or signed out, so automatic refresh does not imply permanent access.

When editing an account, leaving a credential field blank preserves its saved value. Changing the client cookie or Session ID clears the cached JWT. Disabling or removing an account affects the local pool; it does not delete the upstream account.

### Bulk import

Paste a JSON array or upload a JSON file in **批量导入** (Bulk import):

```json
[
  {
    "name": "Work account",
    "clientCookie": "YOUR_CLIENT_COOKIE",
    "sessionId": "sess_YOUR_SESSION_ID",
    "enabled": true
  }
]
```

`{ "accounts": [...] }` is also accepted. JWT-only entries use `name` and `token`. The pool holds up to 200 accounts. Duplicate Session IDs are rejected, and the whole batch is validated before any account is added.

## API usage

### List models

```bash
curl http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer local'
```

Available models are read from the selected upstream account rather than hardcoded.

### Generate a response

```bash
curl http://127.0.0.1:18789/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local' \
  -d '{
    "model": "default",
    "messages": [{"role": "user", "content": "Say hello in one sentence."}]
  }'
```

### Stream a response

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local' \
  -d '{
    "model": "default",
    "stream": true,
    "stream_options": {"include_usage": true},
    "messages": [{"role": "user", "content": "Say hello in one sentence."}]
  }'
```

Omitting `model`, or using `default`, selects the upstream default model (or the first available model). Streaming responses use `chat.completion.chunk` and end with `[DONE]`. Set `stream: false`, or omit it, for a complete JSON response.

Upstream token usage is converted when available. When it is absent, API responses use zero counters for compatibility; the console marks usage as unavailable rather than estimating it.

### Tool mapping and compatibility limits

- `reasoning_effort` is forwarded to the upstream API.
- Function names `web_search` / `web-search` map to the upstream `web-search` tool; `image_generation` / `image-generation` map to `image-generation`. These invoke upstream built-in tools when available for the account/model.
- Other function definitions are added to the prompt as a JSON Schema convention. **Standard OpenAI `tool_calls` execution is not implemented.** Upstream tool events appear as text markers such as `[tool:web-search]`.
- Each request creates a new upstream chat. Conversation messages are combined into one prompt; native upstream multi-turn state is not preserved.
- Attachments are not forwarded. Image-tool output is displayed as text in the playground.
- This adapter implements `/v1/models` and `/v1/chat/completions`; it does not implement the entire OpenAI API.

## Scheduling and refresh behavior

- Enabled, eligible accounts with the fewest in-flight requests are preferred. Ties use round-robin selection, and one account handles the entire chat request.
- HTTP 429 triggers a 60-second cooldown; 401/403 trigger five minutes; upstream server errors trigger 15 seconds. Later requests skip cooling accounts.
- An eligible account can participate again when its cooldown ends. Checking or updating an account also allows recovery. A failed request is not replayed on a different account.
- A JWT with fewer than ten seconds remaining is refreshed on demand. An upstream 401 triggers a forced refresh and one retry.
- Concurrent refreshes for the same account share one operation. New `__client` values returned by Clerk are persisted.
- Clerk refresh uses `POST /v1/client/sessions/{sid}/tokens`.

## Configuration and data

| Environment variable | Purpose |
| --- | --- |
| `PORT` | Listening port; default `18789` |
| `LOGICC_DATA_FILE` | State path; default `data/state.json` in the project directory |
| `LOGICC_API_KEY` | Optional Bearer key required for `/v1/*` |
| `LOGICC_API_BASE` | Upstream base URL; default `https://app.logicc.com/api` |
| `LOGICC_CLERK_BASE` | Clerk base URL; default `https://clerk.logicc.com` |
| `LOGICC_CLIENT_COOKIE` | Import a default account into an empty pool |
| `LOGICC_SESSION_ID` | Session ID for the environment-imported account; alternatively read from a JWT |
| `LOGICC_TOKEN` | Initial JWT for the environment-imported account |
| `LOGICC_TOKEN_FILE` | External JWT file for the environment-imported account |
| `LOGICC_TOKEN_CMD` | External JWT refresh command for that account; configured only through the process environment |

The project does not load `.env` files automatically. Export variables in your shell or, on a compatible Node.js version, start with `node --env-file=.env server.mjs`.

Existing saved accounts take precedence over initial environment credentials, so old environment values do not overwrite rotated cookies. If you use an external refresh command or token file, keep providing its configuration when starting the service.

`data/state.json` contains credentials, the cached model catalog, cooldowns, and at most 2,000 recent request metadata records. Writes use a temporary file plus rename, with a requested file mode of `0600`. The file is **plaintext**, and filesystem permission enforcement depends on the platform. Runtime data, environment files, logs, backups, and browser test artifacts are excluded from Git.

The dashboard summarizes retained chat records from the previous 24 hours. Model discovery and account checks appear in logs but are excluded from chat metrics. The log page displays the latest 200 records. First-text latency includes credential refresh, chat creation, and network time. Prompts and generated answers are not stored in request logs.

The console is intended for a **single local process**, with no separate admin login. Management requests require the local origin/Host checks and `X-Logicc-Admin: 1`; `LOGICC_API_KEY` protects `/v1/*`, not the console. The compatibility endpoint `POST /__set_token` accepts a raw JWT, optionally with `?account_id=<id>`, and permits the Logicc page origin. JWTs are matched to accounts by Session ID.

Do not run multiple instances against the same state file. Use separate `LOGICC_DATA_FILE` paths and ports for independent instances.

## Development and tests

```bash
npm test
```

Tests use Node's built-in test runner and a local mock upstream. They cover account CRUD and atomic import, credential masking and file permissions, scheduling and cooldowns, forced refresh and cookie rotation, concurrent refreshes, JSON/SSE conversion, tool forwarding, usage and failure statistics, origin restrictions, and malformed input. They do not need real credentials or consume upstream quota.

```text
server.mjs              HTTP routes, upstream requests, and protocol conversion
pool.mjs                Account storage, credential refresh, scheduling, and metrics
public/                 Web console, served without a build step
test/adapter.test.mjs    Integration and protocol checks
```

## License

[MIT](LICENSE) © 2026 JamesZhaoY.
