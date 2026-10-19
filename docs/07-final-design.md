# 最终设计（v2.0）

本文件取代 `docs/01` 至 `docs/06` 中与它冲突的内容。术语以 [`GLOSSARY.md`](../GLOSSARY.md) 为准，决策理由见 [`docs/adr/`](./adr/)。

---

## 1. 范围

本期交付一台**只观测**的测量仪器：每个区块读取受监控池子的价格快照、枚举闭环路由、算出最优投入金额与毛利润、按门槛判定，并对通过门槛的机会在下一区块状态上模拟真实执行。**不发送任何交易，不读取私钥。**

不做：Dashboard、实盘执行、私有交易通道、多链。

这样定义范围的原因见 [ADR-0004](./adr/0004-observe-only-and-negative-result.md)。

---

## 2. 已上链核实的事实

这些数字不是估计值，是查询 Polygon 主网得到的。实现时不要用文档里的旧假设覆盖它们。

### 2.1 代币

| 代币 | 地址 | 小数位 | 备注 |
|---|---|---|---|
| USDC（原生） | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` | 6 | |
| USDCe（桥接） | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | 6 | **`symbol()` 也返回 `"USDC"`** |
| WETH | `0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619` | 18 | |
| WPOL | `0x0d500B1d8E8eF31E21C99d1Db9A6444d3ADf1270` | 18 | **`symbol()` 返回 `"WPOL"`，不是 WMATIC** |

因为 USDC 与 USDCe 的 symbol 和 decimals 完全相同，**代币校验必须按地址，不能按 symbol**。

### 2.2 合约地址

| 用途 | 地址 | 核实结果 |
|---|---|---|
| Uniswap V3 Factory | `0x1F98431c8aD98523631AE4a59f267346ea31F984` | 有代码 |
| SwapRouter02 | `0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45` | 有代码；`exactInputSingle`(0x04e45aaf)、`exactInput`、`multicall` 选择器均在 |
| QuoterV2 | `0x61fFE014bA17989E743c5F6cB21bF9697530B21e` | 有代码，报价可用 |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` | 有代码，`aggregate3`(0x82ad56cb) 可用 |
| Balancer V2 Vault | `0xBA12222222228d8Ba445958a75a0704d566BF2C8` | 有代码，闪电贷可用 |
| Balancer ProtocolFeesCollector | `0xce88686553686DA562CE7Cea497CE749DA109f9F` | 闪电贷费率返回 `0` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` | 有代码 |
| UniversalRouter（常见引用地址） | `0x643770E279d5D0733F21d6DC03A9bABf5aD65FA6` | **无代码**，不要使用 |

### 2.3 Balancer Vault 的闪电贷上限

余额随时变动，运行时读取。核实时的量级：34,636 USDC、540,762 USDCe、234.9 WETH、796,070 WPOL。

### 2.4 成本

`baseFeePerGas` 实测 192.9 gwei（不是早期文档假设的 30 gwei）。按 400k gas 估算，一笔交易约 0.0772 POL。POL 的美元价格从快照里的 WPOL/USDC 池子折算，实测约 0.093 美元。

---

## 3. 场所抽象

路由的每一步都是一个「场所类型 + 定位参数」：

```solidity
struct Step {
    uint8  kind;      // 0 = V2 型, 1 = V3 型
    address target;   // kind=0: router 地址；kind=1: 不使用
    address tokenIn;
    address tokenOut;
    uint24  fee;      // kind=1: 费率档（100/500/3000/10000）；kind=0: 不使用
}
```

- **V2 型**一律拆成单跳，`tokenOut` 必须是 router 那条 path 的终点。不允许多跳 path，因为逐跳校验代币衔接是安全前提。
- **V3 型**由 `(tokenIn, tokenOut, fee)` 定位池子，通过 `SwapRouter02.exactInputSingle` 执行。

---

## 4. 注册表与深度下限

池子不硬编码。启动时从 V3 Factory 的 `getPool(tokenA, tokenB, fee)` 枚举白名单代币的全部组合（5 个交易对 × 4 个费率档），再按下述下限过滤：

**深度下限：在 1,000 个计价代币的参考规模下，可执行价格相对边际价格的偏离不超过 50 个基点。**

- 用 QuoterV2 实测，不用余额、也不用 `L`。
- 每 100 个区块重测一次。
- 被挡掉的池子不进入策略计算。

这条下限不只是降噪，它承担着正确性职责：流动性有缺口的池子，其 `slot0` 边际价格与真实成交价可以差 1.6%（实测：USDCe→WETH 的 0.01% 档，边际价 0.00041110 WETH，1 单位真实成交 0.00040450 WETH）。用边际价判定机会会持续产生假阳性。

---

## 5. 路由枚举与最优规模

- 闭环定义：从计价代币出发、经过 1 至 3 跳、回到同一计价代币。
- 计价代币：USDC 与 USDCe 两个独立账本，互不折算。
- 中间代币：WETH、WPOL。
- 枚举全部费率档组合。

**最优规模用三分搜索**（固定路由下，毛利润对投入金额单峰）：

```
lo = 1, hi = 上限（由深度与 Balancer 可借量取小）
重复约 70 次：m1 = lo + (hi-lo)/3, m2 = hi - (hi-lo)/3
  若 profit(m1) < profit(m2) 则 lo = m1+1，否则 hi = m2-1
