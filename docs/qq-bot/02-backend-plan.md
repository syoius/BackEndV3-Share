# 后端执行方案

状态：未实施。依赖：[共享契约](01-contract.md)。执行入口：[后端 prompt](prompts/backend.md)。

## 1. 现有基础与改造边界

已经核验的复用点：

| 文件/模块 | 当前职责 | 本次处理 |
| --- | --- | --- |
| `src/main/kotlin/com/lhs/share/openapi/OpenApiTokenService.kt` | 普通 Token 生成、scope、撤销；绑定一个 accountId，Redis 优先校验 | 保留普通客户端行为；参考权限语义，不直接复用其认证入口 |
| `src/main/kotlin/com/lhs/share/openapi/OpenApiPermission.kt` | inventory/operator scope 目录 | Bot 只使用本期已实现能力的子集 |
| `src/main/kotlin/com/lhs/share/openapi/OpenApiInventoryController.kt` | 普通 Token 库存查询/导入/导出 | 复用其调用的 InventoryService 和事件语义 |
| `src/main/kotlin/com/lhs/share/openapi/OpenApiOperatorController.kt` | 普通 Token 密探查询/导入等 | 复用业务 Service；不能开放扫描导入冒充人工修改 |
| `src/main/kotlin/com/lhs/share/hub/controller/operator/OperatorController.kt` | JWT 局部修改当前密探 | B3 复用局部校正与 expected_revision |
| `src/main/kotlin/com/lhs/share/hub/service/account/SubAccountService.kt` | 统一子账号归属、删除及级联 | 增加 Bot grants 的撤销与当前选择清理 |
| `src/main/kotlin/com/lhs/share/config/security/SecurityConfig.kt` | JWT 及公开路径管理 | 增加明确的机器人认证路由边界 |
| `src/main/kotlin/com/lhs/share/config/security/JwtAuthenticationTokenFilter.kt` | JWT 解析后设置身份 | 与机器人认证隔离，不能将服务密钥当作用户 JWT |

采用独立 `bot_account_tokens` 集合存储 Bot grants/Token，而非修改现有普通 Token 文档格式。目的在于保证已有 `/open-api/**` 查询普通 Token 集合时天然不接受 Bot Token，且不占普通 Token 每子账号 5 个的配额。这不是兼容框架，也不需要迁移旧 Token。

## 2. 数据与配置

全部新增持久化数据放 HubBackend，复用现有 MongoDB 连接和事务配置。

### qq_connections

字段：`id, userId, botInstanceId, platform, senderId, showOtherAccountNames, selectedAccountId, revision, createdAt, updatedAt`。

- 唯一索引 `(botInstanceId, platform, senderId)` 与 `(userId, botInstanceId)`。
- scopes 不重复存于 connection，以 grant/Token 文档为权威来源。
- 解绑事务删除连接及其全部 Token；保留已有操作审计所需标识，不保留可用授权。

### bot_account_tokens

字段：`id(tokenId), connectionId, accountId, token, scopes, createdAt, updatedAt`。

- 唯一索引 `(connectionId, accountId)` 和 `token`；accountId 普通索引用于账号删除级联。
- 文档同时表示一个 grant；移除授权删除该文档。初次授权生成随机凭证，可用明确的 `bot_` 前缀标识类型，但不能仅凭前缀认定有效。
- scope 本期存公开稳定字符串，使用严格 Bot 能力目录验证。不接受通配符或任意数据库 code。
- 每次认证读取有效 Token、连接、子账号归属；第一版不加永久 Redis 授权缓存，避免撤销后仍读到旧权限。
- 明文仅经专用机器领取端点交付；JWT 视图不序列化 token 字段。运营方保管数据库及插件存储；不要声称这里实现了数据库泄露后的凭证保密。
- scopes 更新保留 Token；rotate 原子换发 Token 并更新 tokenId，使尚未确认的旧 Token 操作失效；重新授权生成新 tokenId。

### qq_binding_requests

字段：`id, origin, botInstanceId, userId(nullable), senderId(nullable), bindCode(nullable), ticket(nullable), selection(nullable), state, connectionId(nullable), expiresAt, cleanupAt, createdAt, updatedAt`。

- 活跃 bindCode/ticket 需要唯一约束，空字段不进入唯一索引；遵循 Mongo sparse/partial 索引的实际空值语义。
- 状态变更使用条件更新；最终确认与连接/Token 创建在同一 MongoDB 事务。重复确认读取持久结果。
- bindCode 兑换后保留同身份重试所需的关联，不能允许第二个 QQ 兑换；ticket 认领到第一个 JWT 用户后不可换人。
- expiresAt 为业务期限；cleanupAt 用 TTL 清理，例如创建 24 小时后清理。服务端在操作前显式检查期限，不能把 TTL 当作即时失效机制。
- 已 authorized 的重复查询/确认在保留窗口内返回原结果；不能因后续解绑又将同一请求重新激活。

