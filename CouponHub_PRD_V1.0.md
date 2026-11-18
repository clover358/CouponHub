# 券枢（CouponHub）高并发优惠券营销平台

> 产品需求文档 PRD V1.0  
> 制定日期：2026-11-18  
> 项目性质：Java 后端实习项目  
> 计划周期：4–6 周  
> 状态：五轮核心决策已确认；详细技术设计待完成

## 1. 项目概述

### 1.1 背景与定位

券枢面向电商优惠券营销场景，解决限量抢券中的库存超卖、用户超领、重复发券，以及 Redis、RocketMQ、MySQL 间的数据一致性问题，同时支持面向大量用户的 Excel 异步批量发券。

**已确认**：项目用于 Java 后端实习能力展示，强调业务闭环、高并发、异步消息、故障恢复和可验证的工程质量。提供简易管理后台与用户领券页面，不建设完整商业电商平台。

### 1.2 目标

- 管理员创建满减券模板、配置活动、库存、限领数量与有效期。
- 用户参与限量抢券、查询个人优惠券及领券结果、模拟订单核销。
- 管理员通过 Excel 异步向目标用户批量发券，设计目标为 **10 万人级**。
- 使用 Redis Lua、RocketMQ、MySQL 完成高并发异步领券，并具备幂等、重试、补偿和对账能力。
- 使用自动化测试、JMeter 压测和故障注入验证系统正确性与性能。

### 1.3 非目标

真实支付与退款、完整购物车及物流、商户结算、复杂优惠券叠加引擎、商业级反作弊和多地域容灾不属于 V1.0 范围。

## 2. 角色与权限

| 角色 | 功能 |
|---|---|
| 普通用户 | 注册登录、查看活动、领券、查询领券结果和个人优惠券、模拟核销 |
| 营销管理员 | 模板与活动管理、库存统计、Excel 批量发券、任务进度与失败明细、重试、异常对账查询 |

**建议**：JWT + Spring Security；用户身份由服务端认证信息确定，用户只能访问自己的券和记录，管理员接口实施角色校验。

## 3. 功能需求

### 3.1 优惠券模板管理（P0）

**已确认**：支持满减券；配置优惠金额、最低消费金额、总库存、每人限领 N 张、使用有效期（固定起止日期或领取后 N 天）。

**建议规则**：金额按整数分存储；优惠金额大于 0，最低消费金额不小于优惠金额；库存和限领数为正整数；已发券的核心权益规则不可随意追溯修改。模板状态可采用 `DRAFT / ACTIVE / DISABLED`。

### 3.2 活动管理（P0）

管理员创建、发布、终止活动，设置领取开始与结束时间并关联优惠券模板。用户只能在有效活动窗口且模板允许发券时领取。**建议**活动状态为 `DRAFT / PUBLISHED / RUNNING / ENDED / CANCELLED`，实际可领取性同时校验服务端时间和库存。

### 3.3 高并发领券（P0/P1）

1. 用户认证并提交带幂等键的领券请求。
2. 校验模板、活动时间与领取规则。
3. Redis Lua 原子检查库存和用户额度并预占。
4. 可靠投递 RocketMQ 发券任务。
5. 消费者幂等处理，MySQL 事务持久化用户券。
6. 更新领取结果，客户端查询 `PROCESSING / SUCCESS / FAILED`。

**验收不变量**：不超卖、不超领、同一请求不重复发券；允许用户在 N 张额度内通过不同合法请求领取多张；Redis 预占成功不代表最终发券成功。

### 3.4 用户券查询（P0）

支持个人优惠券分页、状态筛选（未使用、已使用、已过期）和详情查询。**建议**使用 Redis ZSet 加速列表访问，以 MySQL 持久化记录为事实依据；缓存一致性策略待 TDD 细化。

### 3.5 模拟订单核销（P0）

**已确认**：不开发真实订单系统，使用模拟订单核销。提交模拟订单号、用户券实例 ID、原始金额和请求幂等键；服务端确认券归属、未使用状态、有效期及满减门槛。核销成功后原子更新券状态、记录优惠金额及实付金额。同一张券最多成功核销一次；并发和重复请求应安全处理。

### 3.6 Excel 批量发券（P1）

**已确认**：目标支持 10 万人级异步发券；与用户主动抢券**共享模板总库存和每人限领额度**。

流程：上传 Excel → 流式解析与校验 → 持久化任务和明细 → 分批投递 RocketMQ → 消费者使用统一发券能力 → 更新进度 → 查询、导出失败明细及安全重试。

**建议**：Excel 至少包含 `userId`，可包含外部请求 ID；同一任务重复记录不能意外多发；任务中断后可恢复；重试不得重复扣库存；任务取消不自动撤销已发券。

### 3.7 定时补偿与对账（P1）

