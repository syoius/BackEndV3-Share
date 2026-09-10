# 后端执行 prompt

在 `/home/syoius/BackEndV3-Share` 工作区使用。下方可直接复制；若仓库位于其他路径，以当前检出为准。

```text
请在当前 BackEndV3-Share 仓库实现“连接到 QQ”的后端能力，按阶段提交可审查结果。

先阅读：
1. AGENTS.md。
2. docs/qq-bot/00-overview-and-progress.md。
3. docs/qq-bot/01-contract.md。
4. docs/qq-bot/02-backend-plan.md。
5. docs/qq-bot/05-report-and-card-images.md。
6. docs/qq-bot/06-bot-snapshot-page.md。
涉及库存写入前，完整阅读 docs/standards/inventory/backend-design.md、exchange-protocol-v1.md 和 schema/inventory-exchange-v1.schema.json，并检查当前运行版本 DTO/validator。标准语义与当前 v2 子账号包装都必须保留。

任务范围：
- B1：机器人实例配置，QQ 连接，双入口绑定状态，网页最终授权，按子账号 grants/Bot Token，专用鉴权，JWT 管理、换发和解绑。
- B2：机器人授权账号列表、其他账号名称开关、当前账号切换、库存/密探查询、删除子账号时级联撤权、OpenAPI 和联调说明。
- B3：契约定义的 inventory_set_count 与 operator_patch(level/star_level)，后端 preview/confirm/cancel，固定身份/账号/payload/revision，持久幂等和事务提交。
- B4：七日报告和单个密探养成卡的数据组装，以及 inventory:report:read/operator:card:read 两项独立 Bot 只读权限。使用统一 snapshot 创建/短票据 GET/销毁接口，Redis 保存固定数据及 120 秒独立票据；依照已核验组件准备时段汇总、流水/历史完整性、关注及 current/annotation。每次 GET 核对票据与当前连接/Token/scope/账号归属，不能把长期凭证当作短票据。后端生成固定网站截图 URL，插件不能自行签任意账号 URL；不再实现旧 views 端点、不重写前端展示公式、不新增截图服务。B4 在 B2 后即可开展，不必等待 B3。
先完成 B1/B2 并记录检查结果，再推进 B3；真实三端 E1/E2 未完成时保持“待联调”，不要因此把模拟通过写成完整验收，也不要阻塞可独立完成的代码工作。

实施要求：
1. 先查看 git status 和现有 Token/Security/账号/业务服务，保留用户未提交改动。
2. 这是新增 QQ 连接能力，复用业务 Service，不重写库存或密探底座，不把机器人接到数据库。
3. 按共享契约采用独立 bot_account_tokens，保持普通 OpenAPI Token 格式、scope 行为及配额不变。Bot Token 不能在普通 OpenAPI/JWT 路由使用。
4. userId 来自 JWT/连接，botInstanceId 来自服务凭证，senderId 来自已认证插件。业务账号由 Bot Token 推导；任意调用方正文不得覆盖身份。
5. 最终网页 confirm 前没有可用 Bot Token；两个绑定起点、取消、过期、重复兑换/确认、身份冲突必须有明确行为。使用原子状态变更和 Mongo 事务，不留下半授权。
6. 网页列表不含秘密。机器 resolve 可以为该实例/QQ 的已授权账号重复取得当前凭证，解决重试/重启，不每次新签发。
7. scopes/连接/账号归属每次校验；撤销、换发、删除账号后旧 Token 不再通过。不要增加永久授权缓存。
8. 第一阶段权限目录只列已实现的 read；write 端点实现后再开放对应权限。新子账号不自动授权，其他账号名称开关不授予内容访问。
9. 写操作仅限共享契约两种 payload，后端保存预览并固定账号。重复确认只写一次；revision 冲突要求重新预览。业务数据和 committed 结果在同事务提交。
10. 库存绝对设置用 listed stock_snapshot，不能用 reward_delta 或 full；沿用 record_id 幂等及快照基线规则。现有领域校验、审计和通知不要被跳过。
11. 采用仓库既有 package、配置和错误处理方式；新增 QQ 路由的 success/error 包装按契约，不全局改变旧接口。
12. 仅对具体失败风险写测试。必须覆盖真实 HTTP 鉴权边界、绑定幂等/冲突、权限隔离与撤销、写操作事务/重试/并发，以及相关普通 API 回归；按仓库要求运行检查和 ktlintCheck。
13. 服务端版本、外部库 API 或安全配置不确定时查官方资料；不要靠猜测实现。需要外部数据库/真实 QQ 的验证无法运行时，说明实际缺什么，不伪造通过。
14. 本任务不自动部署公网、不发送消息给真实用户、不写生产数据。不要引入无用途的哈希、OAuth 平台、迁移框架或通用代理。

遇到契约与代码不一致：根据领域标准和现有实现判断，先在 01-contract.md 记录明确调整及原因，再同步受影响的三端方案；不要在代码里默默改变语义。正常实现选择自行处理，确实缺失的仓库/运行信息才简短提问，并继续不依赖它的工作。

交付：
- 后端实现和有针对性的测试。
- 可供前端/插件使用的真实 OpenAPI、无秘密配置示例及接口差异说明。
- 更新 00-overview-and-progress.md 对应 B1/B2/B3/B4 状态，附改动位置、测试命令/结果、未完成依赖。
- 最终简述完成内容、验证结果和剩余联调。不要把后端完成等同三端上线。
```
