# 03 · 数据接入与财务特征引擎

状态：待实施；关联 FR-001/003/004/005/007/009。来源编号见 [11](11-sources-and-decisions.md)。

## 1. 数据来源与接入边界

| 来源 | 首版职责 | 限制 |
|---|---|---|
| SEC EDGAR | 申报目录、10-K/10-Q/8-K 及外国发行人材料、XBRL | 不是完整新闻/电话会/共识源 |
| 公司 IR | 业绩新闻稿、指引、原始演示材料 | 全文展示、抓取与 AI 处理需核对权利 |
| FMP 等商业财务提供商 | 标准化财务、日历、可采购的共识/电话会 | 逐数据集核实历史、口径与商业用途 |
| Massive 等行情提供商 | 原始 OHLCV、公司行动、证券映射、授权新闻 | 个体套餐不自动等于公众 App 展示许可 |
| Nasdaq symbol directory | 当前证券目录及类型参考 | 不提供完整无幸存者偏差的历史公司池 |

SEC submissions/companyfacts 不需要 API key，但聚合 XBRL 有非自定义 taxonomy、实体整体等范围限制；分部/自定义事实需补充原始申报解析。[S01] SEC 当前公布的公平访问上限为每用户聚合 10 请求/秒；本系统初始设全服务聚合 5 请求/秒，附访问身份、缓存和退避，不用多机器或代理绕限流。[S02]

证券类型通过目录及供应商主数据交叉核验，过滤测试证券/ETF 等，不能只根据名字猜测。[S03] FMP 的数据和衍生用途按协议审批；供应商替换走相同适配器和差异校验。[S04][S05]

## 2. 身份、覆盖与历史公司池

issuer_id 表示企业，security_id 表示证券，CIK 表示申报实体；ticker 只是具有生效期的别名。分别保存交易所、证券类型、币种、ADR 比率、财年结算日、行业分类版本、上市/退市时间。一家公司多股类不能重复汇总营收，组合敞口按 issuer 合并。

每日保存 universe_snapshot 及成员/排除理由。推荐试点池与因子标准化参考池分别版本化。退市、合并、改名的历史保留；不可用今天仍存在的 30 家公司回测后宣称代表历史整个纳斯达克。资料不适配的银行、保险、REIT 等保留研究入口，专用评分未实现前显示不适用。

每家公司输出 coverage matrix：财务、行情、事件、共识、指引、电话会分别标 available/partial/unavailable/unlicensed。公司数量和更新时延必须实测，不引用销售宣传作为本产品保证。

## 3. 接入流水线

首次：主数据→CIK 映射→申报目录→历史材料/财务→价格/行动→质量校验→覆盖报告。大批历史优先 SEC bulk，再用 submissions/索引做增量，避免全市场逐公司高频轮询。[S01][S02]

增量：发现材料→许可检查→provider ID/accession 去重→下载并 hash→安全解析→标准化→质量检查→事件/指标版本→分析任务。业绩新闻稿可早于正式季报，先展示原始事件，待完整材料后补全；不能承诺财报发布后一律一两分钟生成完整分析。

原始文档在许可范围内不可变存储：document_id、provider/external_id、accession、source_url、MIME、content_hash、object_key、source_event_at、source_published_at、first_seen_at、processing_completed_at、available_at、license_id、状态。抓取失败与没有新材料是不同状态。

## 4. Point-in-time 与修订

前瞻数据 available_at 不早于源公开、系统首次看见与必需处理完成时间。决策只能读 available_at≤cutoff 的版本；财报期末不是公开时间。源时间缺失时用保守规则并显式标记。今天下载的历史重述值没有自动的 PIT 资格。

更正/重述新增 revision 与 supersedes 引用；当前视图可显示最新值，但历史建议使用原时点版本。财报对比必须说明是否采用后来的重述口径，不能把后见信息带回历史训练。SEC frames 接近日历期间，不是不同企业财政季度天然一致的保证。[S01]

## 5. 财务观测模型

必须字段：issuer_id、canonical_metric、taxonomy/tag、period_start/end 或 instant、fiscal_year/period、value_decimal、unit/currency、scale、GAAP/IFRS/non-GAAP、consolidation_scope、segment/dimensions、source accession、evidence anchor、available_at、revision、quality。

选值不能只看 tag；context、单位、合并/分部范围、原始/修订也参与。钱用 Decimal，保留源精度；换币采用明确 FX 时点与来源，原币金额不丢弃。图表四舍五入不回写原值。

## 6. 季度转换与可比性

利润表/现金流为区间，资产负债表为时点。累计现金流按可比原始期间拆分：Q2=半年累计−Q1；Q3=前三季累计−半年累计；Q4=全年−前三季累计。必须币种、会计口径、合并范围与修订一致，否则禁止相减并说明原因。

EPS、加权平均股数、利润率不是可直接相减的流量；不得全年 EPS 减前三季 EPS 构造 Q4 EPS。TTM 只相加不重叠的四个独立季度；利润率以合计金额重新相除，不直接平均季度百分比。

52/53 周财年保留真实天数；跨公司比较显示各自最新披露期间和陈旧程度。外国发行人可能使用 20-F/40-F/6-K，不应假设必有 10-Q 或四份标准季度数据。[S01]

## 7. 指标定义

| 指标 | 确定性规则 |
|---|---|
| 营收 YoY/QoQ | 相应可比期值>0 时 current/prior−1；否则 null+原因 |
| 增长加速度 | 本次同比−上次同比，单位百分点 |
| 毛利率/营业利润率 | 对应利润/revenue；收入≤0 不适用 |
| CFO margin | 同期 CFO/revenue，先正确拆季度 |
| FCF | CFO−资本开支现金流出绝对额，标注本系统定义 |
| TTM | 四个连续、独立、同口径季度金额之和 |
| FCF conversion | FCF/net_income，净利润≤0 不适用 |
| CapEx intensity | 同期资本开支/revenue |
| EPS surprise | 与发布前同期间/GAAP/稀释口径共识比较；预期为0仅差额，负预期百分比用差额/abs(expected)并注明 |
| 指引变化 | 同目标期同指标的上下限/中点比较，不能新季度对旧季度称上调 |
| EV | 同一发行人有效市值+债务+适用权益调整−现金，分项时点/来源齐备 |
| EV/TTM revenue | 有效 EV、正收入、行业适配时计算，缺项或不适用不硬算 |

共识必须有公布前快照；缺失时不写“超预期”。股数、市值、拆股和汇率也必须 as-of，不能今日市值配历史财务。定量评分规则见文档 12，非 GAAP 和 GAAP 始终分开。

## 8. 校验、故障和更新

校验类型/单位/范围、季度年度桥接、资产负债平衡、利润率反算、股数/拆股一致性；容差随源精度记录，不随测试结果放宽。供应商冲突回查原始来源，不取平均掩盖差异。

missing/conflict/stale/not_applicable/unlicensed 独立状态；关键冲突阻止评分，不用 LLM 填零或补估计。财务新披露/修订触发更新，主数据每日及变更触发，行情按实际采购粒度。监控 watermark、429、解析失败、关键字段缺失和各公司数据年龄。
