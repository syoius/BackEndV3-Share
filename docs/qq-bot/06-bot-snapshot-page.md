# Bot snapshot 页面与短期渲染授权

更新时间：2026-09-10。状态：**方案已调整，尚未实现**。本方案采用用户提出的三层结构，替换之前“插件部署独立前端渲染包”的交付方式；不同时维护两条出图链路。

报告/卡片的原组件、数据来源、统计口径和图文 scopes 继续遵守 [图片方案](05-report-and-card-images.md)，跨端 HTTP 身份规则见 [共享契约](01-contract.md)。这里的 snapshot 是临时渲染数据快照，**不是库存协议的 stock_snapshot**，创建它不修改任何库存或养成。

## 1. 三层实现

| 层 | 位置 | 职责 |
| --- | --- | --- |
| 业务页面 | YuanHub 现有 `/inventory`、`/operator` | 查询、编辑和用户操作 |
| 截图页面 | YuanHub 新增 `/bot/snapshot/:snapshotId` | 使用短期票据读取固定展示数据，复用组件，只读渲染 |
| 截图与消息 | AstrBot 插件中的 Playwright | 请求后端创建 snapshot，打开授权 URL，等待就绪，截取并私聊发送 |

前端不需要迁移到 SSR/Next/Nuxt。现有 Vue 3 + Vite + vue-router 可以增加一个不进入菜单的路由，仍随网站正常构建部署。一个页面按服务端 `kind` 渲染 `inventory_weekly` 或 `operator_card`，不需要为不同业务资源各造一套 URL 认证。

插件使用固定的后端/网页地址，只负责向后端申请 URL；不能持有网站签名主密钥自行签发任意账号链接。后端 Kotlin 负责数据与授权，不运行截图浏览器。

## 2. 完整流程

1. 用户私聊 `/七日报告` 或 `/密探卡 司马徽`。插件固定发送者和当前子账号，对密探先解析唯一 ID。
2. 插件使用服务凭证、发送者头和该子账号 Bot Token，POST 创建 snapshot，后端验证对应图文 scope。
3. 后端调用只读数据组装服务，得到一份固定的展示数据，生成随机 snapshot_id 与独立 snapshot_ticket，放入已有 Redis，完成后开始计算 120 秒有效期。
4. 后端返回 screenshot_url，例如 `https://yuanhub.example/bot/snapshot/bs_xxx#ticket=bst_xxx`，以及 kind、账号名称和 expires_at。这里的 URL 是格式示例，不是真实部署地址。
5. Playwright 在独立无登录上下文中打开 URL。前端读 fragment 内的 ticket 到当前页内存，立即移除 URL 中的 ticket；不存 localStorage/sessionStorage。
6. 页面以 `Authorization: Bearer <snapshot_ticket>` 调用专用 GET 数据接口，凭证只发送到配置的后端 snapshot 接口；不使用登录 JWT 或长期 Bot Token。
7. 后端验证票据和当前授权后返回已冻结数据。前端复用组件完成渲染，所有必要字体/图片/统计就绪后标记 ready。
8. 插件截图目标容器；发送前复查创建 snapshot 时那个账号的图文授权，不能改用当前选择的其他账号授权。随后发往原 QQ 私聊。
9. 插件在结束时调用销毁接口并关闭上下文；即使进程崩溃，Redis TTL 也会使票据及数据自动失效。

业务数据在 snapshot 存活期间固定；重复 GET 或多张图片来自同一次组装结果，不每次重新读数据库。`generated_at` 表示组装时刻，并不声称多个业务集合天然属于一个数据库原子快照。若源状态在组装期间变化，按已有完整性提示处理，不能伪造一致性保证。

## 3. 长期 Bot Token 与短期 snapshot_ticket

| 凭证 | 保存位置 | 能做什么 |
| --- | --- | --- |
| bot_service_secret | 插件受控配置 | 证明机器人服务身份 |
| 子账号 Bot Token | 插件内存/受控凭证处理 | 按 QQ 绑定与 scopes 调用机器人业务接口、创建截图授权 |
| snapshot_ticket | 一次截图的 URL fragment、页面内存和后端临时记录 | 只读取指定 snapshot 的固定数据 |

snapshot_ticket 是“持有即可在有效期内读取该 snapshot”的短期能力凭证，不是用户登录，也不验证浏览器使用者的 QQ。它不能查询其他账号/密探、不能修改数据、不能创建其他 snapshot、不能用于普通 OpenAPI/JWT 或 `/bot-api/v1`。

