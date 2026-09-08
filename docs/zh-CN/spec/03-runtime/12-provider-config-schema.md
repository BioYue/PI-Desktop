# 12. 提供商配置架构

> **翻译说明：** 本页是与 [英文源规格](/spec/03-runtime/12-provider-config-schema) 一一对应的机器辅助翻译。代码、协议字段和标识符保持原文；如翻译与英文源事实有歧义，以英文版本为准。


## 1. 存储位置

由 Rust 主机 DB/settings 存储拥有。

表（[04-data-storage](/zh-CN/spec/03-runtime/04-data-storage) §4.3–4.4、§4.11 中的规范 DDL）：

- `providers`
- `models`（单个目录表；`source: bundled | discovered | user` 替换旧的 `provider_models` / `model_catalog_cache` 拆分）
- `secrets_meta`（无原始秘密值）
- 最近模型的 MRU 位于 `kv(ns='cache')` 中，而不是表中

## 2. 提供商记录 JSON 架构（逻辑）

```json
{
  "$id": "pi-desktop.provider.v1",
  "type": "object",
  "required": ["id", "name", "vendorKey", "type", "protocol", "enabled", "authKind"],
  "properties": {
    "id": { "type": "string", "minLength": 1 },
    "name": { "type": "string", "minLength": 1 },
    "vendorKey": { "type": "string", "minLength": 1 },
    "type": { "enum": ["native", "openai_compatible", "custom"] },
    "protocol": {
      "enum": ["openai", "anthropic", "google", "openai_compatible", "bedrock", "custom_http"]
    },
    "enabled": { "type": "boolean" },
    "baseUrl": { "type": "string" },
    "authKind": {
      "enum": [
        "api_key",
        "api_key_and_base_url",
        "bearer",
        "azure_api_key",
        "aws_sdk_default",
        "custom_headers",
        "oauth",
        "none"
      ]
    },
    "secretRef": { "type": "string" },
    "headers": {
      "type": "object",
      "additionalProperties": { "type": "string" },
      "maxProperties": 32
    },
    "userAgent": {
      "type": "string",
      "maxLength": 256,
      "description": "legacy; migrates into headers.User-Agent"
    },
    "apiStyle": {
      "enum": [
        "chat_completions",
        "opencode_go",
        "responses",
        "anthropic_messages",
        "google_generative_ai",
        "openai_codex_responses",
        "pi_messages",
        "auto"
      ]
    },
    "compatibility": {
      "type": "object",
      "properties": {
        "supportsTools": { "type": "boolean" },
        "supportsVision": { "type": "boolean" },
        "supportsStreaming": { "type": "boolean" },
        "supportsReasoning": { "type": "boolean" },
        "supportedThinkingLevels": {
          "type": "array",
          "items": {
            "enum": ["off", "minimal", "low", "medium", "high", "xhigh", "max"]
          },
          "uniqueItems": true
        }
      }
    },
    "defaultModelId": { "type": "string" },
    "models": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "contextWindow", "maxTokens", "thinkingLevels", "defaultThinkingLevel"],
        "properties": {
          "id": { "type": "string", "minLength": 1 },
          "contextWindow": { "type": "integer", "minimum": 1 },
          "maxTokens": { "type": "integer", "minimum": 1 },
          "thinkingLevels": {
            "type": "array",
            "items": { "enum": ["off", "minimal", "low", "medium", "high", "xhigh", "max"] },
            "uniqueItems": true
          },
          "defaultThinkingLevel": {
            "type": ["string", "null"],
            "enum": ["off", "minimal", "low", "medium", "high", "xhigh", "max", null]
          },
          "supportsImages": { "type": ["boolean", "null"] },
          "supportsDocuments": { "type": ["boolean", "null"] },
          "availableForSubagents": { "type": "boolean", "default": false }
        }
      }
    },
    "createdAt": { "type": "string" },
    "updatedAt": { "type": "string" }
  }
}
```

