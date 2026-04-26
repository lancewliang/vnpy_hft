# 期货交易系统架构设计（基于 vnpy_ctp）

## 1. 文档目标

本文基于当前讨论结果，给出一套先落地、可扩展、避免过度设计的分布式交易系统架构。

系统覆盖范围：

- 行情采集（国内期货）
- 因子计算与策略决策
- 下单/撤单/成交执行链路
- 风险控制与告警
- 监控与日终结算
- 数据存储、归档与回放

不包含范围：

- 代码实现细节
- 超高频（毫秒以下）交易优化

### 1.1 四项目定位与关系

`vnpy_hft` 的建设目标是将研究、因子工程与交易执行统一为一个可生产落地的系统。当前涉及 4 个项目：

| 项目 | 定位 | 与 `vnpy_hft` 的关系 |
| --- | --- | --- |
| `ArchetypeTrader` | 交易策略研究项目（策略决策逻辑） | 为 `vnpy_hft` 提供策略判定与信号生成能力 |
| `MacroHFT_Features_SH` | 因子工程项目（训练数据与因子计算） | 为 `vnpy_hft` 提供实时因子计算与特征加工能力 |
| `vnpy` | 量化交易系统基础框架 | 为 `vnpy_hft` 提供交易接入、事件驱动与执行链路基础能力 |
| `vnpy_hft` | 统一的高频量化交易系统项目 | 承接前三者能力，形成一体化实盘系统 |

### 1.2 第一阶段整合目标（代码移植）

- 将 `vnpy` 的交易接入与执行能力作为底座（重点使用 `vnpy_ctp` 及其事件链路）
- 将 `MacroHFT_Features_SH` 的因子工程代码移植为 `Factor Engine` 在线服务
- 将 `ArchetypeTrader` 的策略决策代码移植为 `Strategy Engine` 在线服务
- 在 `vnpy_hft` 内完成统一的数据、风控、调度与监控闭环

## 2. 关键业务约束（已定稿）

- 目标交易所：上海期货交易所（SHFE）、中国金融期货交易所（CFFEX）
- 关键交易接口：`vnpy_ctp`
- 交易日定义：`T日 = T-1 夜盘 + T日上午 + T日下午`
- 决策调度：每秒进行一次判定（`1s scheduler tick`）
- 决策频率：仅在“单位行情完成”时触发（`unit_seconds` 可配置，20/30 秒等）
- 因子计算口径：因子由最近 `x=unit_seconds` 秒的 1 秒五档行情聚合计算，因子表 `1` 行代表 `x` 秒单位行情结果
- 因子与策略内存窗口：固定保留最近 `400` 秒 1 秒五档行情，同时保留对应最近 `ceil(400/x)` 行因子结果
- 行情服务职责边界：仅负责行情接收与消息发布，不负责标准化、窗口聚合和数据库落库
- 行情主题约束：原始行情发布主题采用 `md.tick.raw.<product_id>`（如 `md.tick.raw.rb`、`md.tick.raw.IF`）
- 行情落库职责：由独立归档服务消费 `md.tick.raw.*` 后写入 PostgreSQL
- 单位窗口职责：`unit_seconds` 窗口闭合由因子服务与策略服务各自维护和触发
- 前置约束：必须满足“当前委托已结束（成交/撤单）+ 持仓新鲜 + 行情新鲜 + 风控通过”后，才允许新一轮策略
- 信号时效：策略结果带有效期，过期不得下发到执行层
- 超时委托：需定义超时时间，触发自动撤单
- 日终要求：交易日下午结束后调用外部结算接口拉取各品种全部合约结算数据，生成结算报告，并计算次交易日订阅合约（默认每个品种选择当日成交量前4主力合约）

## 3. 技术选型（已定稿）

- 交易接入：`vnpy_ctp`
- 消息中间件：`NATS JetStream`
- 关系数据库：`PostgreSQL`
- 实时缓存：`Redis`
- 归档介质：`NAS`（文件归档）
- 归档格式：行情以 `CSV` 为主
- 回放数据源：归档文件（不依赖 PostgreSQL）

明确不采用：

- `ClickHouse`
- `MinIO`

## 4. 总体逻辑架构

系统按职责分为 7 个域：

- 行情域：行情接入与原始事件发布
- 决策域：因子计算、策略判定、信号时效控制
- 交易域：下单、撤单、成交回报、委托状态机
- 风控域：事前拦截、事中监控、风险阈值告警
- 数据域：归档服务落库、PostgreSQL 全量存储、Redis 实时态、NAS 归档
- 结算与订阅规划域：外部接口结算同步、次交易日订阅计划生成
- 运维域：监控、报警、任务调度

## 5. 模块职责

### 5.1 行情采集服务（Market Collector）

