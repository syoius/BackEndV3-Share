# QQ 连接共享契约

版本：设计 v1，2026-09-10。**拟实现，尚无可调用接口。**本文件是三端共享语义的来源；实施进度见 [总进度](00-overview-and-progress.md)。

## 1. 身份和权限边界

- `user_id`：网页登录主账号，从 JWT 或连接推导，不接受用户正文声明。
- `account_id`：统一子账号稳定 ID；显示名称不能作为身份键。
- `bot_instance_id`：后端配置的机器人实例标识，与机器人服务凭证关联；插件不能靠请求正文任意声明实例。
- `platform`：本期固定 `qq_onebot`。`sender_id` 为 NapCat 消息事件发送者 QQ 的字符串表示。
- `connection_id`：主账号与 `(bot_instance_id, platform, sender_id)` 的连接。
- `grant`：连接对一个子账号的授权，含 scopes 和对应 Bot Token。
- `selected_account_id`：连接当前选中的已授权子账号；没有授权时为 null。

同一实例内 QQ 与主账号各自至多一条有效连接。已连接另一个主账号时返回冲突，引导先在原主账号解绑，不能通过新绑定覆盖。

机器人服务凭证验证调用方；后端信任该插件提取的消息身份。QQ 号不是秘密，也不是第二因子。服务凭证失窃或受信插件被控制可导致冒用消息身份，应当能够单独轮换服务凭证。正常 QQ 用户不能手填 sender 身份控制工具调用。

## 2. HTTP 认证及返回约定

### 网页接口

前缀 `/v1/qq`，使用现有 `Authorization: Bearer <login_jwt>`。所有连接和绑定请求读取/修改检查当前用户归属。

### 插件接口

前缀 `/bot-api/v1`，所有接口必须先经过机器人专用认证：

```http
Authorization: Bearer <bot_service_secret>
X-Bot-Sender-Id: <event_sender_id>
```

调用子账号业务接口额外携带：

```http
X-Bot-Account-Token: <account_bot_token>
```

后端从服务凭证找到实例，校验 Token → grant → connection → sender → 子账号归属 → scope。缺任何一项即拒绝。不接受正文 `user_id`；业务目标由 Token 推导。若操作正文或路径包含目标账号，必须与 Token 一致。

普通 OpenAPI Token 不能用于 `/bot-api/v1`；Bot Token 不能用于 `/open-api/**` 或 JWT 路径。`/bot-api/v1` 不能仅加入放行名单就结束，必须保证实际路由均执行专用认证。

### 截图页面数据接口

`GET /v1/bot-snapshots/{snapshotId}` 使用独立的 `Authorization: Bearer <snapshot_ticket>`，只读该次临时数据，不接受 JWT、普通 API Token 或长期 Bot Token 替代。创建 snapshot 仍走机器人认证，页面路由 `/bot/snapshot/:snapshotId` 只加载无秘密的外壳；没有有效票据不能取数据。票据绑定范围、120 秒有效期及撤权检查详见 [snapshot 契约](06-bot-snapshot-page.md)。

成功复用 `ApiResult`：`{"status_code":200,"data":...}`，字段 snake_case，时间 RFC 3339。无返回数据的成功统一 `data: true`。拟新增 QQ/机器人接口的失败使用真实 HTTP 状态和 `{"error":{"code":"...","message":"..."}}`，消费端单独解析；不全局修改既有错误包装。沿用业务错误码时保持其原语义。

凭证、绑定请求响应设置 `Cache-Control: no-store`；日志不记录 Authorization、X-Bot-Account-Token、绑定码、领取结果及链接中的授权秘密。

## 3. 权限模型

| scope | 用户展示 | 开放阶段 |
| --- | --- | --- |
| `inventory:read` | 查询库存 | B2 |
| `operator:read` | 查询密探养成 | B2 |
| `inventory:write` | 修改库存数量 | B3，仅本契约允许的写操作 |
| `operator:write` | 修改密探等级与化极/星级 | B3，仅本契约允许的字段 |
| `inventory:report:read` | 库存报告：奖励流水、汇总及特别关注名单 | B4 |
| `operator:card:read` | 密探养成卡：已保存养成、状态、备注及关注标记 | B4 |

这些键复用现有命名，但 Bot Token 只获得机器人路由实际支持的能力。不能因为普通 `operator:write` 支持导入，就给机器人开放任意导入。导出与扫描权限不在本期授权目录内。

两项 `report/card` scope 为新增 Bot 专用能力，不加入普通 OpenAPI 权限目录。它们属于推荐的只读选项，可单独授权，不依赖 write 或普通文件 export；已存在连接不会自动增权。这样不会静默把原来仅查 current 的权限扩展到历史、关注和备注。图片功能完整定义见 [报告与养成卡图片](05-report-and-card-images.md)。