`compatibility.supportsReasoning` 与 `compatibility.supportedThinkingLevels`
为了已存储记录和旧客户端的兼容性而保持可读，但 Electron main 在运行时的模型
解析中会忽略它们。对外的提供商形态由本地的 models.dev 快照丰富。未知的自由
格式模型最初暴露 `supportsReasoning=false` 与 `supportedThinkingLevels=["off"]`；
对于确实支持的端点，设置里仍然可以持久化一个显式的思考级别绑定。原始密钥与
内部兼容性 JSON 保持隐藏。

Anthropic Messages 提供商既可以存服务根地址，也可以存以 `/v1` 结尾的 URL。
模型发现会保留所配置的路径，并且恰好请求一次 `/v1/models`；运行时在调用
pi-ai 之前会去掉结尾的 `/v1`，因为 pi-ai 的 Anthropic SDK 会自行把 `/v1` 加到
messages 路由上。因此两种写法最终都把消息发到所配置服务的 `/v1/messages`
端点，而不是重复的 `/v1/v1/messages` 路径。

对于 OpenAI 兼容的 Chat Completions 模型，运行时把
`compat.supportsDeveloperRole` 默认为 `false`，因此即便所选模型支持推理，
系统指令仍以 `role: "system"` 发送。对于确实接受 `role: "developer"` 的上游，
解析出的模型记录可以显式把它设为 `true`；这个覆盖的作用域限于该模型。

`authKind: "oauth"` 标记厂商账户行（ADR 0095、D237、D240）：其凭据是保存在
`secret:provider:<id>:oauth` 下的 OAuth 授权，而不是粘贴的密钥，因此该行
不为它保存 `secretRef`，并以空密钥启动。最后两个 apiStyle 是厂商账户专用的
线路 API —— `openai_codex_responses`（Codex 会话封装）与 `pi_messages`
（radius 网关）—— 自定义提供商对话框不提供它们，因为二者都无法配合手输的
base URL 与粘贴的密钥工作。厂商行的样式不由厂商固定：GitHub Copilot 同时
提供 Anthropic、Chat Completions 与 Responses 模型，因此样式跟随所选模型，
并在每次切换模型时重写。厂商账户编辑器使用与 AI 服务相同的多模型绑定控件：
可以选择已认证的目录模型与自定义 id，每条绑定都会把它的上下文窗口、最大输出
token、思考级别与默认思考级别持久化到 `models` 中。
`config_json.oauth.accountLabel` 保存已登录账户的非敏感展示标签。
每次成功登录都会创建一个新行，即使另一行有相同的 `vendorKey`；行 id 界定了
凭据与运行时绑定的作用域。厂商目录把这些行以 `accounts` 数组暴露出来，含
`providerId`、可选的非敏感 `accountLabel` 以及一个 `connected` 标志。自定义
提供商对话框不编辑也不删除 OAuth 行；由"厂商账户"卡片对选中的行调用
`providers.delete`。

`models` 是该提供商已选中的模型绑定数组。每条绑定拥有自己的上下文/输出上限
与显式的思考配置。已发布的目录级别会为新选中的已知模型播种，但绑定可以启用
任意规范级别，这样在目录更新之前，代理端点或新发布的端点也是可配置的。
`defaultModelId` 仍然是一个读取兼容字段，新建提供商保存时会让它等于第一条
绑定。当旧记录只有 `defaultModelId` 时，主机在读取时具体化出一条绑定：
128,000 上下文窗口、8,192 最大输出、不启用任何思考级别、默认值为 null。设置
编辑器对这条旧绑定仍然渲染全部规范选项，下一次写入就会把显式的绑定数组存进
`config_json.models`。

就上下文解析而言，那个 128,000 是一个向后兼容的通用种子，而不是隐藏已发布
长上下文上限的理由。如果 models.dev 现在发布了正数的 `limit.context`，有效的
运行时窗口与检查器窗口就跟随它；在模型的 Advanced 控件里填写的非默认值仍然
是一个显式的按模型覆盖。未知 id 继续使用 128,000。

