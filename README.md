# V2Board wyx2685 分支 API 文档

> 本文档基于仓库20251024 `wyx2685/v2board`梳理，覆盖 `/api/v1` 与 `/api/v2` 暴露的接口及数据结构。

## 基础说明
- 默认 API 前缀：`https://<your-domain>/api/v1`，未特殊标注的接口均在该前缀下。
- 需要登录的接口必须在请求头加入 `Authorization: <auth_data>`。`auth_data` 来自登录/注册返回的数据，也可以放在请求体字段 `auth_data` 中。
- 订阅相关接口（`/client/...`）通过查询参数 `token=<用户订阅 token>` 鉴权。该 `token` 可由登录接口返回的 `token` 字段获取或调用 `/user/resetSecurity` 重置。
- 服务端回调（`/server/...` 与 `/api/v2/server/...`）均需提供 `token=<配置中的 server_token>`。
- 余额、金额等财务字段（如 `balance`、`total_amount`、`commission_balance`）单位为 **分**。
- 时间字段除特殊说明外均为 Unix 时间戳（秒）。
- 服务端通用响应格式：
  ```json
  {
    "data": ...,
    "message": "...", // 仅错误时出现
    "errors": ...     // 仅表单验证失败时出现
  }
  ```
  校验失败通常返回 HTTP `422`，权限问题返回 `403`，业务异常直接返回 `500` 携带 message。

## 1. 公共 / Passport 接口（无需 Authorization）

### 1.1 接口索引
| 方法 | 路径 | 描述 |
|------|------|------|
| POST | /passport/auth/register | 用户注册 |
| POST | /passport/auth/login | 登录获取 auth_data/token |
| POST | /passport/auth/loginWithMailLink | 发送邮箱快捷登录链接（需开启配置） |
| POST | /passport/auth/getQuickLoginUrl | 使用现有会话生成一分钟有效的快捷登录链接 |
| GET  | /passport/auth/token2Login | 快捷登录 Token 换取会话或跳转 |
| POST | /passport/auth/forget | 重置密码 |
| POST | /passport/comm/sendEmailVerify | 发送邮箱验证码（注册/找回共用） |
| POST | /passport/comm/pv | 邀请链接 PV 统计 |
| GET  | /guest/comm/config | 公共配置（注册页） |

### 1.2 注册 `POST /passport/auth/register`
请求体（JSON）：
```json
{
  "email": "user@example.com",
  "password": "12345678",
  "invite_code": "ABCD1234",
  "email_code": "123456",
  "recaptcha_data": "..."
}
```
关键逻辑：
- 根据后台开关检查邮箱白名单、reCAPTCHA、Gmail 别名限制等。
- 首次注册可能自动发放试用套餐（`try_out_plan_id`）。

成功响应示例：
```json
{
  "data": {
    "token": "5f6f20f6-...",
    "is_admin": 0,
    "auth_data": "eyJ0eXAiOiJKV..."
  }
}
```
`token` 用于订阅链接，`auth_data` 写入 `Authorization` 头。

### 1.3 登录 `POST /passport/auth/login`
请求体：
```json
{"email":"user@example.com","password":"12345678"}
```
密码连续错误会触发次数限制（默认 5 次/60 分钟）。
成功响应同上（返回 `token`、`is_admin`、`auth_data`）。

### 1.4 邮箱快捷登录 `POST /passport/auth/loginWithMailLink`
- 需后台开启 `login_with_mail_link_enable`。
- 请求体：`{"email":"user@example.com","redirect":"dashboard"}`。
- 接口会限制 60 秒内重复发送，同一链接 5 分钟有效。
- 成功返回 `data` 为登录链接 URL。

### 1.5 快捷登录 URL `POST /passport/auth/getQuickLoginUrl`
- 需在请求头或体中携带 `auth_data`。
- 可选参数 `redirect`，默认 `dashboard`。
- 响应 `{"data":"https://<app>/#/login?verify=...&redirect=..."}`，链接 60 秒有效。

