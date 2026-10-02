# 02 · iOS 技术设计

## 1. 技术选型与边界

建议 Swift + SwiftUI，最低部署目标暂定 iOS 18，使用项目启动时可提交 App Store 的稳定 Xcode/SDK；具体版本、设备矩阵与依赖在 P0 实测后锁定，不在本设计中声称某版本为最新。初版不依赖仅最新设备支持的端侧模型。

端上负责 UI、缓存、鉴权、前台同步和通知接收；后端负责采集、分析、评分、模拟执行及训练。Apple 后台任务/后台推送不作为精确定时工作保证，服务器任务不依赖手机唤醒。[S10]

选择轻量 feature-first 模块化及单向状态更新，避免一开始引入复杂跨平台或微前端。货币用 Foundation Decimal 的可测试包装；展示层不重新计算官方评分或净值。

## 2. 建议目录

```text
ios/MakeMoney/
  App/                 # bootstrap, dependency container, navigation
  Core/Networking/     # APIClient, auth refresh, errors, retry
  Core/Persistence/    # local cache, schema migration, outbox
  Core/Models/         # Money, TradingSession, identifiers, DTO mapping
  Core/DesignSystem/   # typography, state views, charts, disclosure badges
  Features/Onboarding/
  Features/Today/
  Features/Research/
  Features/Recommendations/
  Features/PaperPortfolio/
  Features/StrategyLab/
  Features/Settings/
  Services/            # notifications, purchases, auth, analytics
  Resources/           # localization, synthetic fixtures
  Tests/               # unit, contract, UI and snapshot tests
```

这只是实施目标目录，本次不创建虚假的空 App 工程作为完成证明。

## 3. 状态与数据流

View → FeatureModel（主线程可观察状态）→ Repository protocol → API client / Cache actor。网络 DTO 与 Domain Model 分离；服务依赖通过初始化注入，测试替换为 fixtures。用户动作通过明确 action 触发，复杂页面 state 包含 data、loading phase、freshness、permission、error、revision。

使用 async/await，取消页面离开后的无用读取任务；搜索 debounce 并取消旧请求，响应按 request generation 丢弃过时结果。网络/解码不阻塞 UI。所有共享可变缓存/令牌刷新用串行隔离，避免多个 401 同时刷新。

NavigationStack 的 route 使用 resource type + ID + version；从推送进入历史建议时拉对应版本，不自动跳最新。深链不携带凭证或可信权限，仍需后端校验。

## 4. API 客户端

HTTPS；短期访问令牌与旋转刷新令牌放 Keychain；证书与密钥不打包。超时、取消、ETag/If-None-Match、cursor 分页和统一错误映射封装在 APIClient。GET 可带退避重试，POST 仅当明确幂等时重试；模拟下单统一生成 Idempotency-Key 并复用，不能重试时换 key。

数字金额和价格使用十进制字符串解码；UTC 时间 RFC3339，TradingSession 独立类型。未知枚举落到 unknown 状态并保留 raw value，不默默解释为买入或成交。分页防重复、防遗漏，历史 record revision 不覆盖本地旧证据版本。

在网络失败但服务器可能已接受订单时显示“状态待确认”，先按 idempotency key 查询，禁止用户重复下单凑成功提示。

## 5. 缓存与离线

优先使用 SwiftData 或 SQLite 封装，P0 基准后选定；缓存只存授权允许的字段与期限。日报、公司摘要、最近建议、个人模拟概览可离线读；全文文档与商业数据不默认长期持久缓存。

cache key 包含 user/entitlement scope、resource ID、version、locale；账号切换清理私有缓存。退出/删除清除令牌和私有持仓数据。权限到期执行 tombstone/到期清理，不能把数据展示权缓存永久化。

自选操作可入本地 outbox 并用幂等 PUT 同步；订单、策略切换、账户重置必须在线确认，不离线排队自动生效。启动先呈缓存，再前台增量刷新；显示 data_as_of 而非手机请求时间冒充数据时间。

## 6. 账户与同意

建议 Sign in with Apple 为首选，邮箱登录可后续扩展；后端验证身份 token 的签名、issuer、audience、nonce 与有效期，再发自己的会话令牌。测试环境使用明显的演示登录，不留生产后门。

AI 同意分开记录：通用公开材料分析不需要传用户资料；用户问答可能包含自选/模拟持仓时需说明发送范围、服务商和保留规则，允许拒绝并继续基本浏览。用户原始文本先提醒不要粘贴敏感账户数据。

App 内设置提供账户删除请求，二次身份验证；展示预计保留/清理状态与订阅取消提示。删除账户不等于自动取消 App Store 订阅，UI 需给管理订阅入口并明确说明。[S09]

## 7. 推送与刷新

APNs token 绑定用户/安装实例，变更更新，失效注销。通知 payload 最小化，只含类型、资源 ID、版本和安全文案；不含 API key、全文或默认可见的个人持仓。

通知点击后仍需鉴权和拉数据，push 不是数据真值。前台 polling 只对正在看的任务/订单在限频范围内进行；后台任务只用于机会性预热，不驱动任何金融账本。夏令时与周末显示由后端下发 trading session 辅助，客户端本地时间只影响显示。

## 8. 订阅与实验权限

付费功能设计为研究深度、历史范围和实验数量，不售卖“必中建议”。使用 StoreKit 处理产品、购买/恢复与交易状态；服务端按官方验证机制核验，不能仅信客户端付费标志。具体商店支付/外链规则按发布时所在 storefront 复核，默认路径不依赖豁免。[S11]

免费/试用/订阅的权益在后端统一控制。退款、过期、离线、恢复购买和重复交易都要测试。订阅过期不把账户账本删除；新增实验限制与历史查看权由已公示政策处理。

## 9. 测试与交付

单元：Money/时间/enum、分页、状态机、Keychain wrapper、缓存迁移、权限过期、幂等查询。契约：使用 contracts/ 样例验证解码及缺失字段。UI：五 Tab、深链、离线、误触保护、大字体、VoiceOver、暗色、长公司名、长中文/英文混排。

网络测试注入 401/403/409/429/503、乱序响应、重复响应、SSE 中断；模拟订单永远以服务器状态为准。性能测试注明设备、系统、网络、数据规模；无工具实测前不得声称达到 PRD p95 目标。

## 10. 端侧安全限制

禁用任意 JavaScript 执行型财报阅读；外部证据在安全浏览器/清理后的展示容器打开。日志不得包含完整 token、用户聊天全文或私有模拟持仓。崩溃埋点采集最小数据且经适用同意，分析错误与用户投资行为不混同。
