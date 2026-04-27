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

### 1.3 术语定义（统一口径）

- `product_id`：交易品种标识（如 `rb`、`al`、`IF`），用于主题分片与服务部署分片
- `instrument_id`：具体可交易合约标识（如 `rb2610`、`al2609`、`IF2606`），用于逐合约计算与幂等
- `vt_symbol`：`vnpy` 本地合约代码，格式为 `instrument_id.exchange`（如 `rb2610.SHFE`）
- 映射关系：一个 `product_id` 下可包含多个 `instrument_id`
- 解析规则：`product_id` 由 `instrument_id` 去掉末尾数字得到（如 `rb2610 -> rb`、`IF2606 -> IF`）

## 2. 关键业务约束（已定稿）

- 目标交易所：上海期货交易所（SHFE）、中国金融期货交易所（CFFEX）
- 关键交易接口：`vnpy_ctp`
- 交易日定义：`T日 = T-1 夜盘 + T日上午 + T日下午`
- 决策触发：每次 `factor_time_interval` 完成后由因子模块直接调用策略模块
- 决策频率：仅在“因子时间区间完成”时触发（`factor_time_interval` 可配置，如 `20s`、`30s`）
- 因子计算口径：因子由单个 `factor_time_interval` 区间内全部原始五档行情行聚合计算，因子表 `1` 行代表一个 `factor_time_interval` 结果
- 因子与策略内存窗口：运行期原始行情缓存仅保留未闭合区间数据；重启回看窗口固定 `5` 分钟（`raw_market_cache_seconds=300`）；决策流水线内存固定保留最近 `400` 行因子结果
- 缓存来源约束：重启场景下因子模块必须从数据库同时加载原始行情缓存与因子缓存；非重启场景下原始行情缓存由消费 topic 流实时构建，因子缓存由在线计算结果持续滚动保留
- 缓存淘汰约束：原始行情缓存必须记录“是否已参与因子计算”状态；已闭合区间在计算完成后应整窗淘汰，未参与计算的数据不得删除
- 合约切换约束：若次交易日生效后 `product_id` 的订阅合约集合发生变化，决策流水线必须清空该 `product_id` 的全部原始行情缓存、因子缓存与窗口游标，再重新累计到可决策状态
- 行情服务职责边界：仅负责行情接收与消息发布，不负责标准化、窗口聚合和数据库落库
- 行情主题约束：原始行情发布主题采用 `md.tick.raw.<product_id>`（如 `md.tick.raw.rb`、`md.tick.raw.IF`）
- 行情与因子落库职责：由独立归档服务分别消费 `md.tick.raw.*`、`md.tick.merged.*` 与 `factor.unit.calculated.<product_id>` 后写入 PostgreSQL
- 时间区间职责：`factor_time_interval` 闭合由因子模块维护并触发，策略模块通过进程内调用直接复用闭合结果
- 前置约束：必须满足“当前委托已结束（成交/撤单）+ 持仓新鲜 + 行情新鲜 + 风控通过”后，才允许新一轮策略
- 信号时效：策略结果带有效期，过期不得下发到执行层
- 超时委托：需定义超时时间，触发自动撤单
- 日终要求：交易日下午结束后调用外部结算接口拉取各品种全部合约结算数据，生成结算报告，并计算次交易日订阅合约（默认每个品种选择当日成交量前4主力合约）

## 3. 技术选型（已定稿）

- 交易接入：`vnpy_ctp`
- 消息中间件：`NATS JetStream`
- 关系数据库：`PostgreSQL`
- 实时缓存：决策链路默认使用进程内内存缓存；跨服务共享状态按需使用 `Redis`
- 归档介质：`NAS`（文件归档）
- 归档格式：行情以 `CSV` 为主
- 回放数据源：归档文件（不依赖 PostgreSQL）

明确不采用：

- `ClickHouse`
- `MinIO`

## 4. 总体逻辑架构

系统按职责分为 7 个域：

- 行情域：行情接入与原始事件发布
- 决策域：按 `product_id` 独立部署的决策流水线（因子模块 + 策略模块）、信号时效控制
- 交易域：下单、撤单、成交回报、委托状态机
- 风控域：事前拦截、事中监控、风险阈值告警
- 数据域：归档服务落库、PostgreSQL 全量存储、NAS 归档
- 结算与订阅规划域：外部接口结算同步、次交易日订阅计划生成
- 运维域：监控、报警、任务调度

