# 期货交易系统数据库表设计（PostgreSQL）

## 1. 设计目标与硬约束

- 数据库：`PostgreSQL`
- 数据模式：append-only（业务逻辑只插入，不更新）
- 表空间：分为两个
- `ts_realtime`：实时库，仅保留最近 2 个交易日数据
- `ts_history`：历史库，保留全量历史数据
- 高频核心表采用“实时表 + 历史表”双表设计
- 日终归档：每天下午收盘后执行，从实时库归档到历史库
- 不使用 `bar` 数据表，行情基准仅为 `md_tick_l2`
- 因子由 `tick_l2` 聚合得到（`1` 行因子 = `1` 个 `factor_time_interval` 区间结果）
- 行情服务（Market Collector）不直接写库，仅发布 `md.tick.raw.<product_id>`
- 归档服务（Archive Service）消费 `md.tick.raw.*` 并写入 `md_tick_l2_*`
- 因子服务发布 `factor.unit.calculated.<product_id>`，归档服务消费后写入 `factor_unit_*`
- `factor_time_interval` 区间闭合由因子服务维护，策略模块通过进程内调用直接复用闭合结果

## 2. 表空间与命名规范

## 2.1 表空间

- `ts_realtime`：低延迟存储，服务日内策略与执行
- `ts_history`：大容量存储，服务历史查询、审计、归档导出

## 2.2 表命名（双表）

对以下高频表，统一采用双表后缀：

- 实时表：`*_rt`
- 历史表：`*_his`

示例：

- `md_tick_l2_rt` / `md_tick_l2_his`
- `factor_unit_rt` / `factor_unit_his`
- `order_submit_rt` / `order_submit_his`
- `cancel_request_rt` / `cancel_request_his`
- `trade_fill_rt` / `trade_fill_his`

## 2.3 时间与主键约定

- `trading_day`：交易日，类型建议 `DATE`（按国内期货交易日口径）
- `ts`：交易所原始时间
- `rev_ts`：系统真实接收时间
- 主键统一 `id BIGSERIAL`
- 统一 `created_ts TIMESTAMPTZ DEFAULT now()`

## 3. 高频双表清单（实时 + 历史）

- `md_tick_l2_rt` / `md_tick_l2_his`：秒级五档行情
- `factor_unit_rt` / `factor_unit_his`：按 `factor_time_interval` 聚合的因子结果
- `order_submit_rt` / `order_submit_his`：下单记录
- `cancel_request_rt` / `cancel_request_his`：撤单记录（多次撤单多行）
- `trade_fill_rt` / `trade_fill_his`：成交记录

说明：

- 上述 5 组表同构（字段一致、约束一致）
- 实时表用于在线读取和写入
- 历史表用于全量长期保存
- 各表写入由对应业务服务负责（归档/执行/结算等），非统一由行情服务写入

## 4. 高频双表结构（同构定义）

## 4.1 行情表：`md_tick_l2_rt` / `md_tick_l2_his`

用途：

- 全量保存秒级五档行情
- 因子输入的唯一行情来源（不依赖 bar）
- 写入主体：归档服务（消费 `md.tick.raw.*`）

建议字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | BIGSERIAL | 主键 |
| trading_day | DATE NOT NULL | 交易日 |
| exchange | VARCHAR(8) NOT NULL | 交易所（SHFE/CFFEX） |
| product_id | VARCHAR(16) NOT NULL | 品种 |
| instrument_id | VARCHAR(32) NOT NULL | 合约 |
| ts | TIMESTAMPTZ NOT NULL | 行情时间 |
| rev_ts | TIMESTAMPTZ NOT NULL | 系统接收时间 |
| last_price | NUMERIC(18,8) | 最新价 |
| volume | BIGINT | 累计成交量 |
| turnover | NUMERIC(20,4) | 累计成交额 |
| open_interest | NUMERIC(20,4) | 持仓量 |
| bid_price1..bid_price5 | NUMERIC(18,8) | 买一到买五价 |
| bid_volume1..bid_volume5 | BIGINT | 买一到买五量 |
| ask_price1..ask_price5 | NUMERIC(18,8) | 卖一到卖五价 |
| ask_volume1..ask_volume5 | BIGINT | 卖一到卖五量 |
| source | VARCHAR(32) NOT NULL DEFAULT 'vnpy_ctp' | 数据来源 |
| ingest_id | UUID | 幂等键（可选） |
| created_ts | TIMESTAMPTZ NOT NULL DEFAULT now() | 入库时间 |

约束和索引建议：

- 主键：`(id, trading_day)`（分区表建议）
- 可选唯一键：`UNIQUE (ingest_id)`
- 索引：`(trading_day, instrument_id, ts)`、`(rev_ts)`