已选子账号推荐勾选两项读取权限；写权限必须用户主动勾选，且本期写权限要求同域 read 同时存在，以支持展示和预览。服务端验证此关系，不能静默扩大提交权限。未实现的写权限不出现在权限目录、不接受授权。

`show_other_account_names` 是连接级选项，默认 false，不是数据读权限。已授权账号及 scopes 总是可列出；未授权账号仅在开启时返回 `{name, authorized:false}`，不返回其 ID、数据、其他客户端 Token 或权限。网页始终通过 JWT 查看全部本人子账号。

## 4. 公共 DTO

授权配置 `AuthorizationSelection`：

```json
{
  "grants": [
    {"account_id": "acc_a", "scopes": ["inventory:read", "operator:read"]},
    {"account_id": "acc_b", "scopes": ["inventory:read"]}
  ],
  "selected_account_id": "acc_a",
  "show_other_account_names": false
}
```

首次授权至少一个 grant，每个 grant scopes 非空、无重复且合法，account_id 不重复且属于当前主账号。管理时允许 grants=[]，此时 selected_account_id 必须为 null；保留 QQ 连接，清空所有数据授权。

连接视图 `ConnectionView`：`connection_id, bot_instance_id, bot_name, bot_qq, platform, sender_id, grants[{account_id,account_name,scopes}], selected_account_id, show_other_account_names, revision, created_at, updated_at`。永不包含凭证。`revision` 用于管理页面完整替换配置时发现旧表单覆盖。

绑定状态 `BindingView`：`request_id, origin(web|qq), state, expires_at, bot_instance_id, bot_name, bot_qq, sender_id(nullable), selection(nullable), connection_id(nullable)`。仅返回当前调用方可访问的请求；插件状态响应不包含主账号 ID、邮箱或未授权子账号。

## 5. 两种绑定入口

### 网页起点

1. JWT 创建请求，提交 `bot_instance_id` 与 `selection`。后端确定主账号，保存草稿，返回 `request_id, bind_code, expires_at`，状态 `waiting_qq`。
2. 用户在指定机器人私聊发送 `/绑定 code`。插件提交 bind_code，后端从已认证实例与发送者头确定 QQ。原子兑换绑定码：首次转到 `waiting_confirmation`，同一 QQ 重试返回同一请求，其他 QQ 不得占用已兑换请求。
3. 原网页登录用户轮询自己的 BindingView，核对 QQ 及授权内容，可在最终确认时修改 selection。
4. JWT 最终确认重新检查子账号归属、scope 和请求有效期，事务创建连接及 grants/Token，状态变成 `authorized`。
5. 插件轮询已兑换请求状态；成功后领取当前 QQ 的 Token。

### QQ 起点

1. `/绑定` 无参数，插件以事件身份创建请求，状态 `waiting_web`，返回短期网页授权链接和 request_id。
2. 链接使用网站固定允许的授权路由，例如 `/connections/qq/authorize#request=...&ticket=...`。ticket 是随机短期秘密，与请求、实例、QQ 固定关联，不能含 Token/JWT。链接片段由页面读取，不作为 HTTP 查询参数记录。
3. 网页完成登录后，以 JWT 在 POST 正文提交 ticket，原子认领请求到该主账号；同一用户可重试，其他用户认领返回冲突。进入 `waiting_confirmation`。打开页面本身不能确认授权。
4. 用户选择子账号、权限，核对发送者及“仅授权自己的 QQ”；点击最终确认后签发。
5. 插件通过请求状态得知成功并领取凭证。若收到他人转发的授权链接，不应授权；最终页面必须清楚呈现将获准访问数据的 QQ。

两条路径共享最终 confirm 接口。等待状态均保留原 expires_at，不因轮询/认领延长。终态是 `authorized / cancelled / expired`；到期由服务端时间判断，不能依赖 TTL 清理是否及时。重复 confirm 只能返回同一 connection，不新增 Token。失败事务不能留下可用的部分授权。

### 绑定接口表