## 5. 模块职责

### 5.1 行情采集服务（Market Collector）

- 代码来源：以 `vnpy` 行情接入能力为基础，统一接入 `vnpy_hft` 事件总线
- 通过 `vnpy_ctp` 接收行情
- 发布原始行情事件到 `md.tick.raw.<product_id>`
- 轻量化原则：仅做接收、编码、发布与订阅切换，不承载业务计算和存储
- 不负责行情标准化、时间区间聚合与数据库落库
- 支持次交易日按订阅计划热切换合约（不重启服务）

### 5.2 归档服务（Archive Service）

- 独立消费 `md.tick.raw.*` 原始行情事件、`md.tick.merged.*` 合并原始行事件与 `factor.unit.calculated.*`
- 负责行情标准化、因子归档转换与幂等处理
- 写入 PostgreSQL 行情表、合并原始行表与因子表（append-only）
- 执行实时库与历史库归档策略，输出 NAS 文件归档

### 5.3 因子服务（Factor Engine）

- 代码来源：主要移植 `MacroHFT_Features_SH` 因子工程代码，并改造为在线计算服务
- 物理部署：与策略模块按 `product_id` 同进程部署，组成 `Decision Pipeline`
- 因子计算口径：`factor_time_interval` 定义每个因子行需要多少秒原始数据参与计算，例如 `20s`、`30s`
- 两阶段链路：`Factor PreCalculator` 先将区间内原始行情合并为 `MergedRawUnitRow`，`Factor Calculator` 再基于合并结果计算最终因子行
- 语义基准：`Factor PreCalculator` 对齐 `step4_preprocess_order_files_v2.py` 的 `interval_to_seconds()` 与 `aggregate_by_minute()`
- 因子基准实现：`MacroHFT_Features_SH/src/gen/feature_calculator.py`（`calculate_all_features` / `get_feature_columns`）
- 品种因子清单：按 `product_id` 配置因子列表，仅输出该品种需要的因子列
- 重启恢复：从 `md_tick_l2_rt` 加载最近 `raw_market_cache_seconds` 原始行情，并从 `factor_unit_rt` 加载最近 `400` 行因子到内存缓存
- 非重启保留：因子缓存由在线计算结果持续保留；原始行情缓存仅保留未闭合区间并随闭合整窗淘汰
- 维护内存窗口：运行期“未闭合区间原始缓存 + 最近 `400` 行因子缓存”
- 缓存状态与淘汰：原始行情缓存按 `raw_tick -> factor_calculated(bool)` 维护状态，闭合区间计算完成后执行整窗淘汰
- 因子区间闭合：按分钟锚点切分区间（如 `20s -> 00/20/40/60`），每个区间计算时使用该区间内全部原始数据行
- 因子缓存顺序：按时间升序，最新行始终在末尾；超过 `400` 行淘汰最旧行
- 合约切换处理：当 `contract.plan.generated.<product_id>` 生效且合约集合变化时，执行全量缓存清空并进入预热期，预热完成前不触发策略
- 由因子服务自身维护 `factor_time_interval` 闭合，并在区间完成时计算因子
- 对外发布 `md.tick.merged.<product_id>`（合并后的原始行）供归档服务落库
- 对外发布 `factor.unit.calculated.<product_id>` 供归档服务落库
- 对内通过进程内内存调用直接触发策略模块计算

### 5.4 策略服务（Strategy Engine）

- 代码来源：主要移植 `ArchetypeTrader` 策略决策代码，并适配实盘调度与风控闸门
- 物理部署：与因子模块按 `product_id` 同进程部署，组成 `Decision Pipeline`
- 由因子模块在 `factor_time_interval` 闭合后通过内存直接调用
- 判定前检查闸门：无活动委托、持仓快照版本有效、行情新鲜、风控通过
- 直接读取因子模块内存中的最近 `400` 行因子缓存，不通过消息通道或 Redis 获取因子
- 输出策略信号（`strategy.signal.<product_id>`，含 `generated_at`、`expire_at`、`cycle_id`）
- 信号动作支持 `BUY/SELL/HOLD`；`HOLD` 也必须发布到消息主题

