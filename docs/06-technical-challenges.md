# 技术难点和解决方案

本文档列出开发过程中会遇到的**真正棘手的技术问题**及其解决方案。

## 目录

1. [大整数运算精度丢失](#1-大整数运算精度丢失) 🔴 高风险
2. [价格时效性问题](#2-价格时效性问题) 🔴 高风险
3. [Uniswap V2 vs V3 差异](#3-uniswap-v2-vs-v3-差异) 🟡 中风险
4. [Token Approve 陷阱](#4-token-approve-陷阱) 🟡 中风险
5. [WebSocket 断线重连](#5-websocket-断线重连) 🟡 中风险
6. [交易 Nonce 冲突](#6-交易-nonce-冲突) 🟢 低风险
7. [Gas 估算不准确](#7-gas-估算不准确) 🟢 低风险

---

## 1. 大整数运算精度丢失 🔴

### 问题描述

Solidity 和 JavaScript 的 bigint 在连续计算时容易出现精度丢失或溢出。

**错误示例**:
```typescript
// ❌ 危险 - 可能溢出或精度丢失
const result = (amountIn * 997n * reserveOut) / 
               (reserveIn * 1000n + amountIn * 997n);
```

**问题**:
- 中间结果可能超过 2^256 导致溢出
- 整数除法会向下取整，连续 3 次 swap 误差累积
- 链下计算利润 $5，实际执行只有 $2，或者亏损

### 解决方案

#### 方案 1: 调整计算顺序 (推荐)

```typescript
// ✅ 正确 - 避免中间溢出
function getAmountOut(
  amountIn: bigint,
  reserveIn: bigint,
  reserveOut: bigint,
  feeBps: number = 30  // 30 basis points = 0.3%
): bigint {
  // 检查输入
  if (amountIn === 0n || reserveIn === 0n || reserveOut === 0n) {
    return 0n;
  }
  
  // 分步计算，避免一次性大数乘法
  const feeMultiplier = BigInt(10000 - feeBps);
  const amountInWithFee = amountIn * feeMultiplier;
  
  // 检查是否会溢出 (简化检查)
  if (amountInWithFee < amountIn) {
    throw new Error('Overflow in fee calculation');
  }
  
  const numerator = amountInWithFee * reserveOut;
  const denominator = reserveIn * 10000n + amountInWithFee;
  
  if (denominator === 0n) return 0n;
  
  return numerator / denominator;
}
```

#### 方案 2: 使用安全边际

```typescript
// 在所有利润计算中留 5-10% 安全边际
function calculateSafeProfit(
  path: [Pool, Pool, Pool],
  amountIn: bigint
): bigint {
  const theoreticalProfit = calculateTriangularProfit(path, amountIn);
  
  // 留 10% 安全边际，应对精度损失和价格滑点
  const safeProfit = theoreticalProfit * 90n / 100n;
  
  return safeProfit;
}
```

#### 方案 3: 链上最终验证

```solidity
// 合约内必须验证实际利润
function triangularArbitrage(
    address[3] calldata routers,
    address[4] calldata tokens,
    uint256 amountIn
) external onlyOwner nonReentrant {
    uint256 balanceBefore = IERC20(tokens[0]).balanceOf(address(this));
    
    // 执行三步swap...
    
    uint256 balanceAfter = IERC20(tokens[0]).balanceOf(address(this));
    
    // 最终验证：必须盈利
    require(balanceAfter > balanceBefore, "No profit");
}
```

### 测试建议

```typescript
// 单元测试：验证边界情况
describe('getAmountOut', () => {
  it('应该处理大数运算', () => {
    const amountIn = parseUnits('1000000', 6);  // 1M USDC
    const reserve0 = parseUnits('10000000', 6);  // 10M
    const reserve1 = parseUnits('5000', 18);     // 5000 WETH
    
    const result = getAmountOut(amountIn, reserve0, reserve1);
    expect(result).to.be.gt(0);
  });
  
  it('应该处理三次swap的累积误差', () => {
    // 模拟 USDC → WETH → MATIC → USDC
    const initial = parseUnits('100', 6);
    const final = simulateTriangularPath(path, initial);
    
    // 验证误差 < 1%
    const error = abs(final - expected) * 100n / expected;
    expect(error).to.be.lt(100n);  // < 1%
  });
});
```

---

## 2. 价格时效性问题 🔴

### 问题描述

从查询储备量到交易上链，通常需要 1-3 秒，期间价格可能已经变化。

**时间线**:
```
T0: 查询 reserves，计算利润 $5
T1: 构建交易 calldata
T2: 发送交易 (价格可能已变)
T3: 矿工打包 (价格可能再次变化)
T4: 执行合约 (价格可能第三次变化)
```

**实际影响**:
- 70% 的交易会因为 "No profit" 而 revert
- 每次失败都浪费 Gas 费 ($0.01-0.05)

### 解决方案

#### 方案 1: 发送前二次确认 (必须)

```typescript
async function execute(opportunity: Opportunity): Promise<ExecutionResult> {
  // 第一次检查：发现机会时
  if (opportunity.expectedProfit < CONFIG.minProfit) {
    return { success: false, reason: 'Insufficient profit' };
  }
  
  // 延迟检查：发送前再次查询当前价格
  console.log('⏱️  价格二次确认中...');
  
  const currentReserves = await Promise.all(
    opportunity.path.map(pool => fetchPoolReserves(pool.address))
  );
  
  // 用最新储备量重新计算利润
  const currentProfit = recalculateProfit(
    opportunity.path,
    currentReserves,
    opportunity.amountIn
  );
  
  const gasCost = await estimateGasCost();
  const netProfit = currentProfit - gasCost;
  
  // 如果仍然有利可图才发送
  if (netProfit < CONFIG.minProfit) {
    console.log('❌ 价格已变化，机会失效');
    return { success: false, reason: 'Price changed' };
  }
  
  console.log(`✅ 确认有利可图: ${formatUnits(netProfit, 6)} USDC`);
  
  // 发送交易
  return await sendTransaction(opportunity);
}
```

#### 方案 2: 设置合理的滑点容忍度

```typescript
// 配置
const CONFIG = {
  minProfit: parseUnits('2', 6),        // 最小利润 2 USDC
  slippageTolerance: 500,                // 5% (basis points)
  priceCheckMaxAge: 2000,                // 价格检查最大有效期 2秒
};

// 在计算时考虑滑点
function calculateProfitWithSlippage(
  path: [Pool, Pool, Pool],
  amountIn: bigint
): bigint {
  const theoreticalProfit = calculateTriangularProfit(path, amountIn);
  
  // 减去预期滑点
  const slippageAmount = theoreticalProfit * BigInt(CONFIG.slippageTolerance) / 10000n;
  const profitAfterSlippage = theoreticalProfit - slippageAmount;
  
  return profitAfterSlippage;
}
```

#### 方案 3: 链上最终检查 (必须)

```solidity
// 合约内验证利润，永远不能省略
function triangularArbitrage(
    address[3] calldata routers,
    address[4] calldata tokens,
    uint256 amountIn
) external onlyOwner nonReentrant {
    uint256 balanceBefore = IERC20(tokens[0]).balanceOf(address(this));
    require(balanceBefore >= amountIn, "Insufficient balance");
    
    // 执行swap...
    _executeSwaps(routers, tokens, amountIn);
    
    uint256 balanceAfter = IERC20(tokens[0]).balanceOf(address(this));
    
    // 必须验证：实际余额增加
    require(balanceAfter > balanceBefore, "No profit");
    
    emit ArbitrageExecuted(balanceAfter - balanceBefore, msg.sender);
}
```

#### 方案 4: 记录和分析失效率

```typescript
// 统计机会失效的原因
class OpportunityTracker {
  private stats = {
    discovered: 0,
    priceChanged: 0,
    gaseTooHigh: 0,
    executed: 0,
    reverted: 0
  };
  
  recordDiscovered() { this.stats.discovered++; }
  recordPriceChanged() { this.stats.priceChanged++; }
  recordGasTooHigh() { this.stats.gaseTooHigh++; }
  recordExecuted() { this.stats.executed++; }
  recordReverted() { this.stats.reverted++; }
  
  getEffectiveness(): number {
    // 有效执行率
    return this.stats.executed / this.stats.discovered * 100;
  }
  
  printStats() {
    console.log('📊 机会统计:');
    console.log(`  发现: ${this.stats.discovered}`);
    console.log(`  价格变化: ${this.stats.priceChanged} (${this.pct('priceChanged')}%)`);
    console.log(`  Gas过高: ${this.stats.gaseTooHigh} (${this.pct('gaseTooHigh')}%)`);
    console.log(`  已执行: ${this.stats.executed} (${this.pct('executed')}%)`);
    console.log(`  已回滚: ${this.stats.reverted} (${this.pct('reverted')}%)`);
    console.log(`  有效率: ${this.getEffectiveness().toFixed(1)}%`);
  }
  
  private pct(key: keyof typeof this.stats): string {
    return (this.stats[key] / this.stats.discovered * 100).toFixed(1);
  }
}
```

---

## 3. Uniswap V2 vs V3 差异 🟡

### 问题描述

Uniswap V3 使用**集中流动性**，与 V2 的恒定乘积公式完全不同。

**V2 (简单)**:
```typescript
// x * y = k
amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)
```

**V3 (复杂)**:
- 流动性分布在不同的价格区间 (tick)
- 每个 tick 有独立的流动性
- Swap 可能跨越多个 tick，需要分段计算
- 手续费有多档: 0.01%, 0.05%, 0.3%, 1%

**实际影响**:
- 用 V2 公式计算 V3 价格会**完全错误**
- Uniswap V3 占 Polygon 交易量的 40%+，不支持会错失大量机会

### 解决方案

#### 方案 1: MVP 阶段跳过 V3 —— ❌ 已被推翻，不要采用

> 上链核实后这个方案不成立：SushiSwap 的 USDC/WETH 池只有 0.08 USDC 与 0 WETH，
> QuickSwap 的 USDC/WPOL 池只有 671 USDC，而 Uniswap V3 的 0.05% 档在 USDC 侧深 11 倍。
> 「跳过 V3」等于「没有可交易的池子」。改用方案 2，并见 [ADR-0002](./adr/0002-v3-primary.md)。

```typescript
// 只监控 V2 风格的 DEX
const SUPPORTED_DEXES = {
  quickswap: {
    type: 'v2',
    router: '0xa5E0829CaCEd8fFDD4De3c43696c57F7D7A678ff'
  },
  sushiswap: {
    type: 'v2',
    router: '0x1b02dA8Cb0d097eB8D57A175b88c7D8b47997506'
  },
  // 暂不支持
  // uniswap_v3: { type: 'v3', ... }
};

// 加载池子时过滤
function loadPools() {
  return Object.entries(poolsConfig)
    .filter(([dex, config]) => config.type === 'v2')
    .flatMap(([dex, config]) => config.pools);
}
```

#### 方案 2: 使用 Quoter 合约查询 V3 价格

```typescript
// 如果必须支持 V3，使用链上 Quoter
const QUOTER_ABI = [
  // QuoterV2 的入参是结构体，返回值也不止 amountOut
  'function quoteExactInputSingle((address tokenIn, address tokenOut, uint256 amountIn, uint24 fee, uint160 sqrtPriceLimitX96)) external returns (uint256 amountOut, uint160 sqrtPriceX96After, uint32 initializedTicksCrossed, uint256 gasEstimate)'
];

async function getV3AmountOut(
  pool: PoolV3,
  amountIn: bigint
): Promise<bigint> {
  const quoter = new ethers.Contract(
    UNISWAP_V3_QUOTER,  // Polygon QuoterV2: 0x61fFE014bA17989E743c5F6cB21bF9697530B21e
                        // 旧地址 0xb27308f9... 是主网 Quoter V1，Polygon 上不可用
    QUOTER_ABI,
    provider
  );
  
  try {
    // callStatic 不会真正执行，只返回结果
    const amountOut = await quoter.callStatic.quoteExactInputSingle(
      pool.token0.add