| 方法和路径 | 请求/响应要点 | 调用方 |
| --- | --- | --- |
| `GET /v1/qq/bots` | 返回可连接实例 `[{bot_instance_id,bot_name,bot_qq}]`，无秘密 | JWT |
| `GET /v1/qq/permissions` | 返回当前已实现的 `[{scope,label,description,requires:[]}]` | JWT |
| `POST /v1/qq/binding-requests` | `{bot_instance_id,selection}` → BindingView + bind_code | JWT |
| `POST /bot-api/v1/binding-requests` | 无业务正文；事件身份 → BindingView + authorization_url | 插件 |
| `POST /bot-api/v1/binding-requests/redeem` | `{bind_code}` → BindingView；仅同实例可兑换 | 插件 |
| `POST /v1/qq/binding-requests/claim` | `{request_id,ticket}` → BindingView | JWT |
| `GET /v1/qq/binding-requests/{id}` | 本人已创建/认领请求 → BindingView | JWT |
| `GET /bot-api/v1/binding-requests/{id}` | 仅该实例及该 QQ 发起/兑换过的请求 → 脱敏 BindingView | 插件 |
| `POST /v1/qq/binding-requests/{id}/confirm` | `{selection}` → ConnectionView | JWT |
| `DELETE /v1/qq/binding-requests/{id}` | 取消本人未完成请求 → true | JWT |

bind_code 建议 8 位易输入的随机大写字母/数字，ticket 使用至少 128 bit 随机值。按实例+发送者限制兑换尝试（初值每分钟 5 次），按主账号或实例+发送者限制创建（初值每分钟 3 次）；不要按绑定码单独限流。请求和状态至少保留至到期后的短暂重试窗口，已授权幂等结果由持久化记录保证。具体限流实现复用仓库已有设施。

## 6. 连接管理与凭证领取

| 方法和路径 | 请求/响应要点 | 调用方 |
| --- | --- | --- |
| `GET /v1/qq/connections` | 当前主账号 ConnectionView 列表 | JWT |
| `PUT /v1/qq/connections/{id}/authorization` | `{expected_revision,selection}` 完整替换授权配置 → ConnectionView | JWT |
| `DELETE /v1/qq/connections/{id}` | 幂等解绑，撤销全部 grants/Token → true | JWT |
| `POST /v1/qq/connections/{id}/grants/{accountId}/rotate-token` | 手动换发泄露的 Bot Token，只返回无秘密的连接视图 | JWT |
| `GET /bot-api/v1/accounts` | 返回当前 QQ 的账号列表视图 | 插件 |
| `PUT /bot-api/v1/selection` | `{account_id}` 必须在授权集合 → 账号列表视图 | 插件 |
| `POST /bot-api/v1/credentials/resolve` | `{account_id}` 必须已授权 → `{connection_id,account_id,token_id,token,scopes}` | 插件 |

机器人账号列表视图：`connection_id, revision, selected_account_id, authorized_accounts[{account_id,name,game,scopes}], other_accounts[{name,authorized:false}]`。选项关闭时 other_accounts=[]。

领取端点可以由同一受信实例、同一 QQ 重复调用，返回该 grant 当前有效 Token，解决响应丢失与插件重启；**不会每次领取新建 Token**。这是专用的机器凭证交付接口，不等于普通用户 Token 列表；浏览器不得调用，也不能从连接管理响应取得 Token。此方案明确信任运营方保管服务凭证和数据库，暂不引入单次领取回执或自动续期协议。

修改 scopes 更新同一 grant，Token 明文不变；新增 grant 才签发；移除后重新授权签发新 Token；手动 rotate 使旧 Token 立即不可用。插件遇到 Token 失效，重新查授权并领取一次，不能无限重试，也不能把无权限当作 Token 过期。

删除当前选中子账号或移除其授权时，将 selected_account_id 置 null，不自动换到另一账号执行指令。无选择时提示显式选择，即使只剩一个账号也不能接续此前写操作。

## 7. 业务路由

业务读取、snapshot 创建与写操作使用服务认证、发送者身份和子账号 Bot Token，复用现有业务 Service，不能由插件直连数据库。表中 snapshot 销毁仅需服务认证和归属核对；短票据读取另走专用 GET 入口。

| 路由 | scope | 阶段 |
| --- | --- | --- |
| `GET /bot-api/v1/inventory/current?entity_type=item` | inventory:read；entity_type 可选 item/agent | B2 |
| `GET /bot-api/v1/operator/current` | operator:read；游戏从已授权子账号确定 | B2 |
| `POST /bot-api/v1/snapshots` | 按 kind 验证 inventory:report:read 或 operator:card:read，组装固定数据并签发短票据 | B4 |
| `DELETE /bot-api/v1/snapshots/{snapshotId}` | 仅服务凭证+事件发送者，验证 snapshot 实例/QQ 后清理，不读取业务数据 | B4 |
| `POST /bot-api/v1/operations/preview` | 对应域 read + write | B3 |
| `POST /bot-api/v1/operations/{operationId}/confirm` | 重新检查对应域 read + write | B3 |
| `DELETE /bot-api/v1/operations/{operationId}` | 同连接、QQ、子账号的操作，取消未执行操作 | B3 |