### 5.5 执行服务（OMS/Execution）

- 代码来源：以 `vnpy` 交易执行与委托回报机制为底座，在 `vnpy_hft` 中增强超时与状态管理
- 接收策略信号并校验有效期
- 发出委托指令（经 `vnpy_ctp`）
- 同一 `product_id` 存在未终态委托时阻断 `BUY/SELL` 新信号执行
- `HOLD` 信号按 no-op 处理，不触发新委托
- 记录下单事件
- 管理超时撤单
- 记录多次撤单尝试
- 记录委托状态变化、成交明细与持仓状态变化
- 活动委托持续超阈值未终态时触发管理员告警

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
- 按品种发布 `contract.plan.generated.<product_id>` 给行情采集与决策流水线（次交易日生效）

### 5.9 监控告警服务（Observability）

- 服务可用性、队列积压、处理延迟
- 行情延迟与数据新鲜度
- 归档服务落库延迟与失败率
- 下单成功率、撤单成功率、拒单率
- 风控触发频率与关键告警升级

## 6. 服务通信机制

### 6.1 异步事件总线（NATS JetStream）

建议主题（subject）：

- `md.tick.raw.<product_id>`：原始五档行情（按品种分片）
- `md.tick.merged.<product_id>`：按 `factor_time_interval` 合并后的原始行（按品种分片）
- `md.tick.archived`：行情归档写库完成事件
- `factor.unit.calculated.<product_id>`：因子结果归档事件
- `strategy.signal.<product_id>`：策略信号（`BUY/SELL/HOLD`，执行服务与归档服务共同消费）
- `order.submit.<product_id>`：下单请求
- `order.cancel.<product_id>`：撤单请求
- `order.event.<product_id>`：委托回报
- `trade.fill.<product_id>`：成交回报
- `risk.alert`：风险告警
- `system.alert`：系统告警
- `settlement.contract.synced`：外部结算接口同步完成
- `contract.volume.rank`：品种合约成交量排名
- `settlement.daily`：日终结算完成
- `contract.plan.generated.<product_id>`：夜盘/次日合约池（按品种分片）

### 6.2 同步查询通道

- 使用 gRPC/HTTP 提供状态查询：当前持仓、当前活动委托、风控参数、策略运行状态

### 6.3 幂等与顺序

- 语义：`at-least-once`
- 幂等键：`event_id`、`order_ref`、`trade_id`、`cycle_id`
- 顺序性建议：行情按 `instrument_id` 保序，交易按 `account_id` 保序
- 部署约束：决策流水线必须按 `product_id` 独立部署实例；每个实例内包含因子模块和策略模块，不同 `product_id` 不得混布在同一计算实例内

## 7. 核心业务机制

### 7.1 策略周期闸门

- 每次策略运行分配唯一 `cycle_id`
- 当前周期存在未终态委托时，不进入下一周期
- 若因闸门阻断而不进入交易执行，仍发布 `HOLD` 信号到 `strategy.signal.<product_id>`
- “终态”定义为：已成交完成或已撤单完成

### 7.2 信号时效控制

- 策略信号必须包含 `expire_at`
- 执行层在下单前检查时效
- 过期信号直接丢弃并告警

### 7.3 超时撤单

- 委托单配置 `timeout_ms`
- 超时后自动发起撤单
- 多次撤单均记录（不覆盖历史）
- 同一品种存在未终态委托时阻断 `BUY/SELL` 新信号执行
- 未终态持续超过 `active_order_stuck_alert_ms` 时告警升级到管理员

### 7.4 持仓参与策略

- 策略输入必须包含当前持仓
- 为避免竞态，持仓读取需与“活动委托状态”联动校验版本

### 7.5 因子口径与窗口约束