`availableForSubagents` 是每条模型绑定上可选、可持久化的选择加入项。主机的
读写与归一化会保留 `true`；在该字段出现之前创建的记录默认保持禁用。Electron
main 正是用这个标志来构建委托模型目录的，因此对它的修改能在提供商编辑与应用
重启之后继续有效。

`apiStyle: "opencode_go"` 是叠加在 OpenAI 兼容提供商类型之上的一等 OpenCode Go
预设。它以自己的样式持久化，好让 UI 能识别该服务，但运行时请求使用 OpenAI
Chat Completions 线路适配器。「服务」下拉在命名的智谱端点旁边提供 OpenCode
Go。该预设始终使用 `name: "OpenCode Go"` 与
`baseUrl: "https://opencode.ai/zen/go/v1"`；常见路径是服务 + API key，主机
以摘要形式展示。模型发现用 Bearer key 调用固定的 `/models` 端点，原始 key
仍然走正常的密钥库路径。

OpenCode Go（以及任何 `opencode.ai` 主机）要求 LLM 请求带上稳定的
`x-opencode-session` 标头。Agent 运行时会在会话回合、子代理回合、提示增强
以及插件的一次性调用上，发送该标头，外加 `x-opencode-client: pi-desktop` 与
`User-Agent: pi-desktop/<APP_VERSION>`。调用方自带的标头会覆盖 client 与
User-Agent 的值；缺失或为空的会话标头总是由对话 id 补回。

`headers` 是存放在 `config_json.headers` 中的可选按行映射。留空、省略，或以
`{}` 更新，都保持适配器默认值（pi-ai 的 `pi (…)` 字符串、Anthropic OAuth 的
`claude-cli/<version>`，或 OpenCode 的 `pi-desktop/<APP_VERSION>`）。非空映射
是该行出站 HTTP 的最后写入方——会话回合、子代理、提示增强、插件一次性调用、
`/models` 发现（包括尚未保存的表单值）、连接测试，以及 OAuth token 刷新。
一层 fetch 包装是最后的写入方，因此 Codex 与 Anthropic SDK 无法覆盖它。同样
的值也会被放到流选项标头上，好让 OpenCode 的"调用方优先"规则依然成立。键在
不区分大小写的意义上唯一，最多 32 项，名称 ≤ 256 字节，值 ≤ 4096 字节，
不含 CR/LF，名称由字母数字加连字符组成。保留键（`authorization`、
`proxy-authorization`、`host`、`content-type`、`content-length`、`cookie`、
`set-cookie`、`connection`、`transfer-encoding`、`te`、`trailer`、`upgrade`、
`keep-alive`、`x-api-key`、`api-key`、`chatgpt-account-id`、
`x-opencode-session`）会被拒绝，以免这里冲掉签名或应用路由。它不是密钥。
遗留的 `config_json.userAgent` 在读取时迁移进 `headers["User-Agent"]`；一旦
写入 `headers` 就丢弃它。覆盖 Anthropic OAuth 的 `claude-cli/…` User-Agent
可能导致 Claude Pro/Max 拒绝请求。首次 OAuth 登录不收集标头；它们是在账户
存在之后才在该账户上编辑的。Advanced UI 是一个紧凑的键/值编辑器，而不是
一个专门的 User-Agent 字段。

## 3. 内置供应商预设

仅预设预填表单默认值；他们不是一个封闭的世界。