**已确认**：使用 XXL-Job 支持异常领取扫描、消息重试、批量任务恢复、库存对账和过期处理。所有补偿操作必须幂等；对不确定是否落库的请求，必须先检查持久化事实再决定是否释放预占。

## 4. 业务规则与状态

| 规则 | 要求 | 状态 |
|---|---|---|
| 券类型 | 满减券 | 已确认 |
| 领取方式 | 限量抢券 | 已确认 |
| 每人限领 | 管理员配置 N 张 | 已确认 |
| 使用有效期 | 固定日期、领取后 N 天 | 已确认 |
| 发券渠道 | 主动抢券、Excel 批量发券 | 已确认 |
| 库存与额度 | 两渠道共享 | 已确认 |
| 核销 | 模拟订单 | 已确认 |
| 领取额度 | 累计成功领取数不因核销或过期恢复 | 建议 |
| 请求状态 | `PROCESSING / SUCCESS / FAILED` | 建议 |
| 用户券状态 | `UNUSED / USED / EXPIRED` | 建议 |
| 金额单位 | 整数分 | 建议 |
| 时间、边界与退款 | 统一时区、时间端点和异常回退策略 | 待设计 |

## 5. 技术架构与服务职责

### 5.1 已确认技术栈

- Java、Spring Cloud Alibaba、Spring Cloud Gateway
- MySQL、Redis、RocketMQ、Nacos、XXL-Job
- Vue 3 + Element Plus
- ShardingSphere：预留分片友好模型，后期接入
- JMeter：性能测试

**建议补充**：Spring Boot、Spring Security/JWT、MyBatis 或 MyBatis-Plus、EasyExcel、Maven、Docker Compose、JUnit 5、OpenAPI。具体版本和兼容性仍待 TDD 决策。

### 5.2 服务拆分

| 模块 | 责任 | 数据库 |
|---|---|---|
| `user-service` | 用户注册登录、认证、权限 | `coupon_user_db` |
| `coupon-service` | 模板、活动、批量任务创建与调度 | `coupon_template_db` |
| `coupon-record-service` | 领券、预占、用户券、核销、消费幂等、补偿对账 | `coupon_record_db` |
| `gateway` | 路由、统一认证入口、基础限流与追踪标识 | 无业务数据库 |

**已确认部署**：一个 MySQL 实例、三个逻辑数据库。**建议**各服务只写自己拥有的表；批量任务由 `coupon-service` 调度，实际发券由 `coupon-record-service` 负责。跨服务调用使用 REST/客户端，异步业务使用 RocketMQ。

## 6. 数据模型草案

### 6.1 `coupon_user_db`

- `sys_user`：`id, username, password_hash, role, status, created_at, updated_at`

### 6.2 `coupon_template_db`

- `coupon_template`：`id, name, discount_amount, minimum_amount, total_stock, claim_limit, validity_type, valid_from, valid_to, valid_days, status, version, created_at`
- `coupon_activity`：`id, template_id, activity_name, start_time, end_time, status, created_at, updated_at`
- `coupon_batch_task`：`id, template_id, file_reference, total_count, success_count, failed_count, processing_count, status, created_at, finished_at`
- `coupon_batch_detail`：`id, task_id, user_id, request_id, status, failure_reason, retry_count, updated_at`

### 6.3 `coupon_record_db`

- `user_coupon`：`id, user_id, template_id, activity_id, source_type, claim_request_id, discount_amount, minimum_amount, valid_from, valid_to, status, claimed_at, used_at, version`
- `coupon_claim_order`：`id, request_id, user_id, template_id, activity_id, source_type, status, reserved_at, completed_at, failure_reason, retry_count`
- `coupon_redemption`：`id, user_coupon_id, user_id, mock_order_id, original_amount, discount_amount, final_amount, request_id, created_at`

**待技术设计**：Outbox/可靠投递记录、库存预占流水、领取额度统计、补偿记录、对账记录和任务检查点等是否独立建表。

### 6.4 索引与分片预留

- 领券 `request_id` 唯一；核销请求 `request_id` 唯一。
- 同一张用户券最多存在一条成功核销记录。
- `user_coupon` 支持 `(user_id, status, valid_to)` 等查询路径。
- 批量明细支持 `(task_id, status)` 查询。
- **不得**简单对 `(user_id, template_id)` 建唯一约束，否则会阻止合法的限领 N 张。
- 后期 ShardingSphere 优先评估按 `user_id` 分片，但需验证券 ID 核销、全局唯一性、跨分片统计和分页。

