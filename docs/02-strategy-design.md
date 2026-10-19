# 套利策略设计

> **本文档部分过时（2026-10）。** 权威版本是 [`07-final-design.md`](./07-final-design.md)。
> 已修正：全部浮点运算改为 BigInt（原第 183、280、296 行）；最优规模改为三分搜索。
> 未修正但已作废：本文只描述 V2 的 `x*y=k`。可交易的流动性几乎全在 Uniswap V3，
> 而 V3 的定价不能用储备量表达——见 [ADR-0002](./adr/0002-v3-primary.md)。

## 策略1: 三角套利 (Triangular Arbitrage)

### 原理
在同一DEX内,通过3个交易对形成闭环,利用价格差异套利。

### 路径示例
```
USDC → WETH → MATIC → USDC
100 USDC → 0.055 WETH → 105 MATIC → 102.5 USDC
净利润: 2.5 USDC (扣除Gas后)
```

### 计算公式 (Uniswap V2)

```typescript
// 单步swap输出计算
function getAmountOut(
  amountIn: bigint,
  reserveIn: bigint,
  reserveOut: bigint,
  feeBps: number = 30  // 30 basis points = 0.3%
): bigint {
  const amountInWithFee = amountIn * BigInt(10000 - feeBps);
  const numerator = amountInWithFee * reserveOut;
  const denominator = reserveIn * 10000n + amountInWithFee;
  return numerator / denominator;
}

// 三角套利完整计算
function calculateTriangularProfit(
  path: [Pool, Pool, Pool],
  amountIn: bigint
): bigint {
  // Step 1: USDC → WETH
  const amount1 = getAmountOut(
    amountIn,
    path[0].reserve0,  // USDC reserve
    path[0].reserve1,  // WETH reserve
    path[0].feeBps
  );
  
  // Step 2: WETH → MATIC
  const amount2 = getAmountOut(
    amount1,
    path[1].reserve0,  // WETH reserve
    path[1].reserve1,  // MATIC reserve
    path[1].feeBps
  );
  
  // Step 3: MATIC → USDC
  const amountOut = getAmountOut(
    amount2,
    path[2].reserve0,  // MATIC reserve
    path[2].reserve1,  // USDC reserve
    path[2].feeBps
  );
  
  return amountOut - amountIn;  // 毛利润 (未扣Gas)
}
```

### 最优输入金额计算

套利利润是非线性的,存在一个最优输入金额。

```typescript
// 固定路由下，毛利润对投入金额是单峰函数，用三分搜索。
// 不要用固定 20 步网格：精度不足，且可能整段错过峰值。
function findOptimalAmount(
  path: [Pool, Pool, Pool],
  minAmount: bigint,
  maxAmount: bigint
): { amount: bigint; profit: bigint } {
  let lo = minAmount;
  let hi = maxAmount;

  for (let i = 0; i < 70; i++) {
    const third = (hi - lo) / 3n;
    if (third === 0n) break;
    const m1 = lo + third;
    const m2 = hi - third;

    const p1 = calculateTriangularProfit(path, m1);
    const p2 = calculateTriangularProfit(path, m2);

    if (p1 < p2) lo = m1 + 1n;
    else hi = m2 - 1n;
  }

  let best = { amount: 0n, profit: 0n };
  const start = lo > 2n ? lo - 2n : 1n;
  for (let a = start; a <= hi + 2n; a++) {
    const p = calculateTriangularProfit(path, a);
    if (p > best.profit) best = { amount: a, profit: p };
  }
  return best;
}
```

### 路径发现算法

```typescript
function findAllTriangularPaths(
  pools: Pool[],
  startToken: string  // 例如 "USDC"
): Array<[Pool, Pool, Pool]> {
  const paths: Array<[Pool, Pool, Pool]> = [];
  
  // 暴力搜索所有3步路径
  for (const pool1 of pools) {
    if (!pool1.hasToken(startToken)) continue;
    const token1 = pool1.otherToken(startToken);
    
    for (const pool2 of pools) {
      if (pool2 === pool1) continue;
      if (!pool2.hasToken(token1)) continue;
      const token2 = pool2.otherToken(token1);
      
      for (const pool3 of pools) {
        if (pool3 === pool1 || pool3 === pool2) continue;
        if (!pool3.hasToken(token2)) continue;
        if (pool3.otherToken(token2) !== startToken) continue;
        
        // 找到一条有效路径
        paths.push([pool1, pool2, pool3]);
      }
    }
  }
  
  return paths;
}
```

## 策略2: 跨DEX套利 (Cross-DEX Arbitrage)

### 原理
同一交易对在不同DEX价格不同,低买高卖。

### 示例
```
QuickSwap: 1 WETH = 1800 USDC
Uniswap V3: 1 WETH = 1805 USDC

操作:
1. QuickSwap买入 1 WETH (花 1800 USDC)
2. Uniswap卖出 1 WETH (得 1805 USDC)
净利润: 5 USDC
```

### 计算