在 [lo-2, hi+2] 内线性确认最优
```

不要用固定 20 步网格：精度不足且可能整段错过峰值。

---

## 6. 门槛与成本

```
net_profit = gross_profit − gas_cost_in_numeraire − flash_loan_fee
gate       = max(0.02 计价单位, 2 × gas_cost_in_numeraire)
执行条件   = net_profit ≥ gate
```

- `flash_loan_fee` 当前为 0（Balancer），但保留这个字段，以便换成 Aave 时不用改结构。
- `gas_cost_in_numeraire` = 固定跳数成本表 × 当前 gasPrice，再乘以快照里得到的 POL/计价代币汇率。**不引入外部价格源。**
- 固定成本表（初值，实施时用 fork 的 `estimateGas` 校准）：2 跳 ≈ 280k，3 跳 ≈ 380k，再加闪电贷开销 ≈ 80k。

---

## 7. 合约接口与安全不变量

```solidity
function execute(
    Step[] calldata steps,
    uint256 amountIn,
    uint256 minProfitNumeraire,
    bool    useFlashLoan
) external;
```

**唯一的安全不变量**：

> 记录执行前的计价代币余额 `B0`；执行全部步骤并**归还闪电贷之后**，要求 `balanceOf(numeraire) ≥ B0 + max(minProfitNumeraire, ownerMinProfit)`，否则回滚。

配套要求：

1. **无 `onlyOwner`**。任何人可调用；利润留在合约，owner 用 `withdraw` 提取。理由见 [ADR-0003](./adr/0003-permissionless-executor.md)。
2. **双白名单**：router 与代币都要白名单。否则「任何人可调用」等于「任何人可任意调用」。
3. **逐跳 `minOut`**：由调用者提供，合约逐跳校验。否则中间步骤被夹击时最终余额仍可能达标。
4. **闪电贷回调**：只接受 `msg.sender == Balancer Vault`，且只在 `execute()` 执行期间（用一个 `bool` 标记）。**不要用 `nonReentrant` 修饰 `execute()`**——它会挡住 Balancer 的正常回调。
5. **两步路径**：`useFlashLoan = false` 时使用合约自有余额，供 fork 测试与 Balancer 不可用时降级。

---

## 8. 只观测模式

每个区块：

1. 1 次 `Multicall3.aggregate3` 批量读取所有受监控池子的 `slot0`（V3）或 `getReserves`（V2）。
2. 用边际乘积做廉价筛选，得到候选闭环。
3. 对前 3 条候选，用 QuoterV2 做精确报价（每区块 ≤ 3 次）。
4. 三分搜索最优规模，算净利，过门槛。
5. 对过门槛的机会，在**下一个区块**状态上对真实 `execute()` 做 `eth_call` 模拟（每区块 ≤ 1 次）。

**「本会赢」的定义**：该模拟通过。这是胜率的**上界**，不是胜率——它假设我们能在下一个区块被包含，且不建模竞争与排序。输出里必须按上界表述。

---

## 9. 日志与 CSV 口径

每行一条记录：

```
block, route_id, venue_path, amount_in, gross_profit,
gas_cost_numeraire, flash_fee, net_profit,
gate_passed, sim_result, skip_reason
```

`skip_reason` 是枚举：`below_gate` / `stale` / `depth_floor` / `sim_reverted` / `depeg_guard`。

没有 `skip_reason`，跑一周只会得到「零利润」，无法区分「没有机会」和「机会被门槛挡掉」。

---

## 10. 模块边界

```
src/
├── chain/      provider、multicall、重试与退避、区块订阅
├── venues/     V2 与 V3 的读取与编码（slot0/getReserves、exactInputSingle 的 calldata）
├── registry/   工厂枚举、深度下限、白名单
├── strategy/   闭环枚举、最优规模、门槛判定
└── observe/    快照、probe、CSV 输出、模拟
contracts/      FlashArbitrage.sol 与接口
```

不采用旧的 `PoolManager / PriceMonitor / Executor / Logger` 切法：那个切法把 V3 的读取逻辑塞进「池子管理」里，是最容易出错的地方。

---

## 11. probe

```
npm run probe -- --from <block> --to <block>     # 默认最近 2000 个区块
```

输出：

- 每区块：最佳毛利润、对应路由、对应规模
- 汇总：p50 / p95 / p99 / 最大值，按路由分组的命中次数
- 落盘 CSV，同时打印前 10 条

判据：**≥ 2000 个连续区块上 p95 最佳毛利润低于门槛**，结论是「这批场所与交易对不可套利」，下一步换场所或换交易对，而不是继续等待。

---

## 12. 实施前核实结果

| 项 | 状态 | 证据 |
|---|---|---|
| `@uniswap/v2-periphery@1.1.0-beta.0` 能否在 solc 0.8.20 下编译 | ✅ 能 | 实装 1.1.0-beta.0 并用 solc 0.8.20+commit.a1b79de6 编译一个 import 了 `IUniswapV2Router02` 的合约：通过，bytecode 1,073 字节，仅 SPDX 警告，无错误。接口的 pragma 是 `>=0.6.2`，允许 0.8.20 |
| 能否用免费 RPC 做 Polygon 主网 fork | ✅ 能（不依赖 Alchemy） | hardhat 2.22.17 对 `https://polygon.drpc.org` 在区块 95,800,000 fork 成功：读到真实 `slot0`，池子代码 22,142 字节 |
| Amoy 可用 / Mumbai 已死 | ✅ 确认 | `polygon-amoy-bor-rpc.publicnode.com` 返回 chainId `0x13882`（80002）；Mumbai 的公共 RPC 无响应 |
| `.env.example` 仍是旧参数 | ✅ 确认 | 含 `MUMBAI_RPC_URL`、`MIN_PROFIT_USDC=2`、`MAX_TRADE_SIZE_USDC=50`、`ENABLE_WEB_DASHBOARD=true` |
| Alchemy 免费版具体配额 | ⚠️ 未验证（无 API key） | 但依赖已解除：fork 与历史读取都可用免费公共 RPC 完成。参考：Alchemy 的 `eth_call` = 26 CU，probe 2000 区块约 10,000 次 `eth_call` ≈ 260,000 CU |

