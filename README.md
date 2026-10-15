# logicc2api

**English** · [简体中文](README.zh-CN.md)

> **Disclaimer:** Please read the [disclaimer and usage responsibilities (Simplified Chinese)](DISCLAIMER.md) before use. It covers account authorization, third-party service risks, data handling, and limitations of liability. The [MIT License](LICENSE) continues to govern the software license.

A local OpenAI-compatible adapter for [Logicc](https://app.logicc.com), with account pool management and a web console. It converts Logicc's AI SDK v6 event stream into OpenAI Chat Completions / Responses and Anthropic Messages responses and shows account health, traffic, latency, and request history in one place.

Built with Node.js and plain HTML/CSS/JavaScript. **No runtime dependencies, database setup, or frontend build step.** The web interface is currently in Simplified Chinese; documentation is available in English and Chinese.

## Features

| Area | What it provides |
| --- | --- |
| Protocol compatibility | Model listing; JSON and SSE for Chat Completions, Responses, and Messages |
| Client tool calls | Standard tool_calls / function_call / tool_use and matching tool-result round trips |
| Account pool | Add, edit, remove, enable/disable, search, and bulk-import accounts |
| Credential refresh | On-demand Clerk JWT refresh, shared concurrent refreshes, and persistence of rotated client cookies |
| Scheduling | Prefer the least-busy accounts, round-robin between ties, and cool down failing accounts |
| Dashboard | Request trends, success rate, time to first text, and account availability |
| Models and logs | Live model discovery, observed model latency, request details, filters, and CSV export |
| Playground | Send a real streaming request, choose an upstream tool, and stop the local request |
| Local storage | Keep configuration and request metadata across restarts; credentials are omitted from management responses |

The console polls local state every five seconds. Account checks and model synchronization fetch the upstream model list; the playground sends real generation requests and uses your upstream quota.

## Quick start

Requires **Node.js 22 or newer** and a Logicc account/session you can use. For a container-based setup without installing Node.js on the host, see [Docker deployment](#docker-deployment).

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

Native startup binds to `127.0.0.1` by default. Command-line port selection takes precedence over `PORT`.

## Docker deployment

Requires Docker Engine with the Compose plugin, or Docker Desktop. The repository includes a [Dockerfile](Dockerfile) and [compose.yaml](compose.yaml); the image is built locally from source.

### Start with Docker Compose

```bash
git clone https://github.com/JamesZhaoY/logicc2api.git
cd logicc2api
docker compose up -d --build --wait
```

If you already cloned the repository, run the last command from its root directory. Open **<http://127.0.0.1:18789/>**, add an account, and use **`http://127.0.0.1:18789/v1`** as your client's Base URL.

The image uses Node.js 24 Alpine and runs as the non-root `node` user. Inside the container, `LOGICC_HOST=0.0.0.0` allows Docker's port forwarding to reach the server. Compose publishes the port only on the host's `127.0.0.1`. The image health check calls `/health`; a healthy container means the service is running, not that an upstream account has passed a check.

### Change the port or API key

Create a local `.env` file next to `compose.yaml`, for example:

```dotenv
LOGICC_PORT=18790
LOGICC_API_KEY=replace-with-your-own-key
```

```bash
docker compose up -d --wait
```

Then use `http://127.0.0.1:18790/` and supply the configured Bearer key for `/v1/*`. Both variables are optional; the defaults are port `18789` and no API key requirement. Use another port if a native instance already occupies `18789`. Keep the settings in `.env` so subsequent Compose commands use the same configuration. Compose reads these values through [environment interpolation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/); the file is ignored by Git and excluded from the image build.

`LOGICC_PORT` changes the container port and published port together because the server validates the HTTP Host port. When writing a custom `docker run` command, keep the two ports equal and set `PORT` inside the container if changing the default. An API key protects `/v1/*`; it does not add an admin login to the console.

### Data, logs, and upgrades

Compose stores `/app/data` in the named volume `logicc2api_logicc-data` (or `<project-name>_logicc-data` when using `-p`). Accounts, request records, and rotated credentials survive container recreation. The image build excludes your local `data/`, `.env`, logs, and backups; an existing native installation is not imported automatically. Add your accounts through the console when starting a fresh container.

```bash
# Status and logs
docker compose ps
docker compose logs -f --tail=100 logicc2api

# Update source and rebuild; retain the same data volume
git pull --ff-only
docker compose up -d --build --wait

# Stop and remove containers while keeping account data
docker compose down
```

Do not add `--volumes` / `-v` to `docker compose down` unless you intend to delete the account data. Named volume behavior is documented in [Docker's volume guide](https://docs.docker.com/engine/storage/volumes/).

For deployment on a remote server, access the local-only console through an SSH tunnel from your computer:

```bash
ssh -N -L 18789:127.0.0.1:18789 your-user@your-server
```

Then open `http://127.0.0.1:18789/` on your computer. Adjust both port numbers if you configured another port. The existing local-origin checks remain in effect; public-domain hosting requires a separate authentication/access design.

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

### Responses protocol

`POST /v1/responses` accepts string or message-array `input` and supports `instructions`, `reasoning.effort`, function tools, `tool_choice`, and `stream`.

```bash
curl http://127.0.0.1:18789/v1/responses \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local' \
  -d '{"model":"default","input":"Introduce yourself in one sentence.","stream":true,"store":false}'
```

Streaming includes `response.created`, `response.output_text.delta`, `response.function_call_arguments.delta`, and `response.completed`; failures emit `response.failed`. Function calls appear as `function_call` output items. Execute the function in your client and return `function_call_output` with its `call_id`. Responses `custom` tools support string inputs as `custom_tool_call` items and accept `custom_tool_call_output` results.

The default `store: true` temporarily retains responses and conversation history **in process memory**, bounded to 30 minutes, 128 entries, and 16 MiB with oldest-entry eviction. Continue with `previous_response_id`, retrieve with `GET /v1/responses/{id}`, or delete with `DELETE /v1/responses/{id}`. A restart clears this cache; it is never written to the account file. With `store: false`, send the full conversation history yourself. Top-level `instructions` are not automatically inherited by the next request.

### Messages protocol

`POST /v1/messages` supports `system`, text content blocks, `tools[].input_schema`, `tool_choice`, `tool_use`, and `tool_result`. If a proxy API key is configured, use either `x-api-key` or Bearer authentication.

```bash
curl http://127.0.0.1:18789/v1/messages \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: local' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{"model":"default","max_tokens":512,"stream":true,"messages":[{"role":"user","content":"Introduce yourself in one sentence."}]}'
```

For SDK configuration, use `http://127.0.0.1:18789/v1` as the OpenAI `baseURL`, and `http://127.0.0.1:18789` as the Anthropic `baseURL` (without `/v1`).

SSE responses use `message_start`, `content_block_start/delta/stop`, `message_delta`, and `message_stop`. Tool arguments are emitted as `input_json_delta`, with `stop_reason: tool_use`. Execute the tool in your client, then send a user message containing `tool_result` with the matching `tool_use_id`.

`POST /v1/messages/count_tokens` returns a local estimate with the header `X-Logicc-Token-Count: estimated`. This helps clients estimate context size; it is not the upstream tokenizer's count and must not be used for billing.

### Client function calling

All three protocols share tool handling and preserve call/result IDs:

| Protocol | Tool definition | Returned call | Client result |
| --- | --- | --- | --- |
| Chat Completions | `type: function` and nested `function` | `message.tool_calls`; `finish_reason: tool_calls` | `role: tool` + `tool_call_id` |
| Responses | `type: function` with top-level `name/parameters` | `output[].type: function_call` | `type: function_call_output` + `call_id` |
| Messages | `name` + `input_schema` | `content[].type: tool_use`; `stop_reason: tool_use` | `type: tool_result` + `tool_use_id` |

Example Chat Completions function request:

```json
{
  "model": "default",
  "messages": [{"role": "user", "content": "Get the current weather in Hangzhou using get_weather."}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "The client looks up live weather for a city",
      "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
      }
    }
  }],
  "tool_choice": "required"
}
```

Your **client** executes the returned function. Include the original user message, the assistant message containing `tool_calls`, and a `role: tool` result message in the next request. Sending only the result loses the context and call-ID association.

Supported choices include `auto`, `none`, `required` / Messages `any`, named tools, and disabling parallel calls. `none` does not interpret text JSON as an executable call. If a required call cannot be generated, the adapter returns an explicit error rather than fabricating success. Validation covers JSON syntax, declared tool names, required fields, basic types, enum values, and basic object property constraints.

**Implementation boundary:** Logicc's current chat endpoint does not register arbitrary client functions. This adapter uses a structured prompt to generate tool requests and converts them into the client's protocol. Plain text streams immediately; responses with client tools are buffered for JSON validation before protocol events are sent. Reliability depends on the upstream model. This is not native constrained decoding, full JSON Schema `strict` validation, or a custom grammar validator. The adapter never executes your client's shell, file, or network tools.

### Upstream built-ins and remaining limits

- To request Logicc's built-in tools, send `tools: [{"type":"web_search"}]` or `[{"type":"image_generation"}]`; `web_search_preview` also maps to built-in search. The console playground uses this form.
- **A function named `web_search` remains a client function and is no longer silently replaced by upstream search.** This fixes the previous tool-routing behavior.
- Provider-executed tools are not executed a second time by the client. Built-in tool activity may still appear as text markers.
- Each request creates an upstream chat; conversation roles, tool calls, and results are preserved while being serialized into one prompt.
- `reasoning_effort` / Responses `reasoning.effort` are forwarded; provider-specific reasoning/thinking blocks are not exposed.
- `max_tokens`, `max_completion_tokens`, and `max_output_tokens` are length instructions, not hard upstream token caps. `temperature` / `top_p` do not control upstream sampling.
- Attachments, Responses JSON Schema text output, background tasks, WebSockets, the Conversations API, and the complete OpenAI / Anthropic API are not implemented. Image-tool output is displayed as text in the playground.

Protocol references: [OpenAI function calling](https://developers.openai.com/api/docs/guides/function-calling), [Responses streaming events](https://developers.openai.com/api/reference/resources/responses/streaming-events), and [Anthropic Messages streaming](https://platform.claude.com/docs/en/build-with-claude/streaming).

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
| `LOGICC_HOST` | Bind address; native default `127.0.0.1`, container default `0.0.0.0` |
| `LOGICC_DATA_FILE` | State path; default `data/state.json` in the project directory |
| `LOGICC_API_KEY` | Optional Bearer key required for `/v1/*` |
| `LOGICC_API_BASE` | Upstream base URL; default `https://app.logicc.com/api` |
| `LOGICC_CLERK_BASE` | Clerk base URL; default `https://clerk.logicc.com` |
| `LOGICC_CLIENT_COOKIE` | Import a default account into an empty pool |
| `LOGICC_SESSION_ID` | Session ID for the environment-imported account; alternatively read from a JWT |
| `LOGICC_TOKEN` | Initial JWT for the environment-imported account |
| `LOGICC_TOKEN_FILE` | External JWT file for the environment-imported account |
| `LOGICC_TOKEN_CMD` | External JWT refresh command for that account; configured only through the process environment |

Native Node.js startup does not load `.env` files automatically. Export variables in your shell or, on a compatible Node.js version, start with `node --env-file=.env server.mjs`. Docker Compose reads `.env` for the `LOGICC_PORT` and `LOGICC_API_KEY` mappings above; `LOGICC_PORT` is a Compose setting, not a native server setting. Other environment variables must be explicitly added to the Compose service if needed.

Existing saved accounts take precedence over initial environment credentials, so old environment values do not overwrite rotated cookies. If you use an external refresh command or token file, keep providing its configuration when starting the service.

`data/state.json` contains credentials, the cached model catalog, cooldowns, and at most 2,000 recent request metadata records. Writes use a temporary file plus rename, with a requested file mode of `0600`. The file is **plaintext**, and filesystem permission enforcement depends on the platform. Runtime data, environment files, logs, backups, and browser test artifacts are excluded from Git.

The dashboard summarizes retained chat records from the previous 24 hours. Model discovery and account checks appear in logs but are excluded from chat metrics. The log page displays the latest 200 records. First-text latency includes credential refresh, chat creation, and network time. Prompts and generated answers are not stored in request logs.

The console is intended for a **single local process**, with no separate admin login. Management requests require the local origin/Host checks and `X-Logicc-Admin: 1`; `LOGICC_API_KEY` protects `/v1/*`, not the console. The compatibility endpoint `POST /__set_token` accepts a raw JWT, optionally with `?account_id=<id>`, and permits the Logicc page origin. JWTs are matched to accounts by Session ID.

Do not run multiple instances against the same state file. Use separate `LOGICC_DATA_FILE` paths and ports for independent instances.

## Development and tests

```bash
npm test
```

Tests use Node's built-in test runner and a local mock upstream. They cover account CRUD and atomic import, credential masking and file permissions, scheduling and cooldowns, forced refresh and cookie rotation, concurrent refreshes, JSON/SSE conversion, tool-call/result round trips across all three APIs, usage and failure statistics, origin restrictions, and malformed input. They do not need real credentials or consume upstream quota.

```text
server.mjs              HTTP routes, upstream calls, and temporary Responses history
protocols.mjs           Request, tool-call, JSON, and SSE conversion for all three APIs
pool.mjs                Account storage, credential refresh, scheduling, and metrics
public/                 Web console, served without a build step
test/adapter.test.mjs    Integration and protocol checks
Dockerfile              Non-root container image and service health check
compose.yaml            Local port mapping and persistent account data
```

## License

[MIT](LICENSE) © 2026 JamesZhaoY.