## 7. API 接口规划（建议，非最终契约）

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `/api/auth/register` | 注册 |
| POST | `/api/auth/login` | 登录 |
| GET | `/api/users/me` | 当前用户 |
| POST | `/api/admin/coupon-templates` | 创建模板 |
| PUT | `/api/admin/coupon-templates/{id}` | 修改模板 |
| GET | `/api/admin/coupon-templates` | 模板列表 |
| POST | `/api/admin/activities` | 创建活动 |
| PATCH | `/api/admin/activities/{id}/status` | 活动状态更新 |
| GET | `/api/activities` | 活动列表 |
| GET | `/api/activities/{id}` | 活动详情 |
| POST | `/api/activities/{id}/claims` | 发起领券 |
| GET | `/api/claims/{requestId}` | 查询领取结果 |
| GET | `/api/user-coupons` | 我的优惠券 |
| GET | `/api/user-coupons/{id}` | 用户券详情 |
| POST | `/api/user-coupons/{id}/redeem` | 模拟核销 |
| POST | `/api/admin/batch-tasks` | 上传并创建批量任务 |
| GET | `/api/admin/batch-tasks` | 任务列表 |
| GET | `/api/admin/batch-tasks/{id}` | 任务详情 |
| GET | `/api/admin/batch-tasks/{id}/details` | 任务明细 |
| POST | `/api/admin/batch-tasks/{id}/retry` | 重试失败记录 |
| GET | `/api/admin/batch-tasks/{id}/failures/export` | 导出失败记录 |

**建议统一响应**：`code, message, data, requestId, timestamp`。区分参数错误、未认证、无权限、活动未开始/结束、库存不足、超限、重复请求、处理中、已使用、已过期和暂时不可用等错误。

## 8. 数据一致性与异常场景

### 8.1 已确认可靠性要求

- Redis Lua 原子检查库存和领取额度。
- RocketMQ 异步处理，消费端幂等。
- 消息重试、异常补偿、定时对账、故障注入测试。
- 系统必须保证成功发券不超库存、不超限，同一请求不重复产生业务效果。

### 8.2 故障矩阵

| 故障 | 预期行为 |
|---|---|
| 库存不足或超领 | 拒绝请求，不创建有效预占 |
| 客户端重复提交 | 根据幂等键返回已有请求状态 |
| Redis 预占后 MQ 投递失败 | 可靠重试或补偿，避免悬挂预占 |
| MQ 重复投递 | 消费者幂等，不重复发券 |
| MySQL 写入失败 | 可重试，必要时进入可追踪异常状态 |
| 落库成功但 MQ ACK 失败 | 重投后识别已完成记录 |
| 服务在预占后宕机 | 根据持久化意图/恢复协议处理 |
| Redis 数据丢失 | 基于持久化事实与有效预占恢复 |
| 并发核销 | 同一券至多一次成功 |
| 批量任务中断 | 从持久化进度恢复 |
| XXL-Job 重复执行 | 幂等执行 |

### 8.3 待技术设计的核心协议

1. 库存事实来源与可用库存计算方式。
2. Redis 预占记录如何可靠持久化。
3. MQ 可靠投递：事务消息、Outbox 或其他机制的选型。
4. 预占失败、超时、消息重复及补偿的状态机。
5. 处理中请求如何占用每人限领额度。
6. 主动抢券与批量发券如何竞争同一库存。
7. Redis 故障时的降级策略。
8. 分库分表后幂等与唯一性约束的保持方式。

> 注意：仅有“Lua 扣减 + MQ 消费”不足以证明 Redis、MQ 和 MySQL 的最终一致性；必须设计并验证故障窗口和恢复协议。

## 9. 非功能需求

### 9.1 性能

**已确认**：JMeter 压测；10 万人级异步批量发券为设计目标，而非实测结果。

记录 QPS/TPS、P50/P95/P99、成功率、错误分布、消息积压、任务完成耗时、数据库连接数、Redis 延迟及 CPU/内存。压测报告注明机器配置、实例数量、并发数、持续时间、数据规模和统计口径。不得照搬其他项目的性能数字。

### 9.2 安全

密码安全哈希、JWT 认证、角色权限、用户数据隔离、Excel 文件类型与大小限制、SQL 注入防护、敏感日志脱敏和领券接口限流。

### 9.3 可维护性与可观测性

统一异常与错误码、请求 ID、结构化关键日志、消息发送/消费记录、异常补偿日志、批量任务进度、Docker Compose 本地部署、自动化测试和 API 文档。完整分布式链路追踪平台不是 V1.0 必做。

## 10. 前端页面

**用户端**：登录注册、活动列表、活动详情、领取结果、我的优惠券、模拟核销。

**管理后台**：管理员登录、模板管理、活动管理、库存统计、Excel 批量发券、任务进度、失败明细与重试、异常对账结果。

技术：Vue 3 + Element Plus；以可用、可演示为优先，不追求复杂视觉设计。

## 11. 验收标准

### 11.1 功能验收

