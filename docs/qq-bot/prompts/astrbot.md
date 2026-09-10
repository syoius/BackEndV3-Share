# AstrBot 插件执行 prompt

在用户指定的插件工作目录使用。本任务开发的是 AstrBot 插件，不是 Codex 插件。

```text
请开发一个连接 BackEndV3-Share 的 AstrBot 插件，使用 NapCat / OneBot v11，面向 QQ 私聊。按阶段完成可安装源码、配置、指令、测试和使用说明。

先阅读工作区 AGENTS.md，以及已提供的：
- docs/qq-bot/00-overview-and-progress.md
- docs/qq-bot/01-contract.md
- docs/qq-bot/04-astrbot-plan.md
- docs/qq-bot/05-report-and-card-images.md
- docs/qq-bot/06-bot-snapshot-page.md
这些路径属于后端仓库；若未提供，先定位可读文档。核验实际 AstrBot 安装版本并查官方插件/消息事件/配置/存储文档，不使用过时模板猜 API。

若当前已有插件结构，沿用它；若用户已指定空插件目录，就在该目录创建所需文件。不在后端目录里随意安装整套 AstrBot 或 NapCat，不使用 Codex plugin-creator 来创建 AstrBot 插件。

实现范围：
A1：
- 后端固定基址、网页基址、实例 ID、预期机器人 QQ、服务秘密配置。
- 私聊 /绑定 创建 QQ 起点请求并给网页链接；/绑定 code 兑换网页请求，等待网页最终确认。
- 有限时轮询和 /绑定状态；成功后机器领取对应子账号 Bot Token，绝不发到聊天。
- /账号、/切换账号、/管理授权；多个子账号各自 Token/scopes，后端保存当前选择。
- /库存 [名称]、/密探 [名称] 查询及清晰展示账号、空状态和权限问题。
A2：
- /设置库存 名称 数量 → inventory_set_count 预览。
- /设置密探 名称 等级/星级 数值 → operator_patch 预览。
- /确认 操作号、/取消 操作号，固定账号/对象/变化，冲突重新预览。
A3：
- /七日报告、/密探卡 名称，使用图文读取授权 POST 创建 snapshot，后端返回固定网站 /bot/snapshot/:snapshotId#ticket=... 短期 URL；不在插件保存网站签名主密钥。
- 用 Playwright 打开该 URL，页面仅使用 120 秒 snapshot_ticket 取固定数据，复用 report/card 与统计函数。等待 #bot-report-root 的 ready/error，ready 才截取 PNG；不得把长期 Bot Token/服务秘密/登录 Cookie 放进浏览器，不部署独立前端包或复制模板。
- 图片按固定请求账号隔离、长报告分模块分页、发往原私聊，发送前复查该账号授权，结束后 DELETE snapshot 并清理图片/浏览器资源。票据不得进入日志、QQ 或 Playwright trace；到期只为仍待回复且授权有效的命令重新申请一次。核验 NapCat 分容器时字节/base64 传输，不默认文件路径互通。
- 验证图片中的数值、日期范围、缺数据/历史截断提示与网页一致；不增加定时订阅或公开图床。A3 可以在只读基础上推进，不依赖 A2。

要求：
1. sender 只取 NapCat 消息事件的真实发送者，self ID 校验到配置机器人；群会话、昵称、@对象和引用消息不能代替。未匹配实例的事件不使用本实例凭证。
2. 第一版只在私聊处理；群里不展示任何个人账号、链接或绑定码。
3. 服务凭证放受控插件配置；Bot Token 按实例+sender+account 分区缓存，可内存保存、重启后 resolve 恢复。不无限签发、不要求重启后重新绑定。
4. 异步 HTTP 使用固定后端地址，设置超时、关闭客户端资源，分辨 401 服务配置错误、Token 失效、403 权限不足、404 无连接、409 冲突、410 到期和 429。
5. 不把用户输入作为任意 URL、请求头或内部身份；不做通用接口代理。不使用登录 JWT 或管理员 Token 访问业务。
6. 子账号切换仅限最新授权列表，名称重复要求选择；其他未授权名称仅展示，不能选择或读取。选中账号被撤权后不能静默换到另一个账号继续执行。
7. 绑定状态到 authorized 才回复成功。request_id 等非秘密等待元数据可短期保存以恢复状态；轮询到期即停。
8. 查询/修改对象通过公共目录解析稳定 ID；多候选时询问，不猜测。查询为空不能误报未绑定。
9. 写操作使用后端 preview，显示 account_name 和 before/after。确认与原 account_id/operation_id 绑定，用户切换默认账号不改变待确认操作。
10. 每个消息保持稳定 request_id，网络/消息重投不能创建新写操作。confirm 超时仅复用 operation_id；成功要以服务端 committed 为准。
11. 预览后撤权/换发/冲突不得自动继续旧修改。不要领取新 Token 后自动补执行被拒绝的写操作。
12. 本期不要求自然语言模型。以后接入模型也只能解析白名单业务参数，身份、权限、Token、绑定码与确认均在确定性代码里处理。
13. Token、Authorization、绑定码、链接秘密不进入日志、异常回复、模型上下文或仓库文件。安装说明使用占位符；OneBot Token 和后端服务 Token 分开说明。

测试重点是消息来源隔离、多账号 Token 隔离、绑定流程、重启恢复、权限撤销、固定确认目标和重复提交，不写只镜像实现的测试。官方 API 使用当前版本资料；真实 QQ/NapCat 环境未提供时记录待联调，不擅自发送消息给真实用户或操作生产数据。

交付符合实际版本要求的插件源码、metadata、配置 schema/依赖、浏览器安装与网站 snapshot/API 连通说明、指令帮助和验证结果。按 A1/A2/A3 更新或输出可回填的总进度记录，不能把 Mock 测试或文字回退称为真实 QQ 图片验收。
```
