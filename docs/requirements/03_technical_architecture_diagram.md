# 技术架构图（独立版）

本版以“先总览、再时序”为主，优先可读性。

## 项目来源说明（四项目整合）

- `vnpy_hft` 为统一集成项目，承接其余 3 个项目能力
- `vnpy` 提供交易系统底座（`vnpy_ctp` 接入、事件驱动、执行链路）
- `MacroHFT_Features_SH` 提供因子工程能力（训练数据口径与实时特征计算逻辑）
- `ArchetypeTrader` 提供策略研究与决策能力（信号生成与策略判定逻辑）

## 图 1：系统总览（分层）

```mermaid
flowchart LR
    subgraph L1[接入层]
        EX[交易所/期货公司]
        API[外部结算接口]
    end

    subgraph L2[核心服务层]
        MC[行情采集]
        FE[因子计算]
        SE[策略决策]
        RS[风险控制]
        OMS[交易执行]
        SS[日终结算]
        CSP[次日订阅规划]
        OBS[监控告警]
    end

    subgraph L3[基础设施层]
        MQ[(NATS JetStream)]
        PG[(PostgreSQL)]
        RD[(Redis)]
        NAS[(NAS 归档)]
    end

    EX --> MC
    MQ --> FE
    MQ --> SE
    MQ --> RS
    MQ --> OMS

    MC --> MQ
    FE --> MQ
    SE --> MQ
    RS --> MQ

    OMS --> EX
    OMS --> RD
    SE --> RD

    MC --> PG
    FE --> PG
    OMS --> PG
    SS --> PG
    CSP --> PG

    SS --> API
    API --> SS
    SS --> CSP
    CSP --> MQ

    PG --> NAS
    OBS --> MQ
```

## 图 2：日内主链路（时序）

```mermaid
sequenceDiagram
    autonumber
    participant EX as 交易所/期货公司
    participant MC as 行情采集
    participant MQ as NATS
    participant FE as 因子计算
    participant SE as 策略决策
    participant RS as 风险控制
    participant OMS as 交易执行
    participant PG as PostgreSQL
    participant RD as Redis

    EX->>MC: 推送 1 秒五档行情
    MC->>MQ: 发布 md.tick.raw / md.unit.window.close
    MC->>PG: 写入 md_tick_l2

    MQ->>FE: 消费单位行情
    FE->>PG: 写入 factor_unit(1行=x秒)
    FE->>MQ: 发布 factor.unit.calculated

    MQ->>SE: 触发策略判定
    SE->>RD: 读取持仓与活动委托状态
    SE->>MQ: 发布 strategy.signal(含 expire_at)

    MQ->>RS: 事前风控检查
    RS->>MQ: 发布 risk.alert 或放行

    MQ->>OMS: 接收下单/撤单指令
    OMS->>EX: 委托/撤单
    OMS->>PG: 写入 order_submit/cancel_request/trade_fill
    OMS->>RD: 更新活动委托状态
```

## 图 3：日终结算与次交易日订阅（时序）

```mermaid
sequenceDiagram
    autonumber
    participant SCH as 日终调度
    participant SS as 日终结算服务
    participant API as 外部结算接口
    participant CSP as 次日订阅规划
    participant PG as PostgreSQL
    participant MQ as NATS
    participant MC as 行情采集
    participant SE as 策略决策

    SCH->>SS: 收盘后触发结算任务
    SS->>API: 拉取按品种全合约结算+成交量
    API-->>SS: 返回结算快照

    SS->>PG: 写入 daily_settlement
    SS->>PG: 写入 contract_settlement_snapshot
    SS->>CSP: 触发次交易日订阅规划

    CSP->>PG: 按品种成交量排序并落库 next_day_subscription_plan
    CSP->>MQ: 发布 contract.plan.generated

    MQ-->>MC: 下发次交易日订阅计划
    MQ-->>SE: 下发次交易日订阅计划
```

## 说明

- 因子口径：`1` 行因子 = `x(unit_seconds)` 秒行情结果。
- 内存窗口：固定 `400` 秒五档行情 + 对应 `ceil(400/x)` 行因子。
- 默认订阅规则：次交易日每个品种选当日成交量最高合约。