### bot_operations（B3）

字段：`id, requestId, connectionId, senderId, accountId, tokenId, kind, normalizedPayload, baseRevision, changes, state, expiresAt, result, createdAt, committedAt`。

- 唯一索引 `(connectionId, requestId)`，规范化正文直接比较。
- 保存固定目标及预览数据；confirm 不接受新 payload。
- committed 记录用于重复请求去重与审计，不跟短期绑定码一起清除。过期的未执行预览可以按仓库清理惯例删除，不能清掉幂等所需的已提交记录。
- 与真实写入同事务提交；复用已有业务审计，在 bot_operations 记录 QQ、连接、消息操作 ID 等来源，不把秘密或整个聊天记录写入审计。

### 机器人实例配置

复用 ShareProperties/config 风格，提供实例 ID、展示名称、机器人 QQ、服务凭证、网页基址。服务凭证来自环境或部署秘密配置；前端 bots 响应仅返回公开字段。部署模板使用占位符。

第一版不做机器人注册市场、OAuth 服务、服务密钥管理网页或第三方机器人自助入驻。运营方配置自己的实例即可。轮换服务密钥后旧密钥失效，插件更新配置后恢复，子账号授权无需重建。

## 3. 认证执行顺序

1. 根据路由选择 JWT 管理入口或 Bot 入口。
2. Bot 入口验证服务凭证并固定 botInstanceId，验证发送者头为本期支持的 QQ 字符串。
3. 绑定创建/兑换仅使用上述上下文；不能额外接收可覆盖身份的 JSON 字段。
4. 账号列表及凭证领取只解析该 QQ 的连接，限制可返回范围。
5. 业务入口增加 Token 校验，核对实例、发送者、有效连接、子账号归属和所需 scopes。
6. 生成内部 principal 后调用 Service，所有 userId/accountId 均来自 principal。

实现可用独立 SecurityFilterChain 或项目既有过滤器方式，选择最小且完整的一种。必须用真实 HTTP 层测试证明所有 `/bot-api/v1/**` 路由在缺少专用认证时拒绝请求，不能只测 service，也不能依赖注解文档代替认证。

授权修改、解绑、子账号删除都要覆盖新集合。管理 PUT 使用 expected_revision 防止旧表单覆盖；scope、账号集合和当前选择在同事务更新。子账号已删除时旧 Token 即使因历史数据残留也应因归属校验失败而拒绝。

## 4. 实施步骤

### B1：连接与认证

1. 新建最小 DTO/实体/仓储/配置和 Bot 专用权限目录；保持仓库现有 package 与命名规范。
2. 实现两条绑定起点、原子兑换/认领、状态查询、取消和最终确认。
3. 实现 connection/grant 事务创建、JWT 归属限制、唯一冲突映射。
4. 实现专用服务认证、Token 领取、跨入口隔离及日志脱敏。
5. 实现管理列表、完整替换授权、单 grant 换发和整连接解绑。

### B2：只读闭环

1. 实现机器人账号列表、名称可见性选项及账号切换。
2. 增加库存/密探 current 路由；输出 DTO 复用现有逻辑，游戏版本从子账号推导。
3. 接入子账号删除时 Bot 授权级联；选中账号失效置 null。
4. 生成并核对 OpenAPI：认证头、请求 DTO、HTTP 状态、success/error 包装都可供两端使用。
5. 提供不含秘密的配置模板与联调说明，更新总进度为“已实现待联调”，不提前标记真实 QQ 验收。

### B3：受限写操作

实施前完整读取并遵守：

- [库存后端标准](../standards/inventory/backend-design.md)
- [库存交换协议](../standards/inventory/exchange-protocol-v1.md)
- [机器可读 Schema](../standards/inventory/schema/inventory-exchange-v1.schema.json)

