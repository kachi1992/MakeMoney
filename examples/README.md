# 合成数据示例

本目录没有真实股票建议、交易或市场数据。

- `recommendation.synthetic.json`：正式建议快照结构正例。虚构因子按0.20/0.20/0.15/0.20/0.15/0.10加权得78.25，示意目标仓位9.5%，不是已验证策略结果。
- `evidence.synthetic.json`：对应虚构证据，来源URL为空，不能对外展示成SEC材料。
- `strategy-experiment.template.json`：尚未登记运行的实验模板。历史/前瞻起止与结果为null，须真实完成后填写，不伪造。

推荐样例中的session标为SYNTHETIC，不能送生产交易日历或任何券商API。零值hash仅为占位；生产必须计算真实输入hash。生产发布必须拒绝synthetic=true或未解析占位符。