```typescript
function calculateCrossDexProfit(
  buyPool: Pool,   // 价格低的DEX
  sellPool: Pool,  // 价格高的DEX
  amountIn: bigint
): bigint {
  // 在buyPool买入token
  const tokenAmount = getAmountOut(
    amountIn,
    buyPool.reserve0,  // USDC
    buyPool.reserve1,  // WETH
    buyPool.feeBps
  );
  
  // 在sellPool卖出token
  const amountOut = getAmountOut(
    tokenAmount,
    sellPool.reserve1,  // WETH
    sellPool.reserve0,  // USDC
    sellPool.feeBps
  );
  
  return amountOut - amountIn;
}
```

### 机会发现

```typescript
function findCrossDexOpportunities(
  poolsByPair: Map<string, Pool[]>  // "USDC/WETH" -> [pool1, pool2, ...]
): Opportunity[] {
  const opportunities: Opportunity[] = [];
  
  for (const [pair, pools] of poolsByPair) {
    if (pools.length < 2) continue;
    
    // 计算所有pools的当前价格
    // 用交叉相乘比较，绝不做浮点除法
    const prices = pools.map(p => ({
      pool: p,
      priceNum: p.reserve1,
      priceDen: p.reserve0
    }));
    
    // 找到最低价和最高价
    prices.sort((a, b) => {
      const left = a.priceNum * b.priceDen;
      const right = b.priceNum * a.priceDen;
      return left < right ? -1 : left > right ? 1 : 0;
    });
    const lowest = prices[0];
    const highest = prices[prices.length - 1];
    
    // 计算价差百分比
    const spreadBps =
      ((highest.priceNum * lowest.priceDen - lowest.priceNum * highest.priceDen) * 10000n) /
      (lowest.priceNum * highest.priceDen);
    
    // 如果价差 > 30 基点，认为有套利机会
    if (spreadBps > 30n) {
      opportunities.push({
        type: 'cross-dex',
        buyPool: lowest.pool,
        sellPool: highest.pool,
        spreadBps
      });
    }
  }
  
  return opportunities;
}
```

## 决策流程

```typescript
interface Opportunity {
  type: 'triangular' | 'cross-dex';
  path: Pool[];
  amountIn: bigint;
  expectedProfit: bigint;
  gasEstimate: bigint;
  timestamp: number;
}

async function evaluateOpportunity(
  opp: Opportunity,
  config: Config
): Promise<'execute' | 'skip'> {
  // 1. 扣除Gas成本
  const gasPrice = await getGasPrice();
  const gasCost = opp.gasEstimate * gasPrice;
  const netProfit = opp.expectedProfit - gasCost;
  
  // 2. 检查最小利润阈值
  if (netProfit < config.minProfit) {
    return 'skip';  // 利润太少
  }
  
  // 3. 检查时效性 (价格可能已变)
  const age = Date.now() - opp.timestamp;
  if (age > 2000) {  // 超过2秒
    return 'skip';   // 太旧了,重新计算
  }
  
  // 4. 检查滑点风险
  const slippage = calculateSlippage(opp);
  if (slippage > config.maxSlippage) {
    return 'skip';  // 滑点太大
  }
  
  return 'execute';
}
```

## 性能优化

### 1. 路径缓存
```typescript
// 不需要每次都重新搜索路径
const pathCache = new Map<string, Array<[Pool, Pool, Pool]>>();

function getTriangularPaths(token: string): Array<[Pool, Pool, Pool]> {
  if (!pathCache.has(token)) {
    pathCache.set(token, findAllTriangularPaths(pools, token));
  }
  return pathCache.get(token)!;
}
```

### 2. 并行计算
```typescript
async function findBestOpportunity(): Promise<Opportunity | null> {
  const paths = getTriangularPaths('USDC');
  
  // 并行计算所有路径的利润
  const results = await Promise.all(
    paths.map(async path => {
      const optimal = findOptimalAmount(path, MIN_AMOUNT, MAX_AMOUNT);
      return { path, ...optimal };
    })
  );
  
  // 找到最佳机会
  results.sort((a, b) => (a.profit === b.profit ? 0 : a.profit < b.profit ? 1 : -1));
  return results[0].profit > 0n ? results[0] : null;
}
```

## 风险控制

### 滑点计算
```typescript
function calculateSlippage(opp: Opportunity): number {
  // 实际执行会推动价格
  // 简化模型: 滑点 ≈ amountIn / poolLiquidity
  const totalLiquidity = opp.path.reduce((sum, pool) => {
    return sum + pool.reserve0;
  }, 0);
  
  // 仅对 V2 成立。V3 池子的余额大部分停在价格区间之外，
  // 可用深度必须用 QuoterV2 在参考规模下实测（见 07-final-design.md 第 4 节）。
  const slippageBps = (opp.amountIn * 10000n) / totalLiquidity;
  return Number(slippageBps);
}
```

### 最大交易规模限制
```typescript
const MAX_TRADE_SIZE = parseUnits('50', 6);  // 50 USDC

if (opportunity.amountIn > MAX_TRADE_SIZE) {
  opportunity.amountIn = MAX_TRADE_SIZE;
  // 重新计算利润
}
```

