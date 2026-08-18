# pi agent 多模型接入实战：官方 API 与公司网关双 provider 配置

> 场景：同一个模型（DeepSeek V4），既有官方直连 API，又有公司搭建的内部网关，如何在 pi 中同时接入并区分使用。

## 背景

我在用 pi（终端 AI 编程助手）时遇到一个问题：DeepSeek V4 有两个访问渠道——

1. **官方 API**：`https://api.deepseek.com/v1`，用自己的 key 直连
2. **公司网关**：公司内部搭建的 OpenAI 兼容网关（OneAPI/NewAPI 类），用公司分配的 key

如果只是往同一个 provider 里塞两个模型，`/model` 选择器里会显示两条一模一样的 `deepseek/deepseek-v4-pro`，根本分不清哪条走官方、哪条走公司网关。

## 核心思路：模型标识是 `provider/model-id`

pi 里每个模型的全量标识是 **`provider/model-id`**。所以区分两个来源的正确做法是：

> **建两个 provider，各自指向不同的 baseUrl**，而不是往同一个 provider 里塞两个模型。

```
deepseek/deepseek-v4-pro     ← 官方直连
deepseek-aku/deepseek-v4-flash  ← 公司网关
```

这样 `/model` 选择器按 provider 分组显示，一眼就能分辨。

## 配置文件

pi 的自定义模型配置在两个文件：

| 文件 | 作用 |
|------|------|
| `~/.pi/agent/models.json` | 定义 providers 和 models（模型、baseUrl、thinking 配置等） |
| `~/.pi/agent/auth.json` | 集中管理 API key，按 provider 名匹配 |

## 双 provider 配置示例

### `~/.pi/agent/models.json`

```json
{
  "providers": {
    "deepseek": {
      "baseUrl": "https://api.deepseek.com/v1",
      "modelOverrides": {
        "deepseek-v4-pro": {
          "reasoning": true,
          "thinkingLevelMap": {
            "minimal": "low",
            "low": "low",
            "medium": "medium",
            "high": "high",
            "xhigh": "max",
            "max": "max"
          },
          "compat": {
            "thinkingFormat": "deepseek"
          }
        }
      }
    },
    "deepseek-aku": {
      "baseUrl": "https://deepseek.akusre.com/v1",
      "api": "openai-completions",
      "models": [
        {
          "id": "deepseek-v4-flash",
          "name": "DS V4 Flash (公司网关)",
          "reasoning": true,
          "input": ["text"],
          "thinkingLevelMap": {
            "minimal": "low",
            "low": "low",
            "medium": "medium",
            "high": "high",
            "xhigh": "max",
            "max": "max"
          },
          "compat": {
            "thinkingFormat": "deepseek"
          }
        }
      ]
    }
  }
}
```

### `~/.pi/agent/auth.json`

```json
{
  "deepseek": {
    "type": "api_key",
    "key": "sk-官方key"
  },
  "deepseek-aku": {
    "type": "api_key",
    "key": "sk-公司网关key"
  }
}
```

key 按 provider 名自动匹配，互不干扰。

## 踩过的 4 个坑

### 1. `modelOverrides` 只对内置 provider 生效

官方 `deepseek` 是 pi 内置 provider，用 `modelOverrides` 覆盖模型配置即可。但**公司网关是新 provider，`modelOverrides` 对它不生效**，必须内联 `models` 数组，并且手动带上 `thinkingFormat: "deepseek"` 等 thinking 配置。

### 2. `api` 字段是协议类型，不是 URL 路径

一开始我把公司网关配成了：

```json
"api": "/v1/chat/completions"   // ❌ 错
```

`api` 字段表示**协议类型**，合法值是 `openai-completions` 等（OpenAI 兼容接口就写这个）。填路径会导致 provider 无法加载。

### 3. `baseUrl` 要带 `/v1` 路径前缀

公司网关地址如果是 `https://deepseek.akusre.com`，`baseUrl` 要写成 `https://deepseek.akusre.com/v1`，否则请求拼出来是 `.../chat/completions`（少了 `/v1`）直接 404。

### 4. 模型 id 用网关暴露的名字，不要猜

公司网关是 OneAPI/NewAPI 类，模型名由网关管理员配置（可能和官方名字不同）。**不要猜**，用网关 `/v1/models` 返回的 `id`。不过很多网关不暴露 `/v1/models` 端点（返回 404），那就直接问管理员或在网关管理后台查。

## 网关探测技巧

填 key 之前，可以先裸探测网关格式是否 OpenAI 兼容：

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST https://deepseek.akusre.com/v1/chat/completions \
  -H "Authorization: Bearer test" \
  -H "Content-Type: application/json" \
  -d '{"model":"x","messages":[]}'