### 1.6 Token 换会话 `GET /passport/auth/token2Login`
- `token` 参数：返回 302 跳转到前端 `/#/login?verify=...`。
- `verify` 参数：返回 `{"data":{token,is_admin,auth_data}}` 并消费缓存中的临时 token。

### 1.7 忘记密码 `POST /passport/auth/forget`
请求体：
```json
{"email":"user@example.com","password":"NewPass123","email_code":"123456"}
```
成功返回 `{"data":true}`，会清理该用户所有会话。

### 1.8 发送邮箱验证码 `POST /passport/comm/sendEmailVerify`
请求体：
```json
{"email":"user@example.com","isforget":0,"recaptcha_data":"..."}
```
- `isforget`：`0` 表示注册阶段要求邮箱未注册，`1` 表示找回密码阶段要求邮箱已存在。
- 接口对相同 IP 做 60 秒限流，对同一邮箱 60 秒限速。

成功返回 `{"data":true}`。

### 1.9 邀请 PV 统计 `POST /passport/comm/pv`
请求体：`{"invite_code":"ABCD1234"}`，仅记录访问量。

### 1.10 公共配置 `GET /guest/comm/config`
响应示例：
```json
{
  "data": {
    "tos_url": "https://example.com/tos",
    "is_email_verify": 1,
    "is_invite_force": 0,
    "email_whitelist_suffix": ["gmail.com","example.com"],
    "is_recaptcha": 0,
    "recaptcha_site_key": null,
    "app_description": "...",
    "app_url": "https://example.com",
    "logo": "https://..."
  }
}
```

## 2. 用户接口（需 Authorization）

### 2.1 接口索引
| 类别 | 方法 | 路径 | 描述 |
|------|------|------|------|
| 会话 | GET | /user/checkLogin | 判断 `auth_data` 是否有效 |
| 会话 | GET | /user/getActiveSession | 获取当前用户活跃会话 |
| 会话 | POST | /user/removeActiveSession | 移除指定会话 |
| 资料 | GET | /user/info | 基础账号信息、余额 |
| 资料 | POST | /user/update | 设置自动续费/通知 |
| 安全 | POST | /user/changePassword | 修改登录密码 |
| 安全 | POST | /user/newPeriod | 强制提前清零开新周期（需后台允许） |
| 礼品卡 | POST | /user/redeemgiftcard | 兑换礼品卡 |
| 统计 | GET | /user/getStat | 待办统计（未支付订单/工单/邀请） |
| 快捷登录 | POST | /user/getQuickLoginUrl | 生成快捷登录 URL |
| 订阅 | GET | /user/getSubscribe | 套餐状态/订阅链接 |
| 订阅 | GET | /user/resetSecurity | 重置 UUID/订阅 token |
| 订阅 | GET | /user/unbindTelegram | 解绑 Telegram |
| 佣金 | POST | /user/transfer | 佣金转余额 |
| 订单 | GET | /user/order/fetch | 订单列表 |
| 订单 | GET | /user/order/detail | 订单详情 |
| 订单 | GET | /user/order/check | 查询订单状态 |
| 订单 | GET | /user/order/getPaymentMethod | 可用支付方式 |
| 订单 | POST | /user/order/save | 创建订阅/充值订单 |
| 订单 | POST | /user/order/checkout | 发起支付/结算 |
| 订单 | POST | /user/order/cancel | 取消待支付订单 |
| 套餐 | GET | /user/plan/fetch | 套餐/商店列表 |
| 邀请 | GET | /user/invite/fetch | 邀请码列表 + 统计 |
| 邀请 | GET | /user/invite/save | 生成邀请码 |
| 邀请 | GET | /user/invite/details | 佣金记录分页 |
| 公告 | GET | /user/notice/fetch | 公告列表或详情 |
| 工单 | GET | /user/ticket/fetch | 工单列表或详情 |
| 工单 | POST | /user/ticket/save | 新建工单 |
| 工单 | POST | /user/ticket/reply | 回复工单 |
| 工单 | POST | /user/ticket/close | 关闭工单 |
| 工单 | POST | /user/ticket/withdraw | 发起佣金提现工单 |
| 节点 | GET | /user/server/fetch | 节点列表（含 ETag 缓存） |
| 优惠券 | POST | /user/coupon/check | 校验优惠券 |
| 配置 | GET | /user/comm/config | 用户侧配置（提现、货币等） |
| 配置 | POST | /user/comm/getStripePublicKey | 获取指定支付方式的 Stripe 公钥 |
| Telegram | GET | /user/telegram/getBotInfo | 获取机器人用户名 |
| 知识库 | GET | /user/knowledge/fetch | 文档/教程列表或详情 |
| 知识库 | GET | /user/knowledge/getCategory | （路由存在，当前分支未实现） |
| 统计 | GET | /user/stat/getTrafficLog | 每日流量统计 |