| 供应商密钥 | 默认协议 | 授权类型 | 需要基本网址 |
|---|---|---|---|
| 开放性 | 开放性 | api_key | 不 |
| 人择的 | 人择的 | api_key | 不 |
| 谷歌 | 谷歌 | api_key | 不 |
| 开放路由器 | openai_兼容 | api_key_and_base_url | 是的 |
| 深度搜索 | openai_兼容 | api_key_and_base_url | 是的 |
| 格罗克 | openai_兼容 | api_key_and_base_url | 是的 |
| 在一起 | openai_兼容 | api_key_and_base_url | 是的 |
| 烟花 | openai_兼容 | api_key_and_base_url | 是的 |
| 米斯塔拉尔 | openai_兼容或本机 | api_key | 可选的 |
| 赛 | openai_兼容 | api_key_and_base_url | 是的 |
| azure_openai | openai_兼容 | azure_api_key | 是的 |
| 基岩 | 基岩 | aws_sdk_默认 | 不 |
| 奥拉马 | openai_兼容 | 无 | 是的 |
| 工作室 | openai_兼容 | 无 | 是的 |
| 定制 | openai_兼容 | api_key_and_base_url | 是的 |

### 固定 API 风格预设

| apiStyle | 提供商类型 | authKind | 名称 | baseUrl |
|---|---|---|---|---|
| `opencode_go` | `openai_compatible` | `api_key_and_base_url` | `OpenCode Go` | `https://opencode.ai/zen/go/v1` |

OpenCode Go（以及任何 `opencode.ai` 主机）的 LLM 请求必须带稳定的
`x-opencode-session`。agent-runtime 在会话、子代理、提示增强与插件 one-shot
上发送该头，并附带 `x-opencode-client: pi-desktop` 与
`User-Agent: pi-desktop/<APP_VERSION>`。行上可选的 `headers` 会覆盖这些默认值；留空则保持适配器默认。

每行（AI 服务或 OAuth 账户）可在高级选项中用键值行编辑自定义请求头。空映射保持 pi-ai / `claude-cli` / OpenCode 默认。fetch 包装器是最后写入者，因此 Codex 与 Anthropic SDK 无法覆盖。禁止 `Authorization` / `Host` / `Content-Type` 等保留头。遗留的 `userAgent` 读取时迁入 `headers["User-Agent"]`。首次 OAuth 登录不收集请求头，登录后再编辑。覆盖 Anthropic OAuth 的 `claude-cli/…` 可能导致 Claude Pro/Max 拒绝请求。

### 命名端点预设

这些行由添加提供商对话框的**服务**下拉框创建。命名服务的常见路径是服务 +
API 密钥；自定义端点在常见路径上并排显示 API 密钥与接口格式。`vendorKey`
使用 models.dev 提供商键。

国际：OpenAI、Anthropic、Google Gemini、OpenRouter、Groq、xAI、Mistral、
Together、Fireworks、OpenCode Go、Z.AI。

国内：DeepSeek、通义千问、月之暗面、智谱 / Coding Plan、硅基流动、火山方舟、
MiniMax、Kimi 编程。

智谱 / Z.AI 的 Completions 请求仍使用 `thinkingFormat: "zai"` 与
`zaiToolStream: true`。

### 厂商账户预设

这些行由登录创建（设置 → 模型配置 → 厂商账户），而不是由自定义提供商
对话框创建。列表在运行时由 `models.getProviders().filter(p => p.auth.oauth)`
派生，因此它跟随 pi-ai 而不是本表；`baseUrl`、`apiStyle` 与 `defaultModelId`
在登录后由账户自己的目录填入。

| vendorKey | 订阅 | 典型 apiStyle | 登录形态 |
|---|---|---|---|
| anthropic | Claude Pro/Max | anthropic_messages | PKCE + 本地回调 |
| openai-codex | ChatGPT Plus/Pro | openai_codex_responses | PKCE + 本地回调，或手动贴码 |
| github-copilot | Copilot | 随模型而变 | 设备码 |
| openrouter | 账户余额 | chat_completions | PKCE + 本地回调 |
| kimi-coding | Kimi | chat_completions（仅 headers 认证） | 设备码 |
| xai | xAI | chat_completions | 设备码 |
| radius | Radius | pi_messages | PKCE + 本地回调 |

