# 实施指南 - 给AI编码模型

> **⚠️ 不要按本文档编码（2026-10）。**
> 权威设计见 [`07-final-design.md`](./07-final-design.md)，术语见 [`GLOSSARY.md`](../GLOSSARY.md)，
> 决策理由见 [`docs/adr/`](./adr/)。
>
> 本文中的类型与接口已作废，原因：
> 1. `Pool` 用 `reserves` 描述状态 —— 只对 V2 成立，而可交易的流动性几乎全在 V3；
> 2. `Config` 里的 `minProfit = 2`、`maxTradeSize = 50`、`pollingInterval` 均已重新定义；
> 3. 模块切法（`PoolManager / PriceMonitor / Executor / Logger`）已被 `venues / registry / strategy / observe / chain` 取代；
> 4. `src/config/pools.json` 的硬编码池子清单已改为从工厂枚举 + 深度下限过滤。
>
> 本文保留作为历史记录。

本文档为低成本AI模型(如GPT-3.5, Claude Haiku)提供完整的代码实施指令。

## 项目初始化

### 1. 创建基础目录结构

```bash
mkdir -p contracts/interfaces contracts/test
mkdir -p src/core src/config src/utils src/types  
mkdir -p dashboard/app dashboard/components
mkdir -p scripts logs data
```

### 2. 初始化 package.json

```json
{
  "name": "mev-arbitrage-bot",
  "version": "1.0.0",
  "scripts": {
    "start": "ts-node src/index.ts",
    "dev": "nodemon --exec ts-node src/index.ts",
    "build": "tsc",
    "test": "hardhat test",
    "deploy": "hardhat run scripts/deploy.ts --network polygon"
  },
  "dependencies": {
    "ethers": "^6.9.0",
    "dotenv": "^16.0.0",
    "ws": "^8.14.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/ws": "^8.5.0",
    "typescript": "^5.0.0",
    "ts-node": "^10.9.0",
    "nodemon": "^3.0.0",
    "hardhat": "^2.19.0",
    "@nomicfoundation/hardhat-toolbox": "^4.0.0"
  }
}
```

### 3. TypeScript配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 4. Hardhat配置

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import * as dotenv from "dotenv";

dotenv.config();

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    polygon: {
      url: process.env.POLYGON_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : []
    },
    mumbai: {
      url: process.env.MUMBAI_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : []
    }
  },
  etherscan: {
    apiKey: process.env.POLYGONSCAN_API_KEY
  }
};

export default config;
```

## 核心模块实现

### 模块 1: 类型定义 (src/types/index.ts)

```typescript
export interface Token {
  symbol: string;
  address: string;
  decimals: number;
}

export interface Pool {
  id: string;
  dex: 'quickswap' | 'uniswap-v3' | 'sushiswap';
  address: string;
  token0: Token;
  token1: Token;
  fee: number;  // basis points
  router: string;
  reserves?: {
    reserve0: bigint;
    reserve1: bigint;
    blockNumber: number;
  };
}

export interface Opportunity {
  type: 'triangular' | 'cross-dex';
  path: Pool[];
  amountIn: bigint;
  expectedProfit: bigint;
  gasEstimate: bigint;
  timestamp: number;
}

export interface Config {
  rpcUrl: string;
  privateKey: string;
  contractAddress: string;
  minProfit: bigint;
  maxSlippage: number;
  gasPriceMultiplier: number;
  maxTradeSize: bigint;
  pollingInterval: number;
}
```

### 模块 2: 配置文件 (src/config/pools.json)

```json
{
  "quickswap": {
    "router": "0xa5E0829CaCEd8fFDD4De3c43696c57F7D7A678ff",
    "pools": [
      {
        "id": "quickswap-usdc-weth",
        "address": "0x...",
        "token0": {
          "symbol": "USDC",
          "address": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
          "decimals": 6
        },
        "token1": {
          "symbol": "WETH",
          "address": "0x7ceB23fD6bC0adD59E62ac25578270cFf1b9f619",
          "decimals": 18
        },
        "fee": 30
      }
    ]
  },
  "uniswap-v3": {
    "router": "0xE592427A0AEce92De3Edee1F18E0157C05861564",
    "pools": []
  }
}
```

### 模块 3: PoolManager (src/core/PoolManager.ts)

```typescript
import { ethers } from 'ethers';
import { Pool, Token } from '../types';
import poolsConfig from '../config/pools.json';

const PAIR_ABI = [
  'function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast)'
];