- `factor_time_interval` 表示生成 `1` 行因子需要多少秒原始数据，格式与离线脚本一致，使用如 `20s`、`30s`
- 作用定义：`factor_time_interval` 决定单行因子的时间覆盖范围、策略触发节奏以及归档主键粒度
- 每个 `factor_unit` 行对应一个 `factor_time_interval` 区间（`unit_start_ts` 到 `unit_end_ts`）
- 因子模块消费 `md.tick.raw.<product_id>` 并维护 `factor_time_interval` 闭合
- 在线计算阶段采用两阶段链路：`FactorWindowContext`（区间全部原始行） -> `Factor PreCalculator` 产出 `MergedRawUnitRow`（合并后的原始数据） -> `Factor Calculator` 产出最终因子行
- `MergedRawUnitRow` 需发布到 `md.tick.merged.<product_id>`，由归档服务写入 `merged_tick_l2_*`
- 策略模块不消费因子消息，而是复用因子模块的进程内因子缓存与闭合回调
- 内存窗口规则：运行期原始行情缓存仅保留未闭合区间数据；`raw_market_cache_seconds=300` 用于重启回看；因子缓存滚动保留最近 `400` 行结果
- 时间分块规则：`factor_time_interval_seconds` 必须整除 `60`，并按分钟锚点闭合区间
- 缓存来源规则：重启从 PostgreSQL 恢复；非重启从消费 topic 流持续保留
- 原始行情淘汰规则：闭合区间完成计算后对该区间执行整窗淘汰；未计算数据不得删除
- `MacroHFT_Features_SH/scripts/step4_preprocess_order_files_v2.py` 的 `interval` 聚合逻辑是当前离线口径基准；`vnpy_hft` 在线实现使用“原始行情缓存 + 因子缓存”的增量流式计算复现相同逻辑
- 因子列输出规则：按 `product_id` 的因子配置列表裁剪，且配置项必须为 `get_feature_columns()` 子集

### 7.6 日终结算与次交易日订阅

- 每日下午收盘后调度器调用外部结算接口，拉取“按品种全合约”结算与成交量
- 对每个品种按当日成交量降序排名
- 默认选取每个品种成交量前4的合约，作为次交易日订阅合约池
- 生成订阅计划后发布事件并持久化，供次交易日开盘前加载
- 行情采集服务在 `effective_trading_day` 到达时按 `cutover_time` 原子切换订阅合约
- 决策流水线同步消费 `contract.plan.generated.<product_id>`，若检测到 `product_id` 的合约集合变化则全量清空缓存并进入预热
- 预热完成定义：新合约集合内每个 `instrument_id` 的因子缓存行数达到 `factor_cache_rows` 后，才恢复该 `product_id` 的策略决策

## 8. 数据存储策略

### 8.1 PostgreSQL（主事实库）

- 保存全量数据
- 逻辑上 append-only（只插入，不更新）
- 行情数据与因子数据由归档服务写入；归档服务同时写入决策信号表 `strategy_signal_*`（当前优先落库 `BUY/SELL`，`HOLD` 后续扩展）；交易与结算数据由对应业务服务写入
- `merged_tick_l2_*` 作为合并原始行事实表，由归档服务消费 `md.tick.merged.*` 写入
- 执行服务直接写入交易执行主事实：`order_submit_*`、`order_state_event_*`、`cancel_request_*`、`trade_fill_*`、`position_snapshot_*`
- 采用 PostgreSQL 声明式分区（逻辑仍是一张表）

### 8.2 Redis（实时态）

- 仅按需存放跨服务共享的短期运行状态：活动委托集合、最新持仓快照、风控临时计数
- 不用于因子缓存，也不用于策略模块读取因子输入

### 8.3 NAS（归档）

- 由归档服务定期从 PostgreSQL 导出归档文件
- 行情优先导出 `CSV`
- 回放读取归档文件，而非 PostgreSQL

## 9. 性能目标（防过度设计）

- 调度频率：每秒一次触发检查
- 决策触发：`factor_time_interval` 完成后执行
- 时延目标：因子区间就绪到下单请求产出 `p99 <= 500ms`
- 执行目标：下单请求到交易接口回执 `p99 <= 1s`
- 稳定性目标：交易时段可用性 `>= 99.9%`

## 10. 监控与报警

### 10.1 数据质量

- 行情延迟
- 原始行情发布延迟（`md.tick.raw.<product_id>`）
- 合并原始行发布延迟（`md.tick.merged.<product_id>`）
- 归档服务入库延迟与失败率

### 10.2 交易链路

- 下单成功率
- 撤单成功率
- 拒单率
- 超时撤单触发次数
- 同品种信号阻断次数
- 活动委托超时未终态告警次数

### 10.3 系统健康

- 服务存活
- JetStream 消费堆积
- PostgreSQL 慢查询与连接池
- 可选 Redis 的内存与命中率

