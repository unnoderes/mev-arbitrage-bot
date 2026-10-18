# MEV Arbitrage Bot - MVP

一个运行在 Polygon 链上的最小可用版本套利机器人，支持三角套利和跨DEX套利。

## 项目概述

**目标**
- 自动监控 Polygon 链上的价格差异
- 执行无风险套利交易（原子化执行）
- 提供 CLI + Web Dashboard 实时监控
- 初始资金要求：< $100 USDC

**技术栈**
- 语言：TypeScript
- 区块链交互：ethers.js v6
- 合约开发：Hardhat
- 前端：Next.js + WebSocket
- RPC：Alchemy 免费版
- 目标链：Polygon

## 系统架构

```
┌─────────────────────────────────────────────────────┐
│                   Web Dashboard                     │
│              (Next.js + WebSocket)                  │
└────────────────┬────────────────────────────────────┘
                 │ WebSocket 实时推送
                 ↓
┌─────────────────────────────────────────────────────┐
│                  Bot Core (Node.js)                 │
├─────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ Price Monitor│  │   Strategy   │  │ Executor │ │
│  │  (监听价格)   │→ │  (计算利润)   │→ │ (执行交易)│ │
│  └──────────────┘  └──────────────┘  └──────────┘ │
│         ↑                                   ↓       │
│  ┌──────────────┐                   ┌──────────┐  │
│  │ Pool Manager │                   │  Logger  │  │
│  │ (管理交易池)  │                   │ (记录日志)│  │
│  └──────────────┘                   └──────────┘  │
└─────────────────┬───────────────────────────────────┘
                  │ RPC 调用
                  ↓
┌─────────────────────────────────────────────────────┐
│              Polygon Network (via Alchemy)          │
├─────────────────────────────────────────────────────┤
│  QuickSwap  │  Uniswap V3  │  SushiSwap  │  Curve │
└─────────────────────────────────────────────────────┘
```

## 目录结构

```
mev-arbitrage-bot/
├── docs/                      # 完整设计文档
│   ├── 01-architecture.md     # 系统架构设计
│   ├── 02-strategy.md         # 套利策略详解
│   ├── 03-modules.md          # 模块详细设计
│   ├── 04-smart-contracts.md  # 智能合约设计
│   ├── 05-implementation.md   # 实施指南（给AI编码用）
│   └── 06-deployment.md       # 部署和运维
│
├── contracts/                 # 智能合约
│   ├── FlashArbitrage.sol     # 主套利合约
│   ├── interfaces/            # 接口定义
│   └── test/                  # 合约测试
│
├── src/                       # 核心代码
│   ├── core/                  # 核心模块
│   │   ├── PriceMonitor.ts    # 价格监听器
│   │   ├── PoolManager.ts     # 交易池管理
│   │   ├── Strategy.ts        # 套利策略
│   │   ├── Executor.ts        # 交易执行器
│   │   └── Logger.ts          # 日志系统
│   │
│   ├── config/                # 配置文件
│   │   ├── pools.json         # 监控的交易池
│   │   ├── tokens.json        # 代币地址
│   │   └── settings.json      # 系统参数
│   │
│   ├── utils/                 # 工具函数
│   │   ├── calculations.ts    # 价格计算
│   │   ├── formatters.ts      # 格式化工具
│   │   └── helpers.ts         # 辅助函数
│   │
│   ├── types/                 # TypeScript 类型定义
│   │   └── index.ts
│   │
│   └── index.ts               # 入口文件
│
├── dashboard/                 # Web Dashboard
│   ├── app/                   # Next.js App
│   ├── components/            # React 组件
│   ├── lib/                   # 前端工具
│   └── public/                # 静态资源
│
├── scripts/                   # 部署和维护脚本
│   ├── deploy.ts              # 部署合约
│   ├── setup.ts               # 初始化配置
│   └── backtest.ts            # 回测工具
│
├── logs/                      # 日志文件（运行时生成）
├── data/                      # 数据文件（运行时生成）
│
├── .env.example               # 环境变量模板
├── .gitignore
├── package.json
├── tsconfig.json
├── hardhat.config.ts
└── README.md
```

## 核心功能

### 1. 支持的套利策略

**三角套利（Triangular Arbitrage）**
- 在同一个 DEX 内，通过 3 个交易对循环套利
- 例：USDC → WETH → MATIC → USDC
- 风险：流动性、滑点

**跨DEX套利（Cross-DEX Arbitrage）**
- 同一交易对在不同 DEX 价格不同
- 例：QuickSwap买入 → Uniswap卖出
- 使用闪电贷实现零本金套利

### 2. 监控的交易池（初始版本）

**QuickSwap**
- USDC/WETH
- USDC/MATIC
- WETH/MATIC

**Uniswap V3 (Polygon)**
- USDC/WETH (0.05% fee)
- USDC/MATIC (0.05% fee)
- WETH/MATIC (0.05% fee)

**SushiSwap (Polygon)**
- USDC/WETH
- USDC/MATIC

### 3. Web Dashboard 功能

**实时监控面板**
- 机器人运行状态（运行中/已停止）
- 当前监控的区块高度
- 实时发现的套利机会
- 执行中的交易状态

**统计数据**
- 今日/本周/总计统计
- 发现机会数量
- 执行成功率
- 累计利润
- Gas 消耗

**历史记录**
- 最近 100 条套利机会
- 交易执行详情（成功/失败/原因）
- 可按时间/利润/类型筛选

**控制面板**
- 启动/停止机器人
- 调整参数（最小利润阈值、最大滑点）
- 手动触发测试交易

## 核心参数

```typescript
// 默认配置
const CONFIG = {
  // 最小利润阈值（USDC）
  minProfit: 2,
  
  // 最大滑点（basis points，100 = 1%）
  maxSlippage: 50, // 0.5%
  
  // Gas 价格倍数（1.0 = 标准，1.2 = +20%）
  gasPriceMultiplier: 1.1,
  
  // 最大单笔交易金额（USDC）
  maxTradeSize: 50,
  
  // 轮询间隔（毫秒）
  pollingInterval: 1000,
  
  // 是否启用闪电贷
  useFlashLoan: false, // MVP阶段先用自有资金
}
```

## 安全机制

1. **原子化交易**
   - 所有 swap 步骤在一个交易内完成
   - 要么全部成功，要么全部回滚
   - 不会出现卡在中间状态的情况

2. **利润验证**
   - 交易执行前验证预期利润
   - 合约内再次验证实际利润
   - 不盈利则 revert

3. **滑点保护**
   - 设置最大可接受滑点
   - 超过阈值则放弃交易

4. **资金隔离**
   - 合约只能由 owner 调用
   - 支持紧急提款

## 预期性能

**初始阶段（100 USDC 本金）**
- 每天发现机会：20-50 次
- 实际执行：5-15 次
- 成功率目标：> 70%
- 日均利润：$2-10
- Gas 成本：每次 $0.01-0.05

**优化后（3个月）**
- 每天执行：20-40 次
- 成功率：> 85%
- 日均利润：$10-30

## 快速开始

详见各文档模块。编码工作将由低成本模型执行，所有必要的技术细节和伪代码都已包含在 `docs/` 目录中。
