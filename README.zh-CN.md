# logicc2api

[English](README.md) · **简体中文**

为 [Logicc](https://app.logicc.com) 提供 OpenAI 兼容接口的本地代理，内置号池管理和 Web 控制台。将 Logicc 的 AI SDK v6 数据流转换成 OpenAI Chat Completions 响应，集中查看账户状态、流量、模型延迟和请求日志。

使用 Node.js 与原生 HTML/CSS/JavaScript 实现。**零运行时依赖，无需数据库，无需前端构建。** 当前页面为简体中文，同时提供中英文介绍文档。

## 功能

| 模块 | 功能说明 |
| --- | --- |
| OpenAI 兼容接口 | 模型列表、流式及非流式 Chat Completions |
| 号池管理 | 账户增删改、启停、搜索、JSON 批量导入 |
| 凭据刷新 | 按需刷新 Clerk JWT、合并同账户的并发刷新、保存轮换后的 Cookie |
| 账户调度 | 最低并发优先，同等并发轮询，异常账户自动冷却 |
| 运行看板 | 请求趋势、成功率、首字延迟和账户可用性 |
| 模型与日志 | 实时同步模型、统计实际延迟、查看请求详情、筛选和导出 CSV |
| 请求测试 | 发送真实流式请求、选择上游内置工具、停止本地请求 |
| 本地存储 | 重启后保留配置和请求元数据，管理接口不返回原始凭据 |

页面每 5 秒读取本地状态；自动刷新不会调用上游。账号检测和模型同步仅获取模型列表；请求测试会生成内容并消耗上游额度。

## 快速开始

需要 **Node.js 22 或更新版本**，以及可使用的 Logicc 账户与会话。

```bash
git clone https://github.com/JamesZhaoY/logicc2api.git
cd logicc2api
npm start
```

无需安装依赖，也可以直接执行 `node server.mjs`。

1. 打开 **<http://127.0.0.1:18789/>**。
2. 进入 **号池管理 → 添加账户**，填写凭据后点击 **检测**。
3. 在客户端中设置 Base URL 为 **`http://127.0.0.1:18789/v1`**。未配置 `LOGICC_API_KEY` 时，API Key 填任意非空字符串，例如 `local`。

自定义端口：

```bash
node server.mjs 18790
```

服务只监听 `127.0.0.1`，命令行端口优先于环境变量 `PORT`。

## 账户凭据

添加账户时支持以下任一种组合：

| 输入 | 行为 |
| --- | --- |
| `__client` Cookie + Session ID | 按需获取新的 Session JWT |
| `__client` Cookie + Session JWT | 从 JWT 的 `sid` 中读取 Session ID，再按需刷新 |
| 仅 Session JWT | 有效期内可用；到期后手动替换，或通过 `/__set_token` 推送新 JWT |

在已登录 Logicc 的浏览器中，打开开发者工具 → Application → Cookies → `clerk.logicc.com`，复制 `__client` 的 **Value**。Session ID 以 `sess_` 开头。也可在 Logicc 页面的 Console 获取 Session JWT：

```js
await window.Clerk.session.getToken({ skipCache: true })
```

`__session` Cookie 保存的是短期 Session JWT，与 `__client` 不是同一种凭据。会话被注销、撤销等情况仍需更新凭据，自动刷新不代表永久有效。

编辑时留空的凭据字段会保留已有值。更换客户端 Cookie 或 Session ID 会清除旧 JWT 缓存。启停和移除仅作用于本地号池，不会删除上游账户。

### 批量导入

在 **批量导入** 中粘贴 JSON 数组或上传 JSON 文件：

```json
[
  {
    "name": "工作账户",
    "clientCookie": "YOUR_CLIENT_COOKIE",
    "sessionId": "sess_YOUR_SESSION_ID",
    "enabled": true
  }
]
```

也支持 `{ "accounts": [...] }`。仅使用 JWT 时，每项填写 `name` 和 `token`。号池最多 200 个账户，相同 Session ID 不重复导入；整批校验通过后保存，校验失败不会部分写入。

## API 调用

### 获取模型列表

```bash
curl http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer local'
```

模型列表从选中的上游账户获取，不在代码中固定。

### 非流式响应

```bash
curl http://127.0.0.1:18789/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local' \
  -d '{
    "model": "default",
    "messages": [{"role": "user", "content": "用一句话打个招呼。"}]
  }'
```

### 流式响应

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local' \
  -d '{
    "model": "default",
    "stream": true,
    "stream_options": {"include_usage": true},
    "messages": [{"role": "user", "content": "用一句话打个招呼。"}]
  }'
```

`model` 省略或为 `default` 时，选择上游默认模型，没有默认项则使用第一个模型。流式返回 `chat.completion.chunk`，以 `[DONE]` 结束；省略 `stream` 或设为 `false` 则返回完整 JSON。

上游提供 token 用量时会转换返回；未提供时，API 兼容响应的计数填零，控制台明确显示用量不可用，不估算用量。

### 工具映射与兼容范围

- `reasoning_effort` 透传给上游。
- 函数名 `web_search` / `web-search` 映射到上游内置工具 `web-search`；`image_generation` / `image-generation` 映射到 `image-generation`。实际可用性取决于账户和模型。
- 其他函数定义以 JSON Schema 约定加入提示词，**尚未实现标准 OpenAI `tool_calls` 执行协议**。上游工具事件以 `[tool:web-search]` 等文本标记呈现。
- 每次请求新建上游聊天，多轮消息合并为单条提示词，不保留上游原生多轮会话状态。
- 附件未透传；图片工具输出在测试页面中以文本形式展示。
- 实现了 `/v1/models` 和 `/v1/chat/completions`，并非完整 OpenAI API。

## 调度与刷新行为

- 从已启用、符合调度条件的账户中，优先选择正在处理请求数最少的账户；同等并发采用轮询。同一次聊天固定使用一个账户。
- HTTP 429 冷却 60 秒，401 / 403 冷却 5 分钟，上游服务错误冷却 15 秒。后续请求跳过冷却账户。
- 冷却结束后，符合条件的账户可参与下一次请求；也可主动检测或更新账户。失败的单次请求不会跨账户重放。
- JWT 剩余寿命少于 10 秒时按需刷新；上游返回 401 时强制刷新并重试一次。
- 同一账户的并发刷新会合并；Clerk 返回新 `__client` Cookie 时自动保存。
- Clerk 刷新端点为 `POST /v1/client/sessions/{sid}/tokens`。

## 配置与数据

| 环境变量 | 用途 |
| --- | --- |
| `PORT` | 监听端口，默认 `18789` |
| `LOGICC_DATA_FILE` | 数据文件路径，默认项目目录下 `data/state.json` |
| `LOGICC_API_KEY` | 可选，配置后 `/v1/*` 必须携带对应 Bearer Key |
| `LOGICC_API_BASE` | 上游基址，默认 `https://app.logicc.com/api` |
| `LOGICC_CLERK_BASE` | Clerk 基址，默认 `https://clerk.logicc.com` |
| `LOGICC_CLIENT_COOKIE` | 号池为空时导入默认账户 |
| `LOGICC_SESSION_ID` | 环境变量导入账户的 Session ID，也可从 JWT 中读取 |
| `LOGICC_TOKEN` | 环境变量导入账户的初始 JWT |
| `LOGICC_TOKEN_FILE` | 环境变量导入账户使用的外部 JWT 文件 |
| `LOGICC_TOKEN_CMD` | 该账户的外部 JWT 刷新命令，仅从进程环境读取，页面不可配置 |

程序不自动加载 `.env` 文件。请在 shell 中导出环境变量，或在支持的 Node.js 版本中使用 `node --env-file=.env server.mjs` 启动。

已有账户数据优先于初始环境凭据，避免旧环境变量覆盖已轮换的 Cookie。使用外部刷新命令或 JWT 文件的账户，重启时仍需提供相应配置。

`data/state.json` 保存账户凭据、缓存模型目录、冷却时间及最多 2,000 条请求元数据。采用临时文件写入后重命名，文件请求权限为 `0600`；其中凭据为 **明文**，权限保护效果取决于操作系统。运行数据、环境文件、日志、备份和浏览器测试产物均排除在 Git 之外。

看板统计保留记录中最近 24 小时的聊天请求。模型同步与账号检测显示在日志中，不计入聊天指标。日志页展示最近 200 条记录。首字延迟包含凭据刷新、建立聊天和网络传输耗时；请求日志不保存提示词与生成的回答。

控制台面向 **单进程本机使用**，没有单独的管理员登录系统。管理请求需要通过本机来源/Host 检查并带有 `X-Logicc-Admin: 1`；`LOGICC_API_KEY` 仅保护 `/v1/*`。兼容入口 `POST /__set_token` 接收原始 JWT，可附加 `?account_id=<id>`，允许 Logicc 页面来源访问，并按 JWT 的 Session ID 匹配账户。

不要让多个实例同时写入同一数据文件。独立实例请分别配置端口和 `LOGICC_DATA_FILE`。

## 开发与验证

```bash
npm test
```

使用 Node 内置测试框架和本机模拟上游，覆盖账户增删改与整批导入、凭据脱敏和文件权限、轮询与冷却、强制刷新与 Cookie 轮换、并发刷新合并、JSON/SSE 转换、工具透传、用量与错误统计、来源限制和异常输入。测试无需真实凭据，不消耗上游额度。

```text
server.mjs              HTTP 路由、上游调用和协议转换
pool.mjs                账户存储、凭据刷新、调度和统计
public/                 无需构建的 Web 控制台
test/adapter.test.mjs    集成与协议测试
```

## 许可证

[MIT](LICENSE) © 2026 JamesZhaoY。