### 12.1 fork 的必要配置（实测踩到）

Hardhat 对 Polygon 缺少内置的 hardfork 历史，直接 fork 会报：

```
No known hardfork for execution on historical block <n> in chain with id 137.
The node was not configured with a hardfork activation history.
```

必须在 `networks.hardhat` 下显式声明（注意 `paris` 不是合法值，要用 `merge`）：

```js
chains: {
  137: { hardforkHistory: { london: 0, merge: 0, shanghai: 0 } },
}
```

### 12.2 历史状态深度与限流

- `polygon.drpc.org` 能对至少 **1,000,000 区块前**的状态执行 `eth_call`，2000 区块的 probe 不需要归档专用节点。
- `polygon-bor-rpc.publicnode.com` 在突发请求下会返回 403。probe 必须带重试与退避，并且支持多端点轮换——否则一次限流会让整个观测窗口中断。

### 12.3 仍需在实现阶段做的事

- `hardhat.config.ts` 仍指向 `mumbai` 且 `forking.enabled = false`，需要改成 Amoy 并打开 fork（按 12.1 配好 hardforkHistory）。
- `.env.example` 的参数需要随实现一并调整；本期只观测，不需要 `PRIVATE_KEY`，也不需要 Dashboard。

---

## 13. 已知局限

- 当前 4 个代币、15 个 V3 池子在任何规模下都不可套利（30 区块全枚举实测）。扩交易对是下一期。
- 「边际为正」的闭环每区块有 32 条，全部是薄池子假象。深度下限是唯一能挡住它们的机制。
- 模拟只能给出胜率上界。
- 不做私有交易通道；Polygon 上是否有 Flashbots 等价物尚未调研。
- V3 报价依赖 QuoterV2，配额受限；三分搜索只能用近似模型，精确报价只用于确认。