### 2.2 会话与账号
#### GET /user/checkLogin
响应：
```json
{"data":{"is_login":true,"is_admin":true}}
```

#### GET /user/getActiveSession
返回以 `session_id` 为键的会话：
```json
{
  "data": {
    "f3d6...": {
      "ip": "1.2.3.4",
      "login_at": 1719830400,
      "ua": "Mozilla/5.0 ...",
      "auth_data": "eyJ0eXAiOiJKV..."
    }
  }
}
```

#### POST /user/removeActiveSession
请求体：`{"session_id":"f3d6..."}`，成功返回 `{"data":true}`。

#### POST /user/changePassword
请求体：
```json
{"old_password":"OldPass123","new_password":"NewPass123"}
```
成功 `{"data":true}`，会注销所有会话。

#### POST /user/update
用于设置自动续费/提醒：
```json
{
  "auto_renewal": 1,
  "remind_expire": 1,
  "remind_traffic": 0
}
```

#### POST /user/newPeriod
无额外参数，需后台启用 `allow_new_period` 且当前流量已用完；成功返回 `{"data":true}`。

#### POST /user/redeemgiftcard
请求体：`{"giftcard":"CODE-XXXX"}`。成功返回：
```json
{"data":true,"type":1,"value":5000}
```
`type` 含义：
1. 余额充值（`value` 单位分）
2. 延长有效期（天）
3. 增加流量（GB）
4. 清空流量
5. 直开套餐（`value` 为天数，0 表示不过期）

### 2.3 订阅相关
#### GET /user/info
返回字段：`email、transfer_enable、device_limit、last_login_at、created_at、banned、auto_renewal、remind_expire、remind_traffic、expired_at、balance、commission_balance、plan_id、discount、commission_rate、telegram_id、uuid、avatar_url` 等。

#### GET /user/getStat
返回数组 `[待支付订单数, 待处理工单数, 邀请注册用户数]`。

#### GET /user/getSubscribe
响应包含：
- `plan_id`、`plan`（完整套餐信息，可能为 null）
- `token`、`subscribe_url`
- `expired_at`、`transfer_enable`、`device_limit`、`u`、`d`
- `alive_ip`（当前在线设备数）
- `reset_day`（距离下次流量重置天数，可能为 null）
- `allow_new_period`（是否可用 `/user/newPeriod`）

示例：
```json
{
  "data": {
    "plan_id": 3,
    "token": "5f6f20f6-...",
    "expired_at": 1725100800,
    "transfer_enable": 9674588160,
    "device_limit": 5,
    "u": 123456,
    "d": 789012,
    "alive_ip": 2,
    "subscribe_url": "https://example.com/api/v1/client/subscribe?token=...",
    "reset_day": 12,
    "allow_new_period": 1,
    "plan": {
      "id": 3,
      "name": "Pro 年付",
      "month_price": null,
      "year_price": 19900
    }
  }
}
```