## 4. 模型目录缓存记录

```ts
type ModelCatalogCacheRecord = {
  providerId?: string // empty for global bundled
  modelId: string
  displayName: string
  vendorKey: string
  capabilities: string[]
  contextWindow?: number
  source: "bundled" | "discovered" | "user"
  /** 渲染器标注：该行解析自随包的 models.dev 快照。 */
  catalogSource?: "models.dev"
  updatedAt: string
  raw?: unknown
}
```

## 5. 随包的 models.dev 快照

原始的公开目录被签入发布资源 `apps/desktop/resources/models.dev/api.json`，
并打包到 `resources/models.dev/api.json`。`scripts/release.mjs` 抓取并校验
`https://models.dev/api.json`，然后在创建发布标签之前原子地替换那份签入的
文件。Electron main 在启动时读取这份随包快照，不产生任何网络 I/O。设置 →
模型配置可以重新抓取该 URL，但成功的响应只更新当前进程的内存内目录；它绝不
写入打包资源或用户缓存。缓存读取绝不会把提供商凭据发送给 models.dev。Rust 的
`models` 表继续只存储归一化的提供商选择行；它不需要为这份原始快照做模式变更。

## 6. IPC / 主机方法（提供商域）

- `providers.list`
- `providers.get`
- `providers.create`
- `providers.update`
- `providers.delete`
- `providers.testConnection`
- `providers.listModels`
- `providers.cacheModels`（内部 Electron-main 到主机持久桥）
- `providers.refreshModels`
- `providers.upsertUserModel`
- `providers.deleteUserModel`

## 6. 安全限制

1. list/get 提供商 API 从未返回原始机密
2. 如果可以使用密钥存储，`headers` 不得存储 `Authorization: Bearer <secret>`
3.导出设置默认排除机密

## 7. 迁移

- 通过 `PRAGMA user_version` 的架构版本（04-数据存储§7）
- 提供商记录累加进化；每个提供商的扩展字段登陆 `config_json`
- 未知的未来协议值不应使旧的应用程序版本崩溃（ignore/disable，带有警告）

## 8. SQL（Rust 拥有的 SQLite）

规范的 DDL 位于 [04-data-storage](/zh-CN/spec/03-runtime/04-data-storage) (D086) 中。提供商域表摘要：

```sql
-- providers: id/name/vendor_key/type/protocol/api_style/auth_kind/base_url/
--            enabled/secret_ref/default_model_id + config_json (headers,
--            compatibility, future knobs), INTEGER ms timestamps
-- models:    PK(provider_id, model_id), display_name, source
--            (bundled|discovered|user), capabilities_json, context_window,
--            max_output_tokens, deprecated — refresh upserts never overwrite
--            source='user' rows
-- secrets_meta: secret_ref PK, owner_kind/owner_id, kind, backend
```

> 原始秘密材料**不**存储在这些表中。

## 9. 宿主方法合约 (v1)

### `providers.list`
- 在：`{ includeDisabled?: boolean }`
- 输出：`{ providers: ProviderPublic[] }`
- `ProviderPublic` 排除原始秘密；包括 `hasSecret: boolean`（**任一种**凭据
  存在即为真）、`hasOauth: boolean`、非敏感的 `oauthAccountLabel?: string`
  与可选的 `headers?: Record<string, string>`

### `providers.create` / `providers.update`
- 在：提供商字段 + 可选的 `secretValue` + 可选的 `oauthAccountLabel`
  （合并进 `config_json.oauth`，传空字符串即清除）+ 可选的 `headers`
  （合并进 `config_json.headers`，传 `{}` 即清除）；遗留 `userAgent` 读取时迁入 `headers["User-Agent"]`；旧客户端仍可能发送
  `supportsReasoning` / `supportedThinkingLevels`