查询 data 沿用现有 current 响应 DTO，后端须在生成的 OpenAPI 中展开实际结构。客户端展示时使用已选子账号名称，不能假定库存 current 只有一个 entity_type。无库存/密探数据不等于无绑定，按现有空数据语义处理。

图片功能改用 snapshot 创建/读取/销毁，不再提供早期设计的 views HTTP 接口。POST 请求 `{kind,params}`：inventory_weekly 的 params.end_date 可省略（默认上海今天及前六个自然日）；operator_card 的 params.operator_id 必填。成功返回 `snapshot_id,kind,account_id,account_name,screenshot_url,expires_at`。专用 GET 数据路由及其响应字段、错误码、票据/资源隔离见 snapshot 契约第 4 节；内部具名 DTO 的来源和完整性继续按图片方案第 3–5 节。snapshot 只用于临时渲染，不是库存 stock_snapshot，不产生业务写入。

preview 请求：`{request_id,kind,payload}`；request_id 是该 QQ 消息对应的稳定操作 ID，插件在同一消息重投/网络重试时保持不变。

| kind | payload | 实际语义 |
| --- | --- | --- |
| `inventory_set_count` | `{entity_type,id,count,observed_at}` | 单项绝对库存，listed stock_snapshot；count 非负整数，观察时间取本次用户报告的消息时间 |
| `operator_patch` | `{operator_id,patch:{level?,star_level?}}` | 仅允许这两个标量的局部校正；至少一个字段；版本、星级范围及图鉴规则复用现有校验 |

本期不开放任意 URL/JSON 透传、库存 full 快照、负奖励、删除流水、升阶扣材料、批量修改、导出或自动扫描伪装。新增功能须有对应契约再接入。

预览返回：`operation_id, account_id, account_name, kind, changes[{field,label,before,after}], expires_at, state`。服务端保存规范化 payload、固定账号、消息身份、Token ID、源 revision、观察时间和 request_id；confirm 无业务正文，只执行保存的内容。Token 换发或撤销、授权不足、目标 revision 变化、超时均不继续提交。

confirm 返回 `operation_id, state:committed, account_id, result`，result 为对应业务响应；同操作重复确认返回既有结果。已取消返回冲突，已过期返回过期，不能换一个 record_id 重试。库存 record_id 从已保存 operation_id 生成，不随重试改变。

“写业务数据 + 标记 committed/保存结果”放在同一 MongoDB 事务；修改请求记录不只存在插件内存中。库存比较相关 current 的 revision，并在同一事务调用既有导入；密探使用既有 expected_revision。发生冲突重新预览、重新确认，不能静默读取新值后强行写入。删除/撤权后，即使重复 confirm 也先检查当前身份与授权，不能向已撤权调用方返回历史敏感结果。

## 8. 错误处理

| HTTP / code | 客户端动作 |
| --- | --- |
| 401 `bot_service_unauthorized` | 运维检查插件配置，用户不能靠重绑解决 |
| 401 `bot_token_invalid` | 刷新授权列表及凭证一次；无授权则停止 |
| 403 `bot_sender_mismatch` | 拒绝，不向发送者透露凭证属于谁 |
| 403 `bot_scope_missing` | 提示网页补授权，不自动补执行原写操作 |
| 404 `qq_connection_not_found` | 提示 `/绑定` |
| 404 `binding_request_not_found` | 不存在或无权读取；不透露他人请求 |
| 404 `account_not_authorized` | 刷新列表，重新选择；不能静默切账号 |
| 409 `qq_already_connected` | 先管理/解除原连接，不自动覆盖 |
| 409 `binding_claim_conflict` | 请求已被其他身份认领，重新发起 |
| 409 `authorization_revision_conflict` | 网页重新载入配置后由用户重新保存 |
| 409 `operation_conflict` / 既有 revision 冲突 | 刷新并重新预览；不自动确认 |
| 409 `operation_cancelled` | 提示已取消 |
| 410 `binding_request_expired` / `operation_expired` | 重新发起；不续用原秘密/确认 |
| 422 `invalid_bot_authorization` / `invalid_bot_operation` | 展示具体字段问题，修正输入 |
| 429 `rate_limited` | 按 Retry-After 等待，不后台循环撞接口 |

服务端发现已提交相同 request_id、不同规范化 payload，返回 operation_conflict；直接比较业务字段，不添加无用途的哈希。其他领域错误保留现有语义，前端/插件为未知错误展示安全的简短提示。