## 4.2 因子表：`factor_unit_rt` / `factor_unit_his`

用途：

- 因子结果表，`1` 行 = `1` 个 `factor_time_interval` 区间结果
- 数据来源为 `md_tick_l2_*`，不是 bar 数据
- 写入主体：归档服务（消费 `factor.unit.calculated.*`）

建议字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | BIGSERIAL | 主键 |
| trading_day | DATE NOT NULL | 交易日 |
| exchange | VARCHAR(8) NOT NULL | 交易所 |
| product_id | VARCHAR(16) NOT NULL | 品种 |
| instrument_id | VARCHAR(32) NOT NULL | 合约 |
| factor_time_interval | VARCHAR(16) NOT NULL | 因子时间区间，如 `20s`、`30s` |
| unit_start_ts | TIMESTAMPTZ NOT NULL | 因子时间区间开始时间 |
| unit_end_ts | TIMESTAMPTZ NOT NULL | 因子时间区间结束时间 |
| rev_ts | TIMESTAMPTZ NOT NULL | 计算时最新接收时间 |
| factor_set | VARCHAR(64) NOT NULL DEFAULT 'default' | 因子集合 |
| factor_version | VARCHAR(32) NOT NULL | 因子版本 |
| calc_rev | INTEGER NOT NULL DEFAULT 1 | 修订号（append-only） |
| factors_json | JSONB NOT NULL | 因子值 |
| created_ts | TIMESTAMPTZ NOT NULL DEFAULT now() | 入库时间 |

约束和索引建议：

- 唯一键：`UNIQUE (instrument_id, factor_time_interval, unit_end_ts, factor_set, calc_rev)`
- 索引：`(trading_day, instrument_id, unit_end_ts)`、`(factor_set, factor_time_interval, unit_end_ts)`
- 可选：`GIN (factors_json)`（仅在 JSON 条件检索需要时）

口径规则：

- `factor_time_interval` 表示每行因子使用多少秒原始数据进行聚合
- 每行因子使用 `[unit_start_ts, unit_end_ts)` 范围内的 `tick_l2` 聚合计算

## 4.3 下单表：`order_submit_rt` / `order_submit_his`

建议字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | BIGSERIAL | 主键 |
| trading_day | DATE NOT NULL | 交易日 |
| account_id | VARCHAR(32) NOT NULL | 账户 |
| strategy_id | VARCHAR(64) NOT NULL | 策略标识 |
| cycle_id | VARCHAR(64) NOT NULL | 策略周期 |
| exchange | VARCHAR(8) NOT NULL | 交易所 |
| instrument_id | VARCHAR(32) NOT NULL | 合约 |
| direction | VARCHAR(8) NOT NULL | 买卖方向 |
| offset | VARCHAR(8) NOT NULL | 开平 |
| order_type | VARCHAR(16) NOT NULL | 委托类型 |
| tif | VARCHAR(16) | 时效类型 |
| price | NUMERIC(18,8) NOT NULL | 委托价格 |
| volume | INTEGER NOT NULL | 委托数量 |
| submit_ts | TIMESTAMPTZ NOT NULL | 策略下单时间 |
| send_ts | TIMESTAMPTZ | 网关发送时间 |
| rev_ts | TIMESTAMPTZ | 回报接收时间 |
| front_id | INTEGER | CTP front_id |
| session_id | INTEGER | CTP session_id |
| order_ref | VARCHAR(32) | CTP order_ref |
| order_sys_id | VARCHAR(32) | 交易所系统单号 |
| timeout_ms | INTEGER | 超时撤单阈值 |
| created_ts | TIMESTAMPTZ NOT NULL DEFAULT now() | 入库时间 |

约束和索引建议：

- 唯一键：`UNIQUE (account_id, front_id, session_id, order_ref)`
- 索引：`(trading_day, account_id, instrument_id)`、`(cycle_id)`、`(order_sys_id)`

## 4.4 撤单表：`cancel_request_rt` / `cancel_request_his`

建议字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | BIGSERIAL | 主键 |
| trading_day | DATE NOT NULL | 交易日 |
| account_id | VARCHAR(32) NOT NULL | 账户 |
| order_submit_id | BIGINT NOT NULL | 关联下单ID |
| attempt_no | INTEGER NOT NULL | 第几次撤单 |
| cancel_reason | VARCHAR(128) | 撤单原因 |
| request_ts | TIMESTAMPTZ NOT NULL | 发起撤单时间 |
| send_ts | TIMESTAMPTZ | 发送到网关时间 |
| rev_ts | TIMESTAMPTZ | 回报接收时间 |
| front_id | INTEGER | CTP front_id |
| session_id | INTEGER | CTP session_id |
| order_ref | VARCHAR(32) | CTP order_ref |
| order_sys_id | VARCHAR(32) | 交易所系统单号 |
| result_code | VARCHAR(32) | 结果码 |
| result_msg | VARCHAR(256) | 结果描述 |
| created_ts | TIMESTAMPTZ NOT NULL DEFAULT now() | 入库时间 |

