# 系统架构设计

> **本文档已过时。** 最新的模块边界、数据模型与安全不变量见
> [`07-final-design.md`](./07-final-design.md)。

## 为什么旧的五模块切法被放弃

旧架构把系统切成 `PriceMonitor / PoolManager / Strategy / Executor / Logger`。
这套切法隐含了一个假设：所有池子都是 V2 风格、用「储备量」就能完全描述。
上链核实后这个假设不成立：可交易的流动性几乎全在 Uniswap V3 的集中流动性池里，
而 V3 的定价状态是 `slot0` 的价格与 tick，不是储备量。

新的切法按两条轴划分：**场所类型**（`venues/`）与 **观测/执行**（`observe/`、`chain/`），
另加 `registry/`（工厂枚举 + 深度下限）与 `strategy/`（闭环枚举 + 最优规模）。

详见 [`07-final-design.md` 第 10 节](./07-final-design.md)。