本项目已有 Redis，使用至少 128 bit 的独立随机票据及服务端记录即可；不必为了“签名 URL”这个名称增加自包含 JWT/HMAC 签名系统。实际目标是短期、绑定目标、只读、可撤销。snapshot_id 只标识资源，知道 ID 不等于授权。

初版选择 **120 秒内允许重复读取**，方便页面加载失败重试和多张截图；不在首次 GET 就消费票据。TTL 不因读取而刷新。到期需要插件重新认证后创建新 snapshot，不能拿旧票据续期。重复创建只生成临时只读资源，不触发业务写入，无需建立持久创建幂等框架。

票据关联 `connection_id, token_id, required_scope, account_id, bot_instance_id, sender_id`。每次 GET 都检查原连接仍存在、同一 token_id 仍有效、scope 仍存在、子账号仍归属该主账号；解绑、移除授权、换发 Token 或删除子账号使旧票据不可再读取。无关账号权限变化不应误伤此票据。

短期链接只在后端与插件之间交付，不作为 QQ 回复内容。持有链接的人可能在有效期内看到内容，因此不记录完整 URL，不把它当普通分享链接。已下载或发出的图片不能通过票据撤销收回；撤销阻止之后的数据获取和插件尚未完成的投递。

## 4. URL 与 HTTP 契约

### POST /bot-api/v1/snapshots

使用服务凭证、`X-Bot-Sender-Id`、`X-Bot-Account-Token`，沿用既有 Bot 专用认证。

请求只允许以下之一，account_id 来自 Token：

```json
{"kind":"inventory_weekly","params":{"end_date":"2026-09-10"}}
```

```json
{"kind":"operator_card","params":{"operator_id":"char_105_simahui"}}
```

weekly 的 end_date 可省略，默认上海当天；近七日范围和数据来源见图片方案。拒绝不支持的 kind、额外可变账号或任意 URL。两种 kind 分别要求 `inventory:report:read`、`operator:card:read`。

成功 ApiResult.data：`snapshot_id, kind, account_id, account_name, screenshot_url, expires_at`。URL 由后端固定配置的网页基址生成。插件验证其 origin 和 `/bot/snapshot/` 路径，避免误配到其他站点；不允许聊天输入替代它。

### GET /v1/bot-snapshots/{snapshotId}

仅接受 snapshot_ticket 的 Bearer 认证，不要求服务秘密、QQ 头或登录 Cookie。该路径必须有独立快照读取校验，不能加入公开白名单后直接返回数据。登录 JWT、普通 API Token 和 Bot Token 均不能代替 ticket。

成功 ApiResult.data：`snapshot_id, expires_at, kind, account_id, account_name, game, generated_at, data`；weekly 另含 `timezone, from, to, end_date`。内部 data 是图片方案所需的具名 DTO，正式实现时展开到 OpenAPI，不能返回整个数据库文档或长期凭证。

### DELETE /bot-api/v1/snapshots/{snapshotId}

使用服务凭证及事件发送者，核对临时记录的实例/QQ 后删除，返回 data=true；该清理接口不读取业务内容，不要求仍有效的子账号 Token，以便撤权后的 finally 也能清理。已不存在幂等成功，他人 snapshot 不允许操作。

三接口成功包装沿用 ApiResult，失败沿用本功能的 `{error:{code,message}}`。新增：

| HTTP / code | 语义 |
| --- | --- |
| 401 `snapshot_ticket_invalid` | 缺少/错误票据，或者存在的 snapshot 与 ticket 不匹配 |
| 403 `snapshot_authorization_revoked` | 原连接、Token、scope 或子账号归属已失效，不返回数据 |
| 404 `snapshot_unavailable` | 资源已过期、已销毁或不存在，页面显示“截图请求已失效” |
| 404 `operator_card_not_found` | 创建时没有对应已保存养成卡 |
| 422 `invalid_snapshot_request` | kind/参数无效 |
| 503 `snapshot_store_unavailable` | Redis 不可用，不能回退为公开读取 |

创建及数据响应设置 `Cache-Control: no-store`；snapshot 页面采用 `Referrer-Policy: no-referrer`。网页/API 若跨 origin，按既有部署配置允许固定网页 origin 和 Authorization 头，不为截图放开任意跨域或允许 credential Cookie。

把短票据放在 `#ticket=` 而不是 `?token=`，可以避免它出现在首次页面 HTTP 请求的 URL 中：[MDN 的 URI fragment 说明](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Fragment)。fragment 仍可被页面脚本读取，因此该路由不要采集完整 URL，日志/异常/Playwright trace 也需要避开秘密；前端解析后清除片段。截图浏览器不加载第三方统计脚本。