- 代码来源：以 `vnpy` 行情接入能力为基础，统一接入 `vnpy_hft` 事件总线
- 通过 `vnpy_ctp` 接收行情
- 发布原始行情事件到 `md.tick.raw.<product_id>`
- 轻量化原则：仅做接收、编码、发布与订阅切换，不承载业务计算和存储
- 不负责行情标准化、单位窗口聚合与数据库落库
- 支持次交易日按订阅计划热切换合约（不重启服务）

### 5.2 行情归档服务（Market Archive Service）

- 独立消费 `md.tick.raw.*` 原始行情事件
- 负责行情标准化与幂等处理
- 写入 PostgreSQL 行情表（append-only）
- 执行实时库与历史库归档策略，输出 NAS 文件归档

### 5.3 因子服务（Factor Engine）

- 代码来源：主要移植 `MacroHFT_Features_SH` 因子工程代码，并改造为在线计算服务
- 因子计算口径：`x = unit_seconds`，每个因子行由最近 `x` 秒（1 秒五档行情）聚合计算
- 维护内存窗口：固定 `400` 秒原始五档行情窗口 + 对应 `ceil(400/x)` 行因子窗口
- 由因子服务自身维护 `unit_seconds` 窗口闭合，并在窗口完成时计算因子
- 写入因子表（每单位行情一行）
- 发布 `factor.unit.window.close`、`factor.unit.calculated` 事件

### 5.4 策略服务（Strategy Engine）

- 代码来源：主要移植 `ArchetypeTrader` 策略决策代码，并适配实盘调度与风控闸门
- 每秒触发一次调度检查
- 维护策略侧 `unit_seconds` 窗口闭合，仅在窗口完成时进行策略判定
- 判定前检查闸门：无活动委托、持仓快照版本有效、行情新鲜、风控通过
- 输出策略信号（含 `generated_at`、`expire_at`、`cycle_id`），可发布 `strategy.unit.window.close`

### 5.5 执行服务（OMS/Execution）

- 代码来源：以 `vnpy` 交易执行与委托回报机制为底座，在 `vnpy_hft` 中增强超时与状态管理
- 接收策略信号并校验有效期
- 发出委托指令（经 `vnpy_ctp`）
- 记录下单事件
- 管理超时撤单
- 记录多次撤单尝试
- 记录成交明细

### 5.6 风控服务（Risk Service）

- 事前风控：下单前限制（仓位、风险敞口、限价偏离等）
- 事中风控：委托超时、撤单失败、成交异常
- 事后风控：日终风险归因
- 输出风险事件和告警事件

### 5.7 调度与结算服务（Scheduler & Settlement）

- 日内调度（1 秒 tick）
- 日终调度（收盘后结算任务）
- 调用外部结算接口，拉取各品种全合约结算数据与成交量
- 生成每日结算报告
- 触发次交易日订阅规划任务

### 5.8 订阅规划服务（Contract Subscription Planner）

- 读取日终结算同步结果
- 按品种对合约成交量排名
- 默认规则：每个品种选择当日成交量前4合约作为次交易日主订阅合约池
- 输出并持久化次交易日订阅计划（含 `effective_trading_day`、`cutover_time`）
- 将订阅计划发布给行情采集与策略服务（次交易日生效）

### 5.9 监控告警服务（Observability）

- 服务可用性、队列积压、处理延迟
- 行情延迟与数据新鲜度
- 归档服务落库延迟与失败率
- 下单成功率、撤单成功率、拒单率
- 风控触发频率与关键告警升级

## 6. 服务通信机制

### 6.1 异步事件总线（NATS JetStream）

建议主题（subject）：

- `md.tick.raw.<product_id>`：原始秒级行情（按品种分片）
- `md.tick.archived`：行情归档写库完成事件
- `factor.unit.window.close`：因子服务单位窗口完成事件
- `factor.unit.calculated`：因子结果
- `strategy.unit.window.close`：策略服务单位窗口完成事件
- `strategy.signal`：策略信号
- `order.submit`：下单请求
- `order.cancel`：撤单请求
- `order.event`：委托回报
- `trade.fill`：成交回报
- `risk.alert`：风险告警
- `system.alert`：系统告警
- `settlement.contract.synced`：外部结算接口同步完成
- `contract.volume.rank`：品种合约成交量排名
- `settlement.daily`：日终结算完成
- `contract.plan.generated`：夜盘/次日合约池

### 6.2 同步查询通道

- 使用 gRPC/HTTP 提供状态查询：当前持仓、当前活动委托、风控参数、策略运行状态

### 6.3 幂等与顺序

- 语义：`at-least-once`
- 幂等键：`event_id`、`order_ref`、`trade_id`、`cycle_id`
- 顺序性建议：行情按 `instrument_id` 保序，交易按 `account_id` 保序
- 扩展建议：因子和策略服务可按 `product_id` 订阅 `md.tick.raw.<product_id>`，实现多实例分片消费