- [ ] 管理员创建满减券模板、配置库存与每人限领 N 张。
- [ ] 发布限量领取活动，活动时间约束正确。
- [ ] 两种使用有效期规则均正确生效。
- [ ] 用户领券、查询领取结果、查询个人券正常。
- [ ] 模拟核销校验归属、有效期、金额门槛和状态。
- [ ] Excel 批量任务可创建、执行、查看进度、导出失败明细、重试。
- [ ] 主动抢券与批量发券共享库存和限领额度。

### 11.2 自动化与并发正确性

- [ ] 单元测试、API 集成测试、数据库事务测试。
- [ ] Lua 原子库存测试与 MQ 重复消费测试。
- [ ] 库存 1000、5000 并发请求时成功发券数不超过库存。
- [ ] 每人限领 3 张时，同一用户成功发券数不超过 3。
- [ ] 主动抢券与批量发券同时运行不超卖、不超领。
- [ ] 同一张券并发核销 100 次，最多一次成功。
- [ ] 消费者重启、MQ 重投、数据库暂时不可用后可恢复或补偿。
- [ ] 定时补偿重复执行不会重复释放库存。

### 11.3 性能与故障恢复

- [ ] JMeter 报告包含环境、并发、时长、QPS/TPS、P95/P99、成功率和瓶颈分析。
- [ ] 实际构造 10 万条批量发券数据进行验证并记录结果。
- [ ] 故障注入记录预期、实际结果和最终一致性。
- [ ] 未达到目标时如实记录瓶颈与改进方案。

## 12. 开发里程碑（4–6 周）

| 阶段 | 主要工作 | 交付/验收 |
|---|---|---|
| Week 1 | Maven 多模块、Gateway、Nacos、MySQL/Redis/RocketMQ、登录认证、Docker Compose | 服务注册、网关访问、用户登录 |
| Week 2 | 模板、活动、基础领券、用户券查询、模拟核销、前端页面 | 完整基础业务闭环 |
| Week 3 | Redis Lua、RocketMQ 异步发券、幂等、领取状态、并发测试 | 可验证高并发领券链路 |
| Week 4 | Excel 批量发券、任务进度、重试、XXL-Job、补偿对账 | 可恢复异步批量任务 |
| Week 5 | 自动化回归、JMeter、10 万条批量验证、SQL/Redis/MQ 优化、故障注入 | 性能与正确性报告 |
| Week 6 | ShardingSphere 接入验证、文档、部署演示、简历与面试总结 | 完整工程交付 |

**优先级原则**：如进度延迟，优先保证核心业务正确性与可靠性，可推迟 ShardingSphere，不以牺牲验收质量换取技术栈数量。

## 13. 最终交付物

- Java 后端与 Vue 3 前端源码
- MySQL DDL、Redis Lua、RocketMQ 生产者/消费者、XXL-Job 任务
- Docker Compose 与初始化配置
- OpenAPI 文档、系统架构图、数据库 ER 图
- JMeter 脚本、自动化测试与故障注入脚本
- 性能报告、故障恢复报告、README 和演示说明
- 简历项目描述及关键技术面试总结

## 14. 决策状态与后续工作

### 14.1 五轮已确认决策

1. **项目定位**：Java 后端实习项目；简易管理后台和用户领券页。
2. **业务范围**：满减券、限量抢券、有效期、管理员配置限领 N 张、模拟核销、Excel 批量发券。
3. **架构**：Spring Cloud Alibaba 三业务服务 + Gateway；MySQL、Redis、RocketMQ、Nacos、XXL-Job；Vue 3 + Element Plus；ShardingSphere 后期接入。
4. **可靠性与验收**：重试、补偿、对账、故障测试；功能、自动化、JMeter 和恢复验收。
5. **数据与计划**：一个 MySQL 实例、三个逻辑数据库；批量发券和主动抢券共享库存与额度；10 万人级设计目标；4–6 周开发。

### 14.2 待 TDD 确认

- Maven 工程结构及服务依赖关系。
- Java/Spring Boot/Spring Cloud Alibaba 等组件兼容版本。
- 完整 DDL、索引、ID 生成与事务边界。
- Redis Key、Lua 预占、领取额度与补偿状态机。
- MQ Topic、消息契约、可靠投递协议和消费者幂等。
- XXL-Job 调度、批量任务拆分和恢复。
- ShardingSphere 分片键与路由方案。
- 压测环境、具体阈值和资源预算。

### 14.3 变更原则

PRD V1.0 作为需求基线；新需求先进入后续版本待办。先完成业务闭环、并发正确性和可靠性验收，再考虑额外功能。

---

**成功标准**：项目可运行、可演示、可测试、可解释；所有性能与可靠性结论由实际测试支撑，而非未经验证的宣称。
