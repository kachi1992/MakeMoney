# 05 · 后端架构、数据库与可靠性

## 1. 建议架构

模块化单体 API + 独立任务 worker；PostgreSQL 管事实/账本/审计，对象存储保留许可允许的原始材料，持久队列管调度，检索先全文后按需增加向量扩展。建议 Python 类型化 API 和计算模块；依赖、数据库版本与许可证在 P0 锁定。首版不需要微服务集群或图数据库。

```text
iOS → Auth / API / Entitlements → Research / Signals / Paper / Evaluation
Providers → Ingest → Normalize + Quality → Evidence + Features → AI
Calendar → Daily Freeze → Risk → Orders → Ledger + NAV → Mature Labels
Experiments → Validation → Shadow → Approval → Future Strategy Version
```

LLM worker 不能写 Paper 账本。交易与评估是可重放的确定性模块。事件投递用 transactional outbox，采用至少一次投递加幂等，而非未经证明的 exactly-once。

## 2. 逻辑数据表

PK 默认 UUID，时间 UTC timestamptz，货币/数量 NUMERIC。下表为设计，不是已执行迁移。

| 表 | 必要字段/约束 |
|---|---|
| users / sessions | auth_subject 唯一、状态、locale/timezone；刷新令牌仅存 hash |
| consents / entitlements | 用户、目的、服务商范围、版本、授权/撤销时间；权益有效期 |
| issuers / securities | 企业与证券分离、CIK、财年、类型、币种、ADR、上市/退市 |
| ticker_aliases | security、ticker、exchange、生效起止，区间不重叠 |
| universe_snapshots / members | as_of、规则、用途、hash、成员及排除理由 |
| data_licenses | 数据集、展示/AI/衍生/存储/训练权利、地域、到期、合同引用 |
| source_documents | provider+external_id+hash 唯一、对象地址、源/首次/可用时间、license |
| evidence_chunks | document FK、locator、text_hash、语言、许可范围 |
| financial_observations | issuer、metric、period、value、unit、basis、dimensions、revision、available_at、source |
| derived_metrics | 公式版本、input IDs/hash、值、质量、available_at |
| price_bars / corporate_actions | security、源、原始价格/行动条款、时间、revision、可用时点 |
| events / analyses | 事件簇、公司、状态、时间、引用；结构化结果及模型/prompt/校验版本 |
| feature_snapshots | security、cutoff、universe、values、quality、input_hash、版本 |
| strategy_versions | code_hash、参数、feature/execution/label 版本、审批状态 |
| signals | security+batch+strategy 唯一、评分块、cutoff、evidence、snapshot_hash |
| recommendations | signal、portfolio run、action、前/目标权重、冻结/执行/过期、revision、状态 |
| portfolio_runs | owner 或 system、模式、初始资金、策略/成本/执行/基准版本、起止 |
| paper_orders | run、recommendation 可空、side、数量、session、state、idempotency key |
| fills | order+session+sequence 唯一、数量、价格、费用、事件时间 |
| ledger_events | run、类型、有符号现金、security/quantity、source_ref、reversal_of、recorded_at |
| position_lots | entry_fill、remaining_quantity、成本基础；可从账本重建 |
| nav_observations | run+valuation_time+revision、NAV/现金/应收、陈旧标记、hash |
| evaluation_labels | suggestion/signal、horizon、label_version、entry/exit、状态、cohort |
| experiment_runs / approvals | 数据清单、划分、trial 历史、结果 hash；审批人/理由/未来生效时间 |
| watchlist_items | user+security 唯一、version、updated_at |
| jobs / outbox | type、payload_ref、unique_key、lease、attempts、state、next_run |
| notifications / audit_events | 用户/资源/版本/去重键；actor/action/resource/hash/time/request_id |

## 3. 版本与索引

金融观察采用源有效期+系统记录期；历史查询筛 available_at≤cutoff，再选当时可见 revision，不能简单取今天最新。更正追加，禁止普通角色更新已冻结 signal、recommendation 或 ledger。

索引：issuer+metric+period+available_at；security+bar_time；event available_at；run+ledger_time；strategy+decision_date；evaluation maturity/status。批量财报查询按公司与指标范围限量，历史文件走对象存储而非大 JSON 列表。

冻结 manifest 保存 universe、源 hash、特征、策略/模型/prompt/检索、许可、交易日历、执行/标签版本。模型不能保证重新生成字面一致时保存当次输出；后续评分和交易必须确定性重放。

## 4. 事务、权限与幂等

模拟成交在同一数据库事务锁定订单与账户：检查剩余数量/资金→写 fill→写现金/持仓事件→改状态→写 outbox。唯一键防重放重复成交；并发买单不能重复消耗现金。余额为账本派生，不直接手调。

每个私有 API 从已验证身份计算 owner scope，不信任用户提交的 owner_id。DAO/数据库行级规则做第二道隔离。运营只能追加纠错，策略审批与运行记录审计。对象链接短期签名，发链接时重新校验权利。

## 5. 工作流和调度

discovered→fetched→normalized→validated→analyzed→signal_frozen→execution_scheduled→settled→labels_pending/matured。每步含幂等键、lease、心跳、超时、重试与死信队列；业务口径/许可失败不能无限重试。

截止检查 readiness，部分公司失败按预注册缺失规则处理并显示覆盖变化，不能秘密换入后来知道会涨的标的。错过开盘不补记。调度使用交易会话而非固定 UTC 小时；后台行情延迟也不会改变真实提交时间。

## 6. 备份、扩展与恢复目标

初版设计目标 RPO≤24h、RTO≤4h；账本在正式前瞻阶段启用更细恢复能力，目标 RPO≤15min，必须通过演练后才能声称达到。备份加密、异地/隔离、定期恢复检查；数据库和对象 manifest 一致校验。

先用只读缓存/物化视图提高首页和曲线性能，再考虑读副本/分区；不能先缓存未完成鉴权的个人数据。任务独立按采集/AI/回测队列配额，回测不能挤占每日冻结和账本。

## 7. 可观测性与审计

贯穿 request/job/batch IDs，记录阶段时延、watermark、来源失败、成本、质量拒绝、冻结晚点、拒单、账本校验差额。严禁 token、商业原文和个人持仓默认进入日志。需要修历史时创建 correction/reversal、影响列表和新报告 revision，旧报告保留或依法受限，不能静默美化收益。