### 10.4 报警分级

- `Warning`：人工关注
- `Critical`：触发保护动作（限流、停策略、人工介入）

## 11. 部署建议

- 服务按功能独立部署，便于横向扩容
- 关键顺序通道按 `account_id` 分片，避免账户级竞态
- 行情服务按 `product_id` 优先拆分；决策流水线必须按 `product_id` 水平扩展
- 生产环境建议主备部署 PostgreSQL 与 NATS
- 行情服务默认部署规范：`1 collector instance -> 1 product_id -> 前4主力合约`
- 决策流水线默认部署规范：`1 decision-pipeline instance -> 1 product_id`
- 单个决策流水线实例内包含 `factor-engine` 与 `strategy-engine` 两个逻辑模块，并共享进程内因子缓存
- 例如 `AL` 与 `FU` 需要分别启动 `decision-pipeline-AL` 与 `decision-pipeline-FU`，分别订阅 `md.tick.raw.AL`、`md.tick.raw.FU`
- `vnpy` 技术上支持单个行情采集实例订阅多个品种多个合约，但生产环境不建议作为默认方案
- 多品种混合订阅仅适用于行情采集服务的开发、联调或低流量场景；决策流水线仍必须坚持单实例单 `product_id`

## 12. 配置项清单（建议纳入配置中心）

通用调度与策略：

- `strategy_trigger_mode=inprocess`：策略触发方式，固定为因子模块进程内回调。
- `factor_time_interval`：因子区间长度（如 `20s`、`30s`），决定触发节奏。
- `signal_ttl_ms`：信号有效期，超时不得下发。
- `market_freshness_threshold_ms`：行情新鲜度阈值。
- `position_freshness_threshold_ms`：持仓新鲜度阈值。

执行与风控：

- `order_timeout_ms`：委托超时阈值，超时触发撤单流程。
- `max_cancel_attempts`：单笔委托最大撤单重试次数。

因子与缓存：

- `raw_market_cache_seconds=300`：启动恢复回看窗口（秒）。
- `factor_cache_rows=400`：因子缓存保留行数。
- `factor_feature_list_by_product`：按品种配置输出因子列，示例 `{"al":["wap_balance"],"fu":["imbalance_top3"]}`。
- `factor_feature_list_strict=true`：严格校验因子列名，必须为 `get_feature_columns()` 子集。
- `factor_switch_reset_mode=full_flush`：合约切换时缓存重置策略。
- `raw_cache_evict_requires_calculated=true`：仅允许淘汰已参与计算的原始数据。

主题与分片：

- `md_subject_pattern=md.tick.raw.{product_id}`：原始行情主题模板。
- `md_merged_subject_pattern=md.tick.merged.{product_id}`：合并原始行主题模板。
- `factor_archive_subject_pattern=factor.unit.calculated.{product_id}`：因子归档主题模板。
- `contract_plan_subject_pattern=contract.plan.generated.{product_id}`：次交易日订阅计划主题模板。
- `md_product_shard_subscriptions`：行情服务按品种分片订阅配置。

行情采集：

- `market_collector_publish_buffer`：行情发布缓冲区大小。
- `market_collector_reconnect_backoff_ms`：断线重连退避时间。
- `subscription_cutover_time`：次交易日订阅切换时间点。

归档：

- `archive_writer_batch_rows`：归档批量写入行数。
- `archive_writer_batch_ms`：归档批量写入时间窗口。
- `archive_export_cron`：NAS 导出调度表达式。
- `archive_nas_path`：NAS 根目录路径。

结算与订阅规划：

- `settlement_run_time`：日终结算任务时间。
- `settlement_api_endpoint`：外部结算接口地址。
- `contract_selection_metric=volume`：次日主力合约筛选指标。
- `contract_selection_top_n=4`：每品种次日订阅合约数量。

去重说明：

- 已删除重复功能配置 `raw_market_cache_minutes`，统一使用 `raw_market_cache_seconds`。
- `factor_precalc_semantic`、`factor_precalc_interval_parser`、`factor_calculator_module` 属于实现基线，不作为运行时配置。
- `factor_warmup_min_units` 与 `factor_cache_rows` 目标重复，已删除；预热阈值统一使用 `factor_cache_rows`。