## 5. 前端页面落点与具体处理

已核验 YuanHub 的以下实际位置：

- 新页面建议 `src/pages/bot/snapshot.vue`，在 `src/router/routes.js` 注册 `/bot/snapshot/:snapshotId`，`display:false`、不 requiresAuth，使用明确 `meta.botSnapshot:true`。
- `src/pages/inventory/index.vue` 自己挂载 IslandSidebar；截图页只导入 report/card，不导入整张业务页面即可避开侧栏/账号选择器。
- `src/App.vue` 有全局弹窗、AccountEventToasts、路由动画与账号 SSE 订阅；截图路由不挂载/启动这些业务交互，离开截图路由后恢复正常页面行为。
- `src/router/index.js` 会统一 authInit，而且 scrollBehavior 将任意 hash 当 DOM 选择器。snapshot 路由需要跳过登录恢复和 hash 滚动，避免把 `#ticket=...` 当锚点或带进登录 redirect。
- `src/main.js` 也引用登录初始化和 reveal 指令，实施时检查真正调用位置，确保截图页无需用户登录即可启动。调整应限定截图路由，不破坏普通页面鉴权。

布局初值：report 宽 900 CSS px，单卡宽 720 CSS px，deviceScaleFactor=2，背景沿用现有暖白/纸色，使用固定浅色方案保证重复截图一致。若未来已有可复用深色主题，再通过明确受支持选项切换，不跟随服务器的随机系统主题。

顶层 `#bot-report-root` 包含子账号、游戏、时间范围/生成时间与 YuanHub 来源标记。长报告按模块提供 `[data-bot-page]` 容器，每页带标题/账号/页码。只读组件去掉编辑和保存按钮，数据不完整提示必须保留，不能为了画面干净隐藏。

`#bot-report-root` 的 `data-render-state` 必须从 loading 转为 ready 或 error：

- 取数、统计计算、Vue DOM 更新、字体就绪、必要图片加载/可解释的 fallback 完成后才能 ready。
- 关闭截图页入场动画，处理头像 lazy loading 与图表/图标所需资源，不能靠固定 sleep 或仅 networkidle 判断完成。
- 数据接口错误显示简短失败状态并标记 error，插件不得把登录页/错误页截图当报告发出。

## 6. 插件截图与部署

截图示意（需实际实现错误、到期、取消和资源清理）：

```python
await page.goto(screenshot_url, wait_until="domcontentloaded")
root = page.locator("#bot-report-root")
await page.wait_for_function("""() => {
    const root = document.querySelector('#bot-report-root');
    return root && ['ready', 'error'].includes(root.dataset.renderState);
}""")
if await root.get_attribute("data-render-state") != "ready":
    raise RuntimeError("Snapshot render failed")
png = await root.screenshot(type="png")
```

Playwright 支持元素截图和图片缓冲区：[官方截图文档](https://playwright.dev/python/docs/screenshots)。长报告逐个截图 data-bot-page，不能重复把整页发多次。开图前配置统一 Asia/Shanghai 时区和浅色上下文；每次任务独立 context，关闭后不保留登录、票据或业务数据。

插件配置继续使用固定 backend_base_url/web_base_url 和运营凭证，不再需要 render_bundle_path 或本地前端静态服务器。部署只新增浏览器运行依赖、网站 snapshot 路由及相应 API。网页服务必须让 `/bot/snapshot/*` 深链接回到 SPA 入口；浏览器所在环境需能访问网站、数据 API 和必要公开素材。

渲染和图片发送设明确超时；过期后仅在该命令仍待回复且授权有效时重新创建一次 snapshot。结束时尽力 DELETE，失败由 TTL 清理。QQ 图片仍按实际 AstrBot/NapCat 适配器支持的字节/base64 方式交付，不假定两个容器共享路径。

## 7. 验收与进度

沿用 B4/F3/A3/E3，不新增另一套里程碑：

- B4：只读数据组装、短期创建/读取/销毁、Redis TTL、票据目标隔离和即时授权检查。
- F3：snapshot 页面与无登录启动、共用 report/card、ready/error 状态、尺寸/资源和业务页面回归。
- A3：申请 URL、打开/等待/截图/发送/销毁、票据不泄露、到期与失败处理。
- E3：两类图片对齐网页；票据不匹配/过期/销毁/撤权均不能读数据；读 A 的票据不能读 B，长期凭证不能走短票据入口；不截登录页/未加载页；普通登录/SSE/卡片编辑正常。

本次只修改设计与 prompt；尚未生成 snapshot 页面、后端接口或插件代码，不能把文档示意代码视为可运行实现。
