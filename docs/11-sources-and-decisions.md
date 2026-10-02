# 11 · 官方资料、决策与待确认项

资料核验日期：2026-10-02。以下链接为设计参考，不是已经采购、获许可或通过审核的证明。API、价格、条款和商店要求应在实施/签约/发布前复核。文档中的默认权重、阈值、成本、样本量和时延是本项目提出的实验/工程目标。

## 1. 来源登记

| ID | 官方来源 | 支持的具体事项 |
|---|---|---|
| S01 | [SEC EDGAR APIs](https://www.sec.gov/search-filings/edgar-application-programming-interfaces) | submissions/XBRL、无需API key、聚合范围、frames和bulk |
| S02 | [SEC Developer Resources](https://www.sec.gov/about/developer-resources) | 公平访问聚合限速、索引与自动访问要求 |
| S03 | [Nasdaq Symbol Directory Definitions](https://www.nasdaqtrader.com/trader.aspx?id=symboldirdefs) | 当前证券目录字段、测试证券/ETF等分类参考 |
| S04 | [FMP Terms of Service](https://site.financialmodelingprep.com/terms-of-service) | 数据及衍生信息用途、授权、保密/安全、终止处理；以实际协议为准 |
| S05 | [Massive Stocks API Overview](https://massive.com/docs/rest/stocks/overview) | 行情/参考/公司信息API类别；不等于本项目获商业授权 |
| S06 | [Alpaca Paper Trading](https://docs.alpaca.markets/us/docs/paper-trading) | 模拟与实盘限制、股息等差异；本项目不默认集成其API |
| S07 | [FINRA Rule 2214](https://www.finra.org/rules-guidance/rulebooks/finra-rules/2214) | 对相关会员工具的方法、局限、假设披露参考 |
| S08 | [Apple App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) | 金融服务、第三方内容、个人数据分享至AI、商店审核 |
| S09 | [Apple Account Deletion](https://developer.apple.com/support/offering-account-deletion-in-your-app/) | App内发起账户删除及相关体验 |
| S10 | [Apple Background Execution Limits](https://developer.apple.com/forums/thread/685525)；[Push Troubleshooting](https://developer.apple.com/documentation/usernotifications/troubleshooting-push-notifications) | 后台执行用途及推送限制，交易调度放服务器 |
| S11 | [Apple StoreKit](https://developer.apple.com/storekit/) | 购买、交易和订阅集成能力 |
| S12 | [SEC Investor.gov: Investment Advisers](https://www.investor.gov/introduction-investing/getting-started/working-investment-professional/investment-advisers) | 证券建议/分析业务及报酬要素；不是本产品注册结论 |

只基于上述官方来源说明公开能力/条款；没有引用第三方“据说价格”。不把前面会话中的NVDA/MSFT虚构数值或未来示例当真实数据写入产品。

## 2. 架构决策 ADR

| ID | 决策 | 原因/代价 |
|---|---|---|
| ADR-01 | 原生SwiftUI+后端持续任务 | iOS体验与可靠调度分离，需维护后端 |
| ADR-02 | 模块化单体+worker | 降低首版复杂度，保留后续拆分边界 |
| ADR-03 | 数值/评分/交易确定性，LLM解释 | 可复现可审计，需更多结构化工作 |
| ADR-04 | 30家试点→扩容，不等同全市场 | 验证口径和成本；早期结论范围有限 |
| ADR-05 | 日批冻结、下一会话开盘模拟 | 清晰防倒签；忽略部分盘中机会，撮合有简化 |
| ADR-06 | 系统盘/用户自动盘/手动盘隔离 | 策略归因清楚，账户状态更多 |
| ADR-07 | 长仓无杠杆、不接真实券商 | 限制风险与范围，不代表公众建议当然免责 |
| ADR-08 | 默认概率null、分数非概率 | 防误导，需积累独立校准样本 |
| ADR-09 | 追加修订、PIT快照、失败试验留存 | 防泄漏和挑成绩，增加存储/许可要求 |
| ADR-10 | 候选先样本外+shadow+人工批准 | 避免每日追亏调参，不保证能找出有效策略 |
| ADR-11 | 不将每次新模型历史重跑当实时预测 | 模型可能记忆未来，依靠前瞻证据 |
| ADR-12 | 暂停默认取消全部未执行自动订单 | 与API/QA统一；确认界面列出范围，已有持仓不清空 |

## 3. 仍需实施前确认

首批证券名单与行业参考池；公司数据/新闻/电话会/PIT供应商及商用报价；AI供应商、模型、保留政策与预算；最低iOS/稳定Xcode版本和测试机；主体/目标市场/合规结论；收费模式及权益；历史原文/账本留存权；基准数据许可；退市/复杂行动处理覆盖。

这些未知不妨碍开始合成数据开发，但阻止相关真实数据发布或正式研究结论。没有足够历史/PIT资料时从前瞻试点开始，明确限制，不假造历史。

## 4. 文档版本与本次交付

v1.0 初始化README；v1.1 纳入用户要求的每日买卖建议、标准/个人模拟盘、准确度评分与持续策略优化，补齐15份设计文档、接口目录、关键JSON schema和合成样例。

本次交付仅设计与契约。App工程、后端接入、完整OpenAPI、策略回测、真实前瞻结果、TestFlight/上架与商业许可均未因本次提交自动完成。验收必须按文档08/09实际执行，不得把文档存在当成软件实现通过。
