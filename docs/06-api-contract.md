# 06 · iOS—后端 API 契约

版本 /v1；以下为待实现设计，仓库 JSON schema 定义冻结建议这一关键对象，其余端点按本目录在对应阶段补 OpenAPI 与可执行契约测试，不能宣称所有接口已上线。

## 1. 统一协议

HTTPS+JSON，Authorization: Bearer；UTC RFC3339 时间、UUID 资源 ID、Decimal 金额/价格用字符串，比例默认 0—1，排序分 0—100。currency 明示 USD 等，session_id 为交易所会话身份而非用户本地日期。

成功：`{"data":...,"meta":{"request_id":"...","data_as_of":"...","revision":1,"next_cursor":null,"warnings":[]}}`。列表以稳定排序键+ID 游标分页，默认 limit=20、最大100，不以易漂移 offset 作为正式历史账本分页。

错误：`{"error":{"code":"...","message":"...","request_id":"...","retryable":false,"details":{}}}`。字段质量使用 null+reason，不返回 NaN/Infinity。ETag/If-None-Match 用于读取；变更用 expected_revision/If-Match 防覆盖。

## 2. 端点目录

| Method / path | 输入 | 输出/权限 |
|---|---|---|
| POST /auth/apple | identity_token, nonce | 验证后服务端 session；禁止仅解码即信任 |
| POST /auth/refresh | refresh token | 旋转令牌，撤销旧 token |
| POST /auth/logout | session | 撤销当前 session |
| GET /me | — | 自己的设置/权益/同意 |
| PATCH /me/preferences | locale, timezone, notification prefs | 更新版本 |
| POST /me/consents | purpose, scope, version, granted | 同意记录 ID |
| POST /me/export | dataset scope | 202 job；不含无权再分发源数据 |
| DELETE /me | 近期认证+确认 | 202 deletion job/保留说明 |
| GET /companies | q, exchange, cursor | issuer 与 security 映射、覆盖状态 |
| GET /companies/{id} | — | 概览/财年/状态 |
| GET /companies/{id}/financials | period_type, metrics, as_of | 值、单位、期间、basis、来源、revision |
| GET /companies/{id}/events | as_of, types, cursor | 事件簇、披露/发生时间、分析状态 |
| GET /companies/{id}/analyses | as_of, cursor | 有版本研究结果 |
| GET /calendar/earnings | from, to, universe | 预计/确认日期分开 |
| GET /calendar/sessions | exchange, from, to | 开收盘 UTC、时区、半日市/休市状态 |
| GET /today | session, watchlist filter | 日报、覆盖、任务状态 |
| GET /watchlist | cursor | 当前自选 |
| PUT /watchlist/{security_id} | — | 幂等添加 |
| DELETE /watchlist/{security_id} | — | 幂等删除 |
| GET /evidence/{id} | version | 片段、locator、许可允许的原文链接 |
| POST /research/questions | company_ids, as_of, question | 202 job，需 AI 同意和配额 |
| GET /research/answers/{id} | — | 回答、引用、局限、校验状态 |
| GET /recommendations | session, strategy, action, cursor | 正式冻结建议，不含 draft |
| GET /recommendations/{id} | revision | 当时版本、证据、执行/评价链接 |
| GET /signals/{id} | — | 独立于账户的评分事实 |
| GET /paper/portfolios | — | 系统公开盘+仅自己的盘 |
| POST /paper/portfolios | initial_cash, mode, strategy_version, cost_model | 201 新 run，资金范围校验 |
| GET /paper/portfolios/{run_id} | as_of | NAV/现金/应收/风险/状态 |
| GET /paper/portfolios/{run_id}/positions | cursor | 数量、成本、估值状态 |
| GET /paper/portfolios/{run_id}/nav | from,to,interval | 曲线及基准同起点数据 |
| POST /paper/portfolios/{run_id}/orders | security_id, side, quantity, session_id | 201 order，仅手动模式，必须幂等键 |
| GET /paper/portfolios/{run_id}/orders | state,cursor,idempotency_key | 订单和执行原因 |
| POST /paper/orders/{id}/cancel | expected_revision | 已成交返回冲突，不伪称取消成功 |
| POST /paper/portfolios/{run_id}/pause | cancel_pending=true, expected_revision | 暂停新自动订单、取消所有未执行自动订单；持仓保留 |
| POST /paper/portfolios/{run_id}/resume | expected_revision | 从未来批次恢复，不重放过期单 |
| POST /paper/portfolios/{run_id}/fork | new parameters, archive_old | 201 新实验；旧账本不改 |
| GET /evaluations/recommendations/{id} | horizon=20 | mature/immature/unexecutable/missing，预测与交易分开 |
| GET /evaluations/strategies/{version} | from,to,horizon | 样本量、收益、风险、置信区间/缺失 |
| GET /strategies | state | 已批准可见版本 |
| GET /strategies/{version} | — | 参数、限制、实验和审批记录 |
| GET /notifications | cursor | 通知历史与资源版本 |
| POST /devices | APNs token, environment | 绑定当前用户设备 |
| DELETE /devices/{id} | — | 注销自己设备 |
| POST /feedback | resource_id, revision, category, text | 反馈 ID；内容限长并安全处理 |
| POST /billing/transactions | signed transaction | 后端核验后更新权益 |
| GET /jobs/{id} | — | 仅任务所有者/授权运营可读 |

