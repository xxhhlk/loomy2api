# loomy2api

讯飞 Loomy（`loomyad.xunfei.cn`）桌面客户端 API 的**接口结构参考**，供构建兼容客户端 / 网关使用。

> 本仓库只收录**接口结构与端点清单**：端点、鉴权形状、请求/响应结构。
> 不含签名密钥、凭据值、账号标识，也不含任何可复现的攻击面细节。

## 端点

| 用途 | 值 |
| --- | --- |
| 积分 / iModel 网关 | `https://loomyad.xunfei.cn` |
| 推理 baseURL | `https://loomyad.xunfei.cn/api/v1` |
| 账号服务 | `https://account.xfinfr.com` |
| 交易服务 | `https://trade.xfinfr.com` |
| Athena 会话 | `https://api-athena.xfinfr.com` |
| Nexus 长连 | `wss://dispatch-nexus.xfinfr.com/ws/loomy` |
| 官网 / 微信回调 | `https://loomy.xunfei.cn` |

## 推理请求

```
POST {baseURL}/chat/completions

Accept: application/json
Content-Type: application/json
Authorization: Bearer <session>
token: <session>                       # session 模式双写
traceparent: 00-<32hex>-<16hex>-01     # 必需，缺失会挂死到超时
loomy-version: <app version>
# 会话关联头（有则带）：ChatId / MsgId / TurnId

body: { model, messages, temperature, max_tokens, stream,
        enable_thinking, chat_template_kwargs: { enable_thinking } }
```

- 模型列表：`GET {baseURL}/models`（OpenAI 兼容）
- 模型标识形态：`provider/model`，例：`imodel/spark-x`

### 响应形态

- 非流式：`choices[].message.reasoning_content`（思考）+ `content`；`usage` 含
  `completion_tokens_details.reasoning_tokens`（思考 token）、`points_consumed`（消耗积分）、
  `prompt_tokens_details.cached_tokens`（命中缓存）
- 流式：标准 OpenAI SSE（`data: {...}` 分片，末尾 `data: [DONE]`）

### 已知模型

| ID | 用途 |
| --- | --- |
| `imodel/spark-x` | 主对话模型（讯飞星火 X） |
| `imodel/doubao-seed-2.0-mini` | 轻量（知识库媒体摘要硬锁定） |
| `imodel-anthropic` | Anthropic companion provider（走 `{baseURL}/messages`，与 OpenAI 不兼容） |

## 账号 API（base = `account.xfinfr.com`）

| 端点 | 用途 |
| --- | --- |
| `POST /login/phone/sendMsgCode` | 发送短信验证码 |
| `POST /login/phone/checkCode` | 短信登录 → session |
| `POST /login/account/getPuKey` | 取 RSA 公钥（密码登录用） |
| `POST /login/account/byPwd` | 账号密码登录 |
| `POST /userinfo/query/baseInfo` | 用户信息 |
| `POST /login/account/logout` | 登出 |

响应信封：`{"code":"000000", ...}`（成功码 `000000`）；登录态失效码 `020002` / `100002`。

## session

会话文件（导入型渠道从这里读取）：

| 平台 | 路径 |
| --- | --- |
| Windows | `C:\Users\Public\Loomy\<sha256(用户名)[:12]>\userData\auth-session.json` |
| Windows 回落 | `%APPDATA%\Loomy\auth-session.json` |

- 目录名 = `sha256(用户名)` 前 12 位 hex
- 客户端存储字段：`session` / `userid` / `phone` / `updatedAt`
- 有效期 14 天（由客户端在登录时指定）；**无续期机制**，到期需重新登录。

## 积分

base = `https://loomyad.xunfei.cn`，鉴权同推理。

| 端点 | 用途 |
| --- | --- |
| `GET /api/v1/points/records` | 积分记录 v1 |
| `GET /api/v2/points/records` | 积分记录 v2（按 chatId 聚合） |
| `GET /api/v1/team-points/balance` | 团队积分余额（`currentBalance`） |
| `GET /api/v1/pet-work` + `POST /api/v1/pet-work/rewards` | 每日任务快照 + 领奖 |
| `POST /api/v1/points/first-login` | 首次登录奖励 |

**余额口径**：v1 面 `points/records` 的 `data` 同时下发 `balance`（常规池，长期有效）、
`dailyBalance`（每日池，扣分优先消耗该池）、`availableBalance`（前两者之和，真正可消耗总额）。
只读 `balance` 会漏掉每日积分。

## 行为观察

- **鉴权失败是 `HTTP 200` + 业务码**（`100002` / `020002`），不是 401 —— 业务码判定必须排在状态码判定之前。
- **思考档位**：上游 `/models` 为每个模型声明 `reasoning_efforts`（实测 8 个模型一致，
  `none` / `low` / `medium` / `high` / `xhigh`，默认 `low`）。

## 最小调用示例

仅示意协议形状（导入 → 调用 → 解析 SSE），非可部署实现。导入型不登录，故无需签名、不含任何密钥。

```python
# 读会话文件里的 session → POST /chat/completions → 解析 SSE 分片。
import json, secrets, urllib.request

def traceparent():
    # 必需，缺失会挂死到超时：00-<16字节hex>-<8字节hex>-01
    return f"00-{secrets.token_hex(16)}-{secrets.token_hex(8)}-01"

def chat(base_url, session, model, prompt):
    body = json.dumps({"model": model, "stream": True,
                       "messages": [{"role": "user", "content": prompt}]}).encode()
    req = urllib.request.Request(base_url + "/chat/completions", data=body, method="POST")
    req.add_header("Authorization", f"Bearer {session}")
    req.add_header("token", session)          # session 模式双写
    req.add_header("traceparent", traceparent())
    req.add_header("loomy-version", "0.9.38")
    req.add_header("Accept", "application/json")
    req.add_header("Content-Type", "application/json")
    with urllib.request.urlopen(req) as resp:
        for raw in resp:                      # 流式：逐行读 data: 分片
            line = raw.decode("utf-8", "replace").strip()
            if not line.startswith("data:"):
                continue
            payload = line[5:].strip()
            if payload == "[DONE]":
                break
            delta = json.loads(payload)["choices"][0].get("delta", {})
            print(delta.get("content", ""), end="", flush=True)
```

## 范围与免责

- 仅用于互操作与研究。请遵守目标服务的使用条款。
- 本仓库不提供、也不描述绕过鉴权或校验的方法。

## License

MIT
