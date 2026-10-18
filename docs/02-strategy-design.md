# 套利策略设计

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
function findOptimalAmount(
  path: [Pool, Pool, Pool],
  minAmount: bigint,
  maxAmount: bigint,
  steps: number = 20
): { amount: bigint; profit: bigint } {
  let best = { amount: 0n, profit: 0n };
  const step = (maxAmount - minAmount) / BigInt(steps);
  
  for (let i = 0; i <= steps; i++) {
    const testAmount = minAmount + step * BigInt(i);
    const profit = calculateTriangularProfit(path, testAmount);
    
    if (profit > best.profit) {
      best = { amount: testAmount, profit };
    }
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
    const prices = pools.map(p => ({
      pool: p,
      price: Number(p.reserve1) / Number(p.reserve0)
    }));
    
    // 找到最低价和最高价
    prices.sort((a, b) => a.price - b.price);
    const lowest = prices[0];
    const highest = prices[prices.length - 1];
    
    // 计算价差百分比
    const spreadPct = (highest.price - lowest.price) / lowest.price * 100;
    
    // 如果价差 > 0.3%,认为有套利机会
    if (spreadPct > 0.3) {
      opportunities.push({
        type: 'cross-dex',
        buyPool: lowest.pool,
        sellPool: highest.pool,
        spread: spreadPct
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
  results.sort((a, b) => Number(b.profit - a.profit));
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
    return sum + Number(pool.reserve0);
  }, 0);
  
  const slippagePct = Number(opp.amountIn) / totalLiquidity * 100;
  return slippagePct;
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

