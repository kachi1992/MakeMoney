# 机器可读契约

`recommendation.schema.json` 为 v1.1 正式冻结建议的 JSON Schema（2020-12）。完整业务API目录见 [文档06](../docs/06-api-contract.md)，完整OpenAPI和其他对象schema在对应开发阶段补齐。本目录不是已经部署的服务。

`../examples/recommendation.synthetic.json` 是正例；`../examples/evidence.synthetic.json` 是对应虚构证据；所有值都是合成数据，不表示真实公司/价格/推荐/业绩。

## 必须在应用层补充的校验

JSON Schema只校验结构；另外要求：评分块唯一、权重和为1、coverage等于有效块权重和、分数按文档12重算、缺值有原因、引用确实存在且获许可、前/目标权重和动作一致、issuer/行业/总仓位约束成立。ABSTAIN可为null分数；非空分数不意味着数据足以交易。Rules-v1概率必须null。

时间：原始输入的available_at≤data_cutoff_at；日批派生计算允许在cutoff之后、frozen_at之前完成，但仅能读取截止前已经可用的原始/已解析事实。computed_at≤frozen_at≤published_at<eligible_execution_at≤expires_at；截止后的新原始材料不能借派生计算带入。发布后再有新解释必须另存revision。这里区分原始信息截止和批次计算完成，不能以computed_at晚于cutoff为由读取新事实。

snapshot为不可变内容；成交/取消/撤回等生命周期事件追加保存，不修改冻结分数和理由。策略、特征、执行与标签版本不得省略。

默认暂停合同：取消所有未执行自动订单并停止新自动订单，持仓保留；用户可明确选择不取消，但必须返回实际范围并审计。所有UI/后台遵守同一规则。

## 本次检查边界

已在本地使用Python jsonschema校验schema本身和合成示例的UUID/时间格式，并核算78.25加权分、权重和、时间顺序、引用包含关系。未运行App、后端API、真实行情、回测、模拟引擎或App Store测试；这些需按文档09实施。