运营接口 `/admin/experiments`、`/admin/strategies/{version}/approve`、`/admin/corrections`、`/admin/killswitch` 使用独立角色和审计，不在客户端暴露万能写权限。生产策略晋级必须未来生效，不能修改已冻结订单。

## 3. 下单及幂等

Idempotency-Key 作用域 user+run+key；同 key 同 body 返回同订单，同 key 不同 body 返回 409。数量为正整数字符串，卖出≤可用持仓，模式、现金、session 和风险在后端检查。客户端不能传成交价、最终状态、收益或任意策略代码。

例：`{"security_id":"<uuid>","side":"buy","quantity":"20","session_id":"XNAsynthetic-session"}`。示例 session 为占位符，不能用于生产日历。响应包含 order_id/status、估算金额、执行模型/会话、风险说明及服务器时间。超时后先按 key 查询，不换 key 重下。

暂停合同统一为默认撤销所有未执行自动订单（包括买/卖），保留已有持仓；用户在确认页可明确选择不撤，服务器返回实际取消列表。该合同优先于页面文案中“默认撤买单”的早期描述；实施前同步 UI，防止相互矛盾。

## 4. 错误与重试

400 invalid_request；401 authentication_required；403 forbidden/data_license_required/ai_consent_required；404 resource_not_found；409 revision_conflict/idempotency_conflict/order_already_final；422 insufficient_cash/insufficient_position/ineligible_session/risk_limit/unsupported_security；429 rate_limited（Retry-After）；503 provider_unavailable/analysis_delayed。

GET 可有界退避重试，POST 只有带幂等语义才自动重试。权限错误不重试。客户端不展示内部堆栈、token、数据库语句；详情可给 field 与安全原因码。

## 5. 异步问答和流式协议

创建问题返回 202+job ID，可轮询。可选 SSE `GET /jobs/{id}/events`，使用 event_id、类型 started/progress/evidence/final/failed、payload；断线用 Last-Event-ID 恢复。草稿 token 不作为通过事实校验的正式答案；只有 final 引用完整才可保存/分享。取消任务是 best effort，不宣称能撤回已经完成的供应商计费。

## 6. 权限、兼容和验收

全部私有路径由 session 推导 owner；猜到 UUID 不能读取他人账户。每次证据下载检查 license/territory/retention。枚举新增须兼容 unknown，破坏性变更用 /v2；任何 key/单位/状态机修改必须同步 Swift 解码样例、schema 和迁移。

验收：分页重试不重复；同 key 重放不重下；乱序 revision 不回退 UI；403 不泄漏付费原文；断线问答可恢复；历史 as_of 不能拿到未来材料；系统盘禁止用户写操作。
