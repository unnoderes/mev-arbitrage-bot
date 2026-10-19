# 开发环境配置说明

> **⚠️ 本文档部分过时（2026-10）。** 当前设计见 [`docs/07-final-design.md`](docs/07-final-design.md)。
>
> - 配置里的 `mumbai`（chainId 80001）已关停，应改为 Amoy（80002）；
>   且 `hardhat.config.ts` 的 `forking.enabled` 目前是 `false`，fork 测试跑不起来。
> - `MIN_PROFIT_USDC` 默认值不是 2，而是 `max(0.02, 2 × Gas)`。
> - 只观测模式**不需要** `PRIVATE_KEY`。
> - `@uniswap/v2-periphery@1.1.0-beta.0` 是 0.5/0.6 时代的包，能否在 solc 0.8.20 下编译尚未核实。

## ✅ 环境初始化完成

本文档记录了项目的开发环境配置状态。

---

## 📦 已安装依赖

### 运行时依赖
- `ethers` ^6.9.0 - 以太坊交互库
- `dotenv` ^16.3.1 - 环境变量管理
- `ws` ^8.14.2 - WebSocket 支持

### 开发依赖
- `typescript` ^5.3.2 - TypeScript 编译器
- `ts-node` ^10.9.1 - TypeScript 运行时
- `hardhat` ^2.19.2 - 智能合约开发框架
- `@nomicfoundation/hardhat-toolbox` ^4.0.0 - Hardhat 工具集
- `@openzeppelin/contracts` ^5.0.1 - OpenZeppelin 合约库
- `@uniswap/v2-periphery` ^1.1.0-beta.0 - Uniswap V2 接口
- `@types/node` ^20.10.0 - Node.js 类型定义
- `@types/ws` ^8.5.10 - WebSocket 类型定义

**总计**: 630 个包已安装

---

## 🔧 配置文件

### TypeScript 配置 (`tsconfig.json`)
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "sourceMap": true
  }
}
```

### Hardhat 配置 (`hardhat.config.ts`)
```typescript
{
  solidity: "0.8.20",
  networks: {
    mumbai: { chainId: 80001 },
    polygon: { chainId: 137 }
  }
}
```

### 环境变量 (`.env`)
✅ 已创建（从 `.env.example` 复制）

**需要配置的变量**：
- `RPC_URL` - Alchemy/Infura RPC 节点
- `PRIVATE_KEY` - 钱包私钥
- `POLYGONSCAN_API_KEY` - 区块链浏览器 API
- `MIN_PROFIT_USDC` - 最小利润阈值（默认 2）
- `MAX_SLIPPAGE_BPS` - 最大滑点（默认 50）

---

## 📁 项目结构

```
mev-arbitrage-bot/
├── src/                    # TypeScript 源代码（待开发）
├── contracts/              # Solidity 智能合约（待开发）
├── test/                   # 测试文件（待开发）
├── scripts/                # 部署和维护脚本（待开发）
├── logs/                   # 日志文件（运行时生成）
├── data/                   # 数据文件（运行时生成）
├── docs/                   # 完整设计文档 ✅
├── node_modules/           # 依赖包 ✅
├── .env                    # 环境变量 ✅
├── tsconfig.json           # TypeScript 配置 ✅
├── hardhat.config.ts       # Hardhat 配置 ✅
├── package.json            # 项目配置 ✅
└── DEVELOPMENT_WORKFLOW.md # 开发工作流规范 ✅
```

---

## ✅ 验证结果

### Node.js 环境
- Node.js: `v22.12.0` ✅
- npm: `11.14.1` ✅

### 工具链
- TypeScript: `5.9.3` ✅
- Hardhat: `2.29.1` ✅

### 可用命令
```bash
npm run dev         # 运行 TypeScript 源码
npm run build       # 编译 TypeScript
npm run compile     # 编译智能合约
npm run test        # 运行测试
npm run deploy:mumbai   # 部署到测试网
npm run deploy:polygon  # 部署到主网
```

---

## 🚀 下一步

### 1. 配置环境变量
编辑 `.env` 文件，填入真实的配置：

```bash
# 获取 Alchemy API Key
# 1. 访问 https://www.alchemy.com/
# 2. 注册并创建 Polygon 应用
# 3. 复制 API Key 到 RPC_URL

# 钱包私钥
# ⚠️ 警告：请使用专门的开发钱包，不要使用主钱包
# 从 MetaMask 导出私钥
```

### 2. 开始开发
按照 `docs/04-implementation-guide.md` 开始编码：

```bash
# Phase 1: 智能合约
# 创建 contracts/FlashArbitrage.sol

# Phase 2: 核心模块
# 实现 src/core/PoolManager.ts
# 实现 src/core/PriceMonitor.ts
# 实现 src/core/Strategy.ts
# 实现 src/core/Executor.ts

# Phase 3: 配置文件
# 创建 src/config/pools.json
# 创建 src/config/tokens.json
```

### 3. 测试
```bash
# 运行合约测试
npm run test

# 编译合约
npm run compile

# 本地运行机器人（需要先实现代码）
npm run dev
```

---

## 📚 参考文档

- [开发工作流规范](./DEVELOPMENT_WORKFLOW.md)
- [项目概览](./docs/00-overview.md)
- [系统架构](./docs/01-architecture.md)
- [策略设计](./docs/02-strategy-design.md)
- [智能合约](./docs/03-smart-contract.md)
- [实施指南](./docs/04-implementation-guide.md)
- [快速开始](./docs/05-quickstart.md)

---

## ⚠️ 注意事项

1. **私钥安全**
   - `.env` 文件已添加到 `.gitignore`
   - 永远不要提交私钥到 Git
   - 使用专门的开发钱包

2. **测试网优先**
   - Mumbai 已关停，改用 Amoy（chainId 80002）
   - 但 Amoy 只能验证「部署流程能跑通」，策略验证必须在 Polygon 主网 fork 上做

3. **小额资金**
   - 初始建议使用 $50-100 USDC
   - 验证策略可行后再增加资金

4. **Gas 费用**
   - Polygon Gas 费较低（$0.01-0.05/交易）
   - 但仍需监控 Gas 价格波动

---

**环境初始化完成时间**: 2026-10-18  
**初始化版本**: v1.0  
**Node.js 版本**: v22.12.0  
**依赖包总数**: 630