- 行为：保留配置；如果存在secretValue，则写入密钥存储并设置
  `secretRef`；传统思维领域可能仍保留在
  `config_json.compatibility` 但不影响运行时分辨率
- 输出：`ProviderPublic`

### `providers.delete`
- 在：`{ id, deleteSecret?: boolean }` 默认 `deleteSecret=true`
- 行为：同时清除两个凭据引用（`:api_key` 与 `:oauth`）及其元数据记录，
  因此重新创建的提供商绝不会继承他人的刷新令牌
- 输出：`{ ok: true }`

### `providers.testConnection`
- 在：`{ id, modelId?: string }`
- 输出：`{ ok: boolean, latencyMs?: number, error?: AppError, sampleModelId?: string }`
- `authKind: "oauth"` 行通过解析厂商认证（必要时刷新令牌）来自证，而不是用
  它并不持有的密钥去访问网络

### `providers.listModels`
- 渲染器 IPC 位于：`{ providerId, source?: "cache"|"refresh" }`； `cache`
  返回没有提供商网络访问权限的持久目录，而 `refresh`
  在 Electron main 中运行发现
- 将 RPC 托管在：`{ providerId?: string }` 中；只读取 Rust 拥有的 `models`
  表
- 对 `authKind: "oauth"` 行，Electron 主进程读取已认证的目录
  （`models.getAvailable`，它已应用厂商自己的 `filterModels`，因此 Copilot
  账户列出的是其订阅包含的模型），而不是调用 `/models`；返回的每个模型都
  带着其线路 API 所隐含的 apiStyle。`openai-codex` 这类静态厂商使用已固定
  的 pi-ai 目录（0.85.1 包含 `gpt-6-astra`）；models.dev 不会发明这些 ID。
- 输出：`{ models: ModelCatalogItem[] }`；每个模型都带有 pi-resolved
  `reasoning` 功能和 `supportedThinkingLevels`。缓存的功能标签
  旧提供程序字段无法覆盖 pi 模型记录。

### `providers.cacheModels`（内部主机 RPC）
- 在：`{ providerId, models: DiscoveredModelInput[] }`
- 行为：以事务方式将成功的实时发现更新到 `models` 中
`source='discovered'`；永远不会覆盖 `source='user'` 行并且永远不会删除
  失败或部分刷新时先前的缓存行
- 输出：`{ cached: number, models: ModelCatalogItem[] }`
- 原始机密和授权标头绝不是此调用的一部分

### `providers.refreshModels`
- 在：`{ id }`
- 输出：`{ added: number, updated: number, removed: number, models: ModelCatalogItem[] }`

### `providers.upsertUserModel` / `providers.deleteUserModel`
- 管理自由格式/覆盖模型条目

## 10. 验证规则

1. `name` 在提供商中是唯一的（不区分大小写）
2. `openai_compatible` / 本地网关需要绝对 `baseUrl`，除非预设表示可选
3. `authKind=none` 禁止用于需要密钥的云预设
4. headers key 不区分大小写，唯一
5. headers key 不区分大小写且唯一，最多 32 条；禁止保留头与 CR/LF
6. 强制实施 SecretValue 最大长度（例如 8KB）
6. modelId 必须是非空的修剪字符串；允许 `/`、`.`、`:`、`-`
7.旧客户端上的未知协议 => 提供程序显示为禁用并带有警告，而不是崩溃
8. 旧版 `supportsReasoning`（如果存在）仍必须验证为布尔值，但
   没有运行时效果
9. 旧版 `supportedThinkingLevels`（如果存在）仍必须验证为
   一系列规范思维水平，但没有运行时效果

## 11. 秘密引用格式

```text
secret:provider:<providerId>:api_key
secret:provider:<providerId>:oauth
```

两个引用相互独立，因此一行可以只有密钥、只有厂商账户，或两者兼有；参见
[14-secrets-storage](14-secrets-storage.md) §10。未来的多重秘密提供商可能会
继续添加后缀（`:client_secret` 等）。