#### GET /user/resetSecurity
返回最新订阅链接 `{"data":"https://..."}`，同时刷新 `uuid` 与 `token`（原链接立即失效）。

#### GET /user/unbindTelegram
成功返回 `{"data":true}`。

#### POST /user/getQuickLoginUrl
与 Passport 版本相同，返回一分钟有效的前端登录链接。

### 2.4 佣金与余额
#### POST /user/transfer
请求体：`{"transfer_amount":1000}`（单位分）。将佣金余额扣除并转入账号余额。成功 `{"data":true}`。

### 2.5 订单与支付
#### GET /user/order/fetch
可选查询参数：
- `status`：订单状态（0 待支付 / 3 已完成 / 4 已折抵等）。

返回数组，订单包含 `trade_no、total_amount、balance_amount、status、plan` 等字段（`plan` 为关联套餐）。

#### GET /user/order/detail
参数 `trade_no`。充值订单会额外返回：
- `plan: {"id":0,"name":"deposit"}`
- `bounus`（注意拼写）与 `get_amount`。

#### GET /user/order/check
参数 `trade_no`，返回订单状态码。

#### GET /user/order/getPaymentMethod
返回启用中的支付方式列表：
```json
{
  "data": [
    {
      "id": 10,
      "name": "USDT-TRC20",
      "payment": "Epay",
      "icon": "https://...",
      "handling_fee_fixed": 0,
      "handling_fee_percent": 0
    }
  ]
}
```

#### POST /user/order/save
创建订单，根据 `plan_id` 分两种模式：

1. 余额充值（`plan_id`=0）：
   ```json
   {"plan_id":0,"period":"deposit","deposit_amount":5000}
   ```
   `deposit_amount` 必须 >0 且 <9,999,999（分），返回 `trade_no`。

2. 订阅/流量包：
   ```json
   {
     "plan_id":3,
     "period":"year_price",
     "coupon_code":"ABC123"
   }
   ```
   `period` 取值：`month_price, quarter_price, half_year_price, year_price, two_year_price, three_year_price, onetime_price, reset_price, deposit`。
   当用户余额覆盖全部金额时自动抵扣并返回 `total_amount=0`。

成功响应：`{"data":"202401010001"}`（trade_no）。

#### POST /user/order/checkout
请求体：
```json
{"trade_no":"202401010001","method":10,"token":null}
```
- 当订单金额 <=0：返回 `{"type":-1,"data":true}` 并直接开通。
- 其余情况 `type`/`data` 由支付插件决定（`type=0` 通常为二维码内容，`type=1` 为跳转 URL）。

#### POST /user/order/cancel
请求体：`{"trade_no":"202401010001"}`。仅待支付订单可取消。

### 2.6 套餐
#### GET /user/plan/fetch
- 无参数：返回上架套餐列表；`capacity_limit` 字段为剩余可售数量（若设置了上限）。
- 可传 `id` 获取单个套餐详情（需确认套餐对当前用户可见）。

### 2.7 邀请
#### GET /user/invite/fetch
返回：
```json
{
  "data": {
    "codes": [{"id":1,"code":"ABCD1234","status":0,"created_at":0}],
    "stat": [
      12,
      3560,
      1200,
      20,
      500
    ]
  }
}
```

#### GET /user/invite/save
单次生成一个邀请码，达到后台 `invite_gen_limit`（默认 5）后返回错误。成功 `{"data":true}`。

#### GET /user/invite/details
分页参数：
- `current`（默认 1）
- `page_size`（最小 10）

返回 `data` 列表与 `total`。

### 2.8 公告
#### GET /user/notice/fetch
- `id` 参数存在时返回单条公告。
- 否则支持 `current`、`pageSize`（默认 5，最大 100），返回 `{data: [...], total: 12}`。

### 2.9 工单
#### GET /user/ticket/fetch
- 无参数：返回当前用户所有工单（按创建时间倒序）。
- 带 `id` 时返回详情及 `message` 列表，每条消息附带 `is_me` 标记。