1. 实现 preview/confirm/cancel 和两种 operation payload，使用严格允许字段。
2. 库存设置复用当前运行版本的 InventoryImportRequest、校验器及 InventoryService，生成单项 listed stock_snapshot。标准规定语义，代码中的 v2 子账号包装仍需保留；不能把标准 v1 示例直接当现有在线请求 DTO。
3. 观察时间取已验证消息报告时间并在预览固定；若已有更新使该观察过时，应返回冲突或明确未应用，不能回复“修改成功”而实际没有改变 current。
4. 当前库存直接读 current，不能回放全部奖励；设置绝对值不能伪装 reward_delta。
5. 预览记录相关 current revision；确认在同事务检查 revision 再调用业务写入。密探复用当前校正服务和 expected_revision，不覆盖未出现字段。
6. 操作结果和幂等记录与业务数据一并提交；遇到冲突停止并重新预览。保持已有库存/密探事件和审计行为，避免重试重复广播虚假的新变更。
7. 只有上述路由可用后才把两项 write scope 加入 Bot 授权目录。普通 OpenAPI 的扫描权限无需变化。

## 5. 有针对性的验证

每项验证回答具体故障及处置；不为纯 DTO 镜像增加无意义测试。

| 验证 | 检测的故障 | 失败后的处理 |
| --- | --- | --- |
| 两入口、超时、同身份重试、异身份认领、重复 confirm | 错绑/重复授权/提前签发 | 修正绑定状态及事务条件 |
| 真实 MVC/HTTP 的无服务凭证、错 sender、错账号、JWT/API/Bot Token 互用 | 专用鉴权被绕过 | 修复过滤器路由与 principal 校验 |
| 两子账号不同 scopes、展示开关、旧表单 PUT | 串号、信息超范围、覆盖新权限 | 修复授权查询和 revision 检查 |
| 授权撤销、rotate、解绑、子账号删除后使用旧 Token | 撤权未生效 | 修复授权链和级联删除 |
| Token resolve 重试及重新启动后的领取 | 插件断线导致凭证丢失或不断新签发 | 修复稳定领取逻辑 |
| 同 operation 重复提交/不同正文/事务失败 | 库存重复改动或半提交 | 修复持久幂等和事务 |
| 预览后目标 revision 变化、换 Token、切账号 | 旧确认误改目标或覆盖新数据 | 修复固定目标和确认校验 |
| 既有 OpenApiToken、Inventory/Operator 契约及相关业务回归 | 影响 MaaYuan 和网页已有行为 | 修复公共改动或缩小改动面 |

按实际测试类运行 Gradle 定向测试与 `./gradlew ktlintCheck`。事务行为使用仓库现有 replica set 测试环境验证；无环境则记录未验证，不能以纯 Mock 测试宣称事务通过。必要的完整检查按 AGENTS.md/仓库惯例执行，不在文档阶段启动业务服务。

## 6. 交付要求

实现代码、相应测试、OpenAPI、无秘密配置模板、共享契约差异、准确的总进度记录。后端完成不代表前端/QQ 已联调；不得自动部署公网、发送真实用户消息或写生产数据。

## 7. B4：前端模块图片的数据支持

新增范围详见 [报告与养成卡图片](05-report-and-card-images.md) 和 [snapshot 页面契约](06-bot-snapshot-page.md)，可在 B2 后实施，不依赖 B3。加入两个独立 Bot 图文读取 scope 和两种只读数据组装服务，通过统一 snapshot 创建/读取/销毁接口交付；复用 acquired/listRecords、特别关注、current/annotation 等服务，不能把只有 current 的原接口当作完整出图数据源。

七日报告包含两类 acquired 汇总、七日明细、截至区间终点的 agent 历史、关注名单及完整性标记；时段汇总不受 5000 条明细上限影响。聚合 reward_delta，统计展示公式继续复用前端 acquiredStats.js。密探卡限定目标密探和已保存数据，包含展示需要的 annotation/关注，不能扩大返回整个账号。

更新 OpenAPI 及授权文案，验证新 scope 隔离、跨账号拒绝、日期边界、分页/截断、空数据与部分来源失败。后端只提供受权数据，不运行出图浏览器、不托管公开报告图片。

临时 Redis 记录字段：snapshotId、独立随机 ticket、connectionId、tokenId、accountId、botInstanceId、senderId、requiredScope、kind、固定 data、generatedAt、expiresAt。组装完后设 120 秒 TTL，每次读取同时检查期限及原授权链，不延长 TTL。Redis 失效则本次 snapshot 不可用，不回退到无票据读取；无需落 Mongo、签名框架或一次性领取回执。

创建接口需要完整 Bot 认证；GET /v1/bot-snapshots/{id} 只接受对应短票据并检查授权链，不要求浏览器知道服务秘密/QQ 头；清理接口按实例和发送者核对记录，撤权后仍可清理。长期 Token、JWT 和 snapshot_ticket 三种入口必须有真实 HTTP 隔离测试。配置网页基址用于生成固定截图 URL，仅在创建响应中向受信插件交付短票据，不在普通管理响应或日志中泄露；浏览器只拿本 snapshot 的固定数据。