```

| 返回码 | 含义 |
|--------|------|
| `401` | 端点存在且是 OpenAI 兼容格式，只是 key 无效 → baseUrl 配置正确 |
| `404` | 路径不对，或网关不暴露该端点 |
| `200` | 通了（不太可能，因为 key 是假的） |

我实测的结果：`POST /v1/chat/completions` → 401（端点存在），`/v1/models` → 404（不暴露模型列表）。

## thinkingLevelMap 详解

DeepSeek 是推理模型，思考级别映射是配置里的重头戏。

### pi 的 7 个思考级别

```
off | minimal | low | medium | high | xhigh | max
```

由弱到强，排序的量化证据是 settings.json 里的 `thinkingBudgets`（token 预算严格递增）：

| 级别 | budget |
|------|--------|
| minimal | 1024 |
| low | 4096 |
| medium | 10240 |
| high | 32768 |
| xhigh | 65536 |
| max | 131072 |

### 字段迁移：`reasoningEffortMap` → `thinkingLevelMap`

旧配置用 `compat.reasoningEffortMap`，新版已迁移到**模型顶层**的 `thinkingLevelMap`：

```json
{
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": "low",
    "low": "low",
    "medium": "medium",
    "high": "high",
    "xhigh": "max",
    "max": "max"
  },
  "compat": { "thinkingFormat": "deepseek" }
}
```

map 的 key 是 pi 的思考级别，value 是**发送给 provider 的字符串**。用 `null` 表示该级别不支持（UI 里隐藏、循环切换时跳过）。

### 省略语义（容易误解）

`thinkingLevelMap` 里**没写的 key** 意味着：

- `high` 及以下级别 → 走 provider 默认映射
- `xhigh` 和 `max` → **不支持**（隐藏、切换时跳过/clamp）

所以如果想让 `max` 级别可选，必须显式配置 `"max": "max"` 或保守的 `"max": "high"`。

### DeepSeek 官方 `reasoning_effort` 枚举

`thinkingFormat: "deepseek"` 会向 API 发送 `thinking: { type: "enabled" | "disabled" }` + `reasoning_effort`，pi 侧不做校验，map 里 value 是什么就原样发。DeepSeek 官方接口的枚举是：

```
low | medium | high
```

注意：**`max` 不在官方枚举里**。配置 `"xhigh": "max"` / `"max": "max"` 时，官方 API 严格校验会报参数错误（不校验则可能被忽略/降级）。公司网关通常是透传，接受值取决于网关配置——先按官方枚举配，报错再问网关管理员。我的实测是官方 API 6 个级别全部正常响应，`max` 也被接受。

## 验证

### 1. 检查模型加载

```bash
pi --list-models
```

输出确认两个 provider 都正常加载：

```
deepseek      deepseek-v4-flash  1M       384K     yes       no
deepseek      deepseek-v4-pro    1M       384K     yes       no
deepseek-aku  deepseek-v4-flash  128K     16.4K    yes       no
```

### 2. 全思考级别测试

```bash
pi -p "回复OK两个字" --model deepseek/deepseek-v4-pro --thinking off
pi -p "回复OK两个字" --model deepseek/deepseek-v4-pro --thinking low
# ... medium / high / xhigh / max
```

实测 6 个级别全部通过，`xhigh`/`max` 发送 `"max"` 到 DeepSeek API 均正常处理。

## 设置默认模型

临时指定用 `--model`，永久默认改 `~/.pi/agent/settings.json`：

```json
{
  "defaultProvider": "deepseek-aku",
  "defaultModel": "deepseek-v4-flash",
  "defaultThinkingLevel": "medium"
}
```

`defaultThinkingLevel` 取值：`off/minimal/low/medium/high/xhigh/max`。

## 日常切换

| 操作 | 方式 |
|------|------|
| 交互式切换 | `/model` 里按 provider 分组选择 |
| 快速循环 | `Ctrl+P`（`app.model.cycleForward`），配合 `--models` 或 settings 的 `enabledModels` 限定循环范围 |
| 启动指定 | `pi --model deepseek-aku/deepseek-v4-flash` |
| 临时切思考级别 | `Shift+Tab`（`app.thinking.cycle`） |

## 总结

```
官方 API 和公司网关同时接入 = 两个 provider + 各自 baseUrl/key
模型标识 provider/model-id 天然区分，/model 分组展示
modelOverrides 只改内置 provider；新 provider 必须内联 models
api 是协议类型（openai-completions），baseUrl 带 /v1
thinkingLevelMap 控制思考级别映射，省略 = 不支持 xhigh/max
DeepSeek 官方 reasoning_effort 枚举只有 low/medium/high
```

配置一次，之后 `/model` 里点选即可，两个渠道互不干扰。