export class PoolManager {
  private pools: Map<string, Pool> = new Map();
  private provider: ethers.Provider;
  
  constructor(provider: ethers.Provider) {
    this.provider = provider;
    this.loadPools();
  }
  
  private loadPools() {
    for (const [dex, data] of Object.entries(poolsConfig)) {
      for (const poolData of data.pools) {
        const pool: Pool = {
          id: poolData.id,
          dex: dex as any,
          address: poolData.address,
          token0: poolData.token0,
          token1: poolData.token1,
          fee: poolData.fee,
          router: data.router
        };
        this.pools.set(pool.id, pool);
      }
    }
    console.log(`✅ 加载了 ${this.pools.size} 个交易池`);
  }
  
  async updateReserves(poolIds: string[]): Promise<void> {
    const updates = await Promise.all(
      poolIds.map(id => this.fetchReserves(id))
    );
    
    updates.forEach(({ poolId, reserves }) => {
      const pool = this.pools.get(poolId);
      if (pool) {
        pool.reserves = reserves;
      }
    });
  }
  
  private async fetchReserves(poolId: string) {
    const pool = this.pools.get(poolId);
    if (!pool) throw new Error(`Pool ${poolId} not found`);
    
    const contract = new ethers.Contract(pool.address, PAIR_ABI, this.provider);
    const [reserve0, reserve1, _] = await contract.getReserves();
    
    return {
      poolId,
      reserves: {
        reserve0: BigInt(reserve0.toString()),
        reserve1: BigInt(reserve1.toString()),
        blockNumber: await this.provider.getBlockNumber()
      }
    };
  }
  
  getPool(id: string): Pool | undefined {
    return this.pools.get(id);
  }
  
  getAllPools(): Pool[] {
    return Array.from(this.pools.values());
  }
  
  getPoolsByToken(tokenSymbol: string): Pool[] {
    return this.getAllPools().filter(p => 
      p.token0.symbol === tokenSymbol || p.token1.symbol === tokenSymbol
    );
  }
}
```

### 模块 4: Strategy (src/core/Strategy.ts)

```typescript
import { Pool, Opportunity } from '../types';

export class Strategy {
  // 计算Uniswap V2风格的swap输出
  static getAmountOut(
    amountIn: bigint,
    reserveIn: bigint,
    reserveOut: bigint,
    feeBps: number = 30
  ): bigint {
    const amountInWithFee = amountIn * BigInt(10000 - feeBps);
    const numerator = amountInWithFee * reserveOut;
    const denominator = reserveIn * 10000n + amountInWithFee;
    return numerator / denominator;
  }
  
  // 三角套利利润计算
  static calculateTriangularProfit(
    path: [Pool, Pool, Pool],
    amountIn: bigint
  ): bigint {
    let amount = amountIn;
    
    for (const pool of path) {
      if (!pool.reserves) return 0n;
      
      amount = this.getAmountOut(
        amount,
        pool.reserves.reserve0,
        pool.reserves.reserve1,
        pool.fee
      );
    }
    
    return amount - amountIn;
  }
  
  // 找到最优输入金额
  static findOptimalAmount(
    path: [Pool, Pool, Pool],
    minAmount: bigint,
    maxAmount: bigint,
    steps: number = 20
  ): { amount: bigint; profit: bigint } {
    let best = { amount: 0n, profit: 0n };
    const stepSize = (maxAmount - minAmount) / BigInt(steps);
    
    for (let i = 0n; i <= BigInt(steps); i++) {
      const testAmount = minAmount + stepSize * i;
      const profit = this.calculateTriangularProfit(path, testAmount);
      
      if (profit > best.profit) {
        best = { amount: testAmount, profit };
      }
    }
    
    return best;
  }
  
  // 查找所有三角套利路径
  static findTriangularPaths(
    pools: Pool[],
    startToken: string
  ): Array<[Pool, Pool, Pool]> {
    const paths: Array<[Pool, Pool, Pool]> = [];
    
    for (const pool1 of pools) {
      const token1 = this.getOtherToken(pool1, startToken);
      if (!token1) continue;
      
      for (const pool2 of pools) {
        if (pool2.id === pool1.id) continue;
        const token2 = this.getOtherToken(pool2, token1);
        if (!token2) continue;
        
        for (const pool3 of pools) {
          if (pool3.id === pool1.id || pool3.id === pool2.id) continue;
          const token3 = this.getOtherToken(pool3, token2);
          if (token3 === startToken) {
            paths.push([pool1, pool2, pool3]);
          }
        }
      }
    }
    