## 7. 核心业务机制

### 7.1 策略周期闸门

- 每次策略运行分配唯一 `cycle_id`
- 当前周期存在未终态委托时，不进入下一周期
- “终态”定义为：已成交完成或已撤单完成

### 7.2 信号时效控制

- 策略信号必须包含 `expire_at`
- 执行层在下单前检查时效
- 过期信号直接丢弃并告警

### 7.3 超时撤单

- 委托单配置 `timeout_ms`
- 超时后自动发起撤单
- 多次撤单均记录（不覆盖历史）

### 7.4 持仓参与策略

- 策略输入必须包含当前持仓
- 为避免竞态，持仓读取需与“活动委托状态”联动校验版本

### 7.5 因子口径与窗口约束

- 因子单位为 `x` 秒，`x` 由 `unit_seconds` 配置决定
- 每个 `factor_unit` 行对应一个单位窗口区间（`unit_start_ts` 到 `unit_end_ts`）
- 因子服务与策略服务均消费 `md.tick.raw.<product_id>` 并各自维护窗口闭合
- 内存窗口固定覆盖最近 `400` 秒原始行情，对应保留 `ceil(400/x)` 行因子结果

### 7.6 日终结算与次交易日订阅

- 每日下午收盘后调度器调用外部结算接口，拉取“按品种全合约”结算与成交量
- 对每个品种按当日成交量降序排名
- 默认选取每个品种成交量前4的合约，作为次交易日订阅合约池
- 生成订阅计划后发布事件并持久化，供次交易日开盘前加载
- 行情采集服务在 `effective_trading_day` 到达时按 `cutover_time` 原子切换订阅合约

## 8. 数据存储策略

### 8.1 PostgreSQL（主事实库）

- 保存全量数据
- 逻辑上 append-only（只插入，不更新）
- 行情数据由行情归档服务写入；交易与结算数据由对应业务服务写入
- 采用 PostgreSQL 声明式分区（逻辑仍是一张表）

### 8.2 Redis（实时态）

- 存放短期运行状态：活动委托集合、最新持仓快照、调度闸门状态、风控临时计数

### 8.3 NAS（归档）

- 由归档服务定期从 PostgreSQL 导出归档文件
- 行情优先导出 `CSV`
- 回放读取归档文件，而非 PostgreSQL

## 9. 性能目标（防过度设计）

- 调度频率：每秒一次触发检查
- 决策触发：单位行情结束后执行
- 时延目标：单位行情就绪到下单请求产出 `p99 <= 500ms`
- 执行目标：下单请求到交易接口回执 `p99 <= 1s`
- 稳定性目标：交易时段可用性 `>= 99.9%`

## 10. 监控与报警

### 10.1 数据质量

- 行情延迟
- 原始行情发布延迟（`md.tick.raw.<product_id>`）
- 归档服务入库延迟与失败率

### 10.2 交易链路

- 下单成功率
- 撤单成功率
- 拒单率
- 超时撤单触发次数

### 10.3 系统健康

- 服务存活
- JetStream 消费堆积
- PostgreSQL 慢查询与连接池
- Redis 内存与命中率

### 10.4 报警分级

- `Warning`：人工关注
- `Critical`：触发保护动作（限流、停策略、人工介入）

## 11. 部署建议

- 服务按功能独立部署，便于横向扩容
- 关键顺序通道按 `account_id` 分片，避免账户级竞态
- 行情与因子可按 `instrument_id` 水平扩展
- 生产环境建议主备部署 PostgreSQL 与 NATS
- 行情服务默认部署规范：`1 collector instance -> 1 product_id -> 前4主力合约`
- `vnpy` 技术上支持单实例订阅多个品种多个合约，但生产环境不建议作为默认方案
- 多品种混合订阅仅建议在开发、联调或低流量场景使用，正式上线前需完成容量压测

## 12. 配置项清单（建议纳入配置中心）

- `unit_seconds`（单位行情秒数）
- `decision_scheduler_interval=1s`
- `order_timeout_ms`
- `signal_ttl_ms`
- `max_cancel_attempts`
- `factor_window_seconds=400`
- `md_subject_pattern=md.tick.raw.{product_id}`
- `md_product_shard_subscriptions`
- `market_collector_publish_buffer`
- `market_collector_reconnect_backoff_ms`
- `subscription_cutover_time`
- `archive_writer_batch_rows`
- `archive_writer_batch_ms`
- `market_freshness_threshold_ms`
- `position_freshness_threshold_ms`
- `settlement_run_time`
- `settlement_api_endpoint`
- `contract_selection_metric=volume`
- `contract_selection_top_n=4`
- `archive_export_cron`
- `archive_nas_path`
