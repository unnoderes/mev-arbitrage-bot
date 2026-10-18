# 快速开始指南

## 第一步: 环境准备

### 1. 安装 Node.js
确保安装了 Node.js 18+ 和 npm

```bash
node --version  # 应该 >= v18.0.0
npm --version
```

### 2. 获取 Alchemy API Key

1. 访问 https://alchemy.com
2. 注册账号
3. 创建一个 Polygon 应用
4. 复制 HTTPS RPC URL

### 3. 准备钱包

```bash
# 生成新钱包 (或使用现有的)
# 确保钱包有一些 MATIC (用于 Gas) 和 USDC (用于套利)
```

最小资金要求:
- **MATIC**: 1-2 个 (~$1-2, 用于Gas)
- **USDC**: 50-100 个 (套利本金)

## 第二步: 项目设置

### 1. 克隆/初始化项目

```bash
cd ~/Desktop/mev-arbitrage-bot
npm install
```

### 2. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env`:
```env
# RPC节点
RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/YOUR_API_KEY
WS_RPC_URL=wss://polygon-mainnet.g.alchemy.com/v2/YOUR_API_KEY

# 钱包私钥 (不要泄露!)
PRIVATE_KEY=0xabcd1234...

# 合约地址 (部署后填写)
CONTRACT_ADDRESS=

# Polygonscan API Key (用于验证合约)
POLYGONSCAN_API_KEY=
```

### 3. 编译合约

```bash
npx hardhat compile
```

## 第三步: 部署合约

### 测试网部署 (Mumbai - 推荐先测试)

```bash
# 1. 从水龙头获取测试MATIC
# https://mumbaifaucet.com

# 2. 部署合约
npx hardhat run scripts/deploy.ts --network mumbai

# 输出示例:
# ✅ 合约已部署: 0x1234...
# 复制这个地址到 .env 的 CONTRACT_ADDRESS
```

### 主网部署 (Polygon)

```bash
# 确保钱包有 0.1 MATIC 用于部署Gas
npx hardhat run scripts/deploy.ts --network polygon
```

### 验证合约 (可选但推荐)

```bash
npx hardhat verify --network polygon YOUR_CONTRACT_ADDRESS
```

## 第四步: 给合约转入资金

```bash
# 方法1: 通过Metamask转账
# 打开Metamask → 发送 50 USDC 到合约地址

# 方法2: 使用脚本
npx hardhat run scripts/fund-contract.ts --network polygon
```

验证余额:
```bash
npx hardhat run scripts/check-balance.ts --network polygon
# 应该显示: USDC Balance: 50.0
```

## 第五步: 启动机器人

### 开发模式 (带热重载)

```bash
npm run dev
```

### 生产模式

```bash
npm start
```

预期输出:
```
🚀 MEV Arbitrage Bot 启动中...

✅ 加载了 9 个交易池
🔍 启动价格监听...
📊 Block 12345678 - 检查套利机会...
📊 Block 12345679 - 检查套利机会...
💰 发现机会! 预期利润: 3.45 USDC
⚡ 执行套利: triangular
📤 交易已发送: 0xabcd...
✅ 交易确认! Gas used: 245621
```

## 第六步: 监控和优化

### 查看日志

```bash
tail -f logs/bot-$(date +%Y-%m-%d).log
```

### 查看统计

```bash
npm run stats
```

输出示例:
```
📊 今日统计:
  发现机会: 47 次
  执行成功: 12 次
  成功率: 92.3%
  总利润: $28.45
  Gas成本: $1.23
  净利润: $27.22
```

### 调整参数

编辑 `src/config/settings.json`:
```json
{
  "minProfit": 2,        // 提高这个值 = 更保守
  "maxSlippage": 50,     // 降低这个值 = 更安全
  "maxTradeSize": 50     // 根据资金调整
}
```

重启机器人后生效。

## 常见问题

### Q1: 机器人一直没发现机会?

**原因**:
1. 市场暂时没有套利机会 (正常情况,可能需要等几小时)
2. 配置的最小利润阈值太高
3. 监控的交易池流动性不足

**解决**:
```typescript
// 临时降低阈值测试
minProfit: ethers.parseUnits('0.5', 6)  // 0.5 USDC
```

### Q2: 交易总是失败?

**检查清单**:
1. 合约有足够的USDC余额?
2. 钱包有足够的MATIC支付Gas?
3. 价格是否在发送交易前已经变化? (增加 `pollingInterval`)

### Q3: Gas费太高?

**优化方案**:
1. 降低 `gasPriceMultiplier` (但可能被抢跑)
2. 只在Gas价格低时执行:
```typescript
const gasPrice = await provider.getFeeData();
if (gasPrice.gasPrice > maxGasPrice) {
  console.log('Gas价格过高,跳过');
  return;
}
```

### Q4: 如何停止机器人?

```bash
# 按 Ctrl+C 优雅退出
# 或
pkill -f "node.*index.ts"
```

### Q5: 想提取合约中的资金?

```bash
npx hardhat run scripts/withdraw.ts --network polygon
```

## 安全检查清单

- [ ] `.env` 文件不在 Git 中
- [ ] 私钥没有泄露
- [ ] 合约 owner 是你的钱包地址
- [ ] 测试网充分测试后再上主网
- [ ] 设置了合理的最大交易规模
- [ ] 监控钱包余额和合约余额

## 下一步优化

1. **添加更多DEX**: 编辑 `src/config/pools.json`
2. **添加Telegram通知**: 集成Telegram Bot API
3. **添加Web Dashboard**: 运行 `npm run dashboard`
4. **优化Gas**: 使用Flashbots或私有RPC
5. **多链部署**: 同时在Arbitrum、Base等链上运行

## 获取帮助

如果遇到问题:
1. 检查日志文件
2. 在测试网上重现问题
3. 使用 Hardhat console 调试合约

祝套利顺利! 🚀