#### POST /user/ticket/save
请求体：
```json
{
  "subject":"无法连接节点",
  "level":1,
  "message":"具体描述..."
}
```
后台可配置建单限制（必须有支付订单/完全禁用等）。成功返回 `{"data":true}`。

#### POST /user/ticket/reply
`{"id":123,"message":"补充信息"}`。要求最新回复不是当前用户，否则返回错误。

#### POST /user/ticket/close
`{"id":123}`，成功关闭返回 `{"data":true}`。

#### POST /user/ticket/withdraw
请求体：
```json
{"withdraw_method":"alipay","withdraw_account":"user@alipay.com"}
```
会自动创建提现工单，受后台开关 `withdraw_close_enable` 与最小提现额度限制。

### 2.10 节点列表
#### GET /user/server/fetch
返回可用节点数组，字段包含：
- `id`, `type`（`shadowsocks`/`vmess`/`vless`/`trojan`/`tuic`/`hysteria`/`anytls`/`v2node` 等）
- `name`, `host`, `port`（或 `mport` 表示端口区间字符串）
- `network`, `tls`, `cipher`, `flow`, `server_key` 等（节点类型特有字段）
- `last_check_at`, `is_online`
- `cache_key`（用于生成 `ETag`）

支持 `If-None-Match` 缓存，未变化时返回 304。

示例（简化）：
```json
{
  "data": [
    {
      "id": 9,
      "type": "vless",
      "name": "HK | IEPL",
      "host": "hk01.example.com",
      "port": 443,
      "network": "grpc",
      "tls": 1,
      "flow": "xtls-rprx-vision",
      "server_key": "5f6f20f6...",
      "is_online": 1,
      "cache_key": "vless-9-1719830400-1"
    }
  ]
}
```

### 2.11 优惠券
#### POST /user/coupon/check
请求体：
```json
{"code":"ABC2024","plan_id":3}
```
返回优惠券详情（折扣、限制、过期时间等）。

### 2.12 用户配置
#### GET /user/comm/config
响应：
```json
{
  "data": {
    "is_telegram": 1,
    "telegram_discuss_link": "https://t.me/xxxx",
    "stripe_pk": null,
    "withdraw_methods": ["alipay","usdt"],
    "withdraw_close": 0,
    "currency": "CNY",
    "currency_symbol": "¥",
    "commission_distribution_enable": 0,
    "commission_distribution_l1": 10,
    "commission_distribution_l2": 0,
    "commission_distribution_l3": 0
  }
}
```

#### POST /user/comm/getStripePublicKey
请求体：`{"id":10}`（支付方式 ID），返回对应 Stripe 公钥字符串。

### 2.13 Telegram
#### GET /user/telegram/getBotInfo
返回 `{"data":{"username":"YourBot"}}`。

### 2.14 知识库
#### GET /user/knowledge/fetch
- 当携带 `id`：返回文章详情，`body` 中的订阅占位符已替换为当前用户的订阅链接/token。
- 未携带 `id` 时需提供 `language`（如 `zh-CN`），可选 `keyword`，返回按分类分组的文档列表。

> `/user/knowledge/getCategory` 在该分支路由存在但未实现，会返回 500。

### 2.15 流量统计
#### GET /user/stat/getTrafficLog
返回当前月每天的流量记录：
```json
{
  "data": [
    {"u":123456,"d":654321,"record_at":1724956800,"user_id":1,"server_rate":1.0}
  ]
}
```

## 3. 客户端接口（订阅 token）

这些接口挂载 `client` 中间件，必须在查询参数提供 `token=<订阅 Token>`。

### 3.1 GET /client/subscribe
- 根据 UA 或 `flag` 参数自动输出对应协议（Clash/V2Ray/Shadowrocket/Sing-box 等）格式的配置文件。
- 支持 `flag=clash`、`flag=shadowrocket`、`flag=singbox` 等。