约束和索引建议：

- 唯一键：`UNIQUE (order_submit_id, attempt_no)`
- 索引：`(trading_day, account_id)`、`(order_submit_id, request_ts)`

## 4.5 成交表：`trade_fill_rt` / `trade_fill_his`

建议字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | BIGSERIAL | 主键 |
| trading_day | DATE NOT NULL | 交易日 |
| account_id | VARCHAR(32) NOT NULL | 账户 |
| order_submit_id | BIGINT NOT NULL | 关联下单ID |
| exchange | VARCHAR(8) NOT NULL | 交易所 |
| instrument_id | VARCHAR(32) NOT NULL | 合约 |
| trade_id | VARCHAR(64) NOT NULL | 成交编号 |
| order_sys_id | VARCHAR(32) | 交易所系统单号 |
| direction | VARCHAR(8) NOT NULL | 买卖方向 |
| offset | VARCHAR(8) NOT NULL | 开平 |
| price | NUMERIC(18,8) NOT NULL | 成交价 |
| volume | INTEGER NOT NULL | 成交量 |
| trade_ts | TIMESTAMPTZ NOT NULL | 成交时间 |
| rev_ts | TIMESTAMPTZ NOT NULL | 接收时间 |
| commission | NUMERIC(18,8) | 手续费 |
| created_ts | TIMESTAMPTZ NOT NULL DEFAULT now() | 入库时间 |

约束和索引建议：

- 唯一键：`UNIQUE (account_id, trade_id)`
- 索引：`(trading_day, account_id, instrument_id, trade_ts)`、`(order_submit_id)`

## 5. 低频表（仅历史表空间）

以下表直接建在 `ts_history`，不做实时/历史双表：

- `daily_settlement`
- `contract_settlement_snapshot`
- `next_day_subscription_plan`

说明：`next_day_subscription_plan` 默认保存“每个品种成交量前4主力合约”的次交易日订阅池（可配置）。

原因：

- 写入频率低
- 查询以历史为主
- 无需日内高并发低延迟读写

## 6. 日终归档与实时保留策略

## 6.1 每日收盘归档流程

收盘后执行归档任务（建议按交易日粒度）：

1. 将 `T` 日数据从 `*_rt` 复制到 `*_his`
2. 逐表做行数校验、可选校验和校验
3. 校验通过后标记归档完成
4. 清理实时表中过期数据，仅保留最近 2 个交易日（`T` 和 `T-1`）

说明：

- 归档失败时不执行清理
- 归档任务需可重试且幂等
- 推荐使用 `INSERT INTO ... SELECT ... ON CONFLICT DO NOTHING`
- 归档流程由归档服务（或其调度任务）执行

## 6.2 保留策略

- 实时库：仅保留最近 `2` 个交易日
- 历史库：全量永久保留（除非手动归档策略变更）

## 7. 文件归档与回放

- 每日收盘后，从 `md_tick_l2_his` 导出 `CSV(.zst)` 到 NAS
- 回放仅依赖归档文件，不依赖 PostgreSQL
- 本系统不需要 bar 文件，回放输入为 `tick_l2` 原始行情

建议目录：

- `/nas/archive/md_tick_l2/trading_day=YYYYMMDD/exchange=SHFE/instrument_id=rbXXXX.csv.zst`

建议清单文件（manifest）字段：

- `row_count`
- `sha256`
- `schema_version`
- `export_ts`

## 8. 分区与索引建议

- `*_rt`：按 `trading_day` 日分区（便于 2 日滚动清理）
- `*_his`：按 `trading_day` 月分区（平衡分区数量与查询效率）
- 提前创建未来分区（建议 7-30 天）
- 保持最小必要索引，避免写入放大

## 9. 数据一致性与幂等键

- 下单锚点：`account_id + front_id + session_id + order_ref`
- 成交锚点：`account_id + trade_id`
- 因子锚点：`instrument_id + factor_time_interval + unit_end_ts + factor_set + calc_rev`
- 入库幂等：`ON CONFLICT DO NOTHING`

## 10. DDL 落地顺序建议

1. 创建表空间 `ts_realtime`、`ts_history`
2. 创建 5 组高频双表（`*_rt`、`*_his`）与分区模板
3. 创建低频历史表（结算、结算快照、订阅计划）
4. 创建索引和唯一约束
5. 上线日终归档任务与校验任务
6. 上线实时库 2 日保留清理任务
7. 上线 NAS 导出与回放读取链路