### 3.2 GET /client/app/getConfig
- 返回 Clash 配置 YAML，内容基于 `resources/rules/app.clash.yaml` 或自定义模板合并节点。
- Header `Content-Type: text/yaml`。

### 3.3 GET /client/app/getVersion
- 默认返回各平台版本及下载链接。
- 当 UA 包含 `tidalab/4.0.0` 等，会根据平台返回单一版本字段。

## 4. Guest Webhook / 支付回调

### 4.1 POST /guest/telegram/webhook
- 需通过 `access_token=md5(telegram_bot_token)` 进行简单校验。
- 消息根据加载的插件自动分发。

### 4.2 GET/POST /guest/payment/notify/{method}/{uuid}
- 供支付网关回调，`method` 为支付驱动标识，`uuid` 为支付实例 UUID。
- 回调体透传给对应 `PaymentService` 校验后执行订单入账。
- 成功返回支付插件自定义响应或 `success`。

## 5. 服务端节点回调（Server API）

基础地址：`/api/v1/server/{Class}/{action}`，常见 `{Class}` 值：

| Class | 典型 action | 说明 |
|-------|-------------|------|
| Deepbwork | user / submit / config | VMess Aurora 节点 |
| ShadowsocksTidalab | user / submit | Shadowsocks 节点 |
| TrojanTidalab | user / submit / config | Trojan 节点 |
| UniProxy | user / push / alivelist / alive / config | 通用代理（Shadowsocks/VLESS/V2Node/Hysteria/TUIC/AnyTLS 等） |

通用规则：
- 所有请求必须携带 `token`（后端配置的 `server_token`），以及 `node_id`。
- `user`：拉取允许接入的用户列表。响应会携带 `ETag`，客户端可通过 `If-None-Match` 减少流量。
- `submit`/`push`：上报流量数据，数组元素应包含 `user_id`、`u`、`d` 等。
- `alive`：UniProxy 用于上报在线 IP；`alivelist` 返回需要限流的在线数。
- `config`：输出节点运行所需配置，部分控制器返回 JSON 字符串。

示例（UniProxy `user`，JSON 格式）：
```json
{
  "users": [
    {"id":1,"speed_limit":0,"device_limit":3,"uuid":"12ab..."}
  ]
}
```
如请求头 `X-Response-Format: msgpack`，会返回 MessagePack 格式。

## 6. V2 Node API

### GET /api/v2/server/config
- 参数：`token`、`node_id`。
- 返回 V2board V2 节点配置（listen_ip、server_port、network 等），并附带 `base_config`（推送间隔、统计阈值）。
- 如节点类型为 2022 加密，会额外返回 `server_key`。
- 响应带 `ETag` 支持 304。

示例：
```json
{
  "listen_ip": "0.0.0.0",
  "server_port": 30001,
  "network": "tcp",
  "protocol": "reality",
  "tls": 1,
  "flow": "xtls-rprx-vision",
  "server_key": "5f6f20...",
  "base_config": {
    "push_interval": 60,
    "pull_interval": 60,
    "node_report_min_traffic": 0,
    "device_online_min_traffic": 0
  }
}
```

## 7. 其他说明
- #### 金额单位
  所有金额字段（`balance`、`total_amount`、`commission_balance` 等）均为整数，单位分。前端显示时需除以 100。
- #### 订阅 token
  初次登录或注册返回的 `token` 字段即订阅 token；`/user/resetSecurity` 会重置 `uuid` 与 `token`，原订阅链接立即失效；`/user/getQuickLoginUrl`、`/passport/auth/getQuickLoginUrl` 生成的链接有效期 60 秒。
- #### 缓存控制
  多个接口（`/user/server/fetch`、服务端 `user/config` 等）使用 `ETag`，建议配合 `If-None-Match` 使用。
- #### 注销
  当前分支未提供 `/user/logout` 接口，前端清理 `auth_data` 即视为登出。
- #### 未实现的路由
  `/user/knowledge/getCategory` 在当前源码中没有对应实现，请勿调用。
