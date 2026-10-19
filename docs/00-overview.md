# MEV 套利机器人 - 项目概览

> **本文档的收益预期与技术方案已过时（2026-10）。**
> 当前状态、已核实的事实与最终设计见 [`07-final-design.md`](./07-final-design.md)。
>
> 关键修正：
> - 原定的「9 个 V2 池子」里，SushiSwap 的 USDC/WETH 是**空池**（0.08 USDC），
>   QuickSwap 的 USDC/WPOL 只有 671 USDC。真正的流动性在 Uniswap V3 的 0.05% 档。
> - 「日均 $2–10」没有依据。30 个区块的全枚举实测：**当前 4 个代币、15 个 V3 池子在任何规模下都不可套利**。
> - 闪电贷改为 Balancer V2（费率实测为 0），不再用自有本金。

## 项目简介

这是一个在 Polygon 链上运行的 **MEV (最大可提取价值) 套利机器人**，通过技术手段自动捕捉 DEX 之间的价格差异并执行套利交易。

**核心特点**:
- ✅ 低成本：初始投入 < $100，月运行成本 < $5
- ✅ 低风险：原子化交易，要么全成功要么全回滚
- ✅ AI友好：完整设计文档，支持低成本AI模型编码
- ✅ 合法合规：通过技术优势捕捉价格差异，不是欺诈

## 文档结构

### 核心文档

1. **00-overview.md** (本文档) - 项目总览
2. **01-architecture.md** - 系统架构设计
3. **02-strategy-design.md** - 套利策略详细设计
4. **03-smart-contract.md** - 智能合约完整代码
5. **04-implementation-guide.md** - 实施指南 (给AI编码模型)
6. **05-quickstart.md** - 快速开始指南
7. **06-technical-challenges.md** - 技术难点和解决方案 ⭐ 重要
8. **architecture-decisions.md** - 架构决策记录 (ADR)

### 架构图

- **diagrams/01-container-diagram.mmd** - 系统容器视图
- **diagrams/02-arbitrage-sequence.mmd** - 套利执行时序图
- **diagrams/03-bot-components.mmd** - Bot Core 组件结构

## 技术栈

- **链下程序**: TypeScript + ethers.js v6 + Node.js
- **智能合约**: Solidity 0.8.20 + Hardhat
- **部署链**: Polygon (低 Gas 费)
- **RPC 节点**: Alchemy 免费版
- **Dashboard**: Next.js (可选)

## 工作原理

```
1. 监听链上价格变化 (WebSocket)
   ↓
2. 计算三角套利机会 (USDC → WETH → MATIC → USDC)
   ↓
3. 评估利润 (扣除Gas费后 > 最小利润阈值?)
   ↓
4. 构建并发送原子化交易
   ↓
5. 等待确认并记录结果
```

## 快速开始

### 阅读顺序 (首次接触)

1. **本文档** - 了解项目全貌
2. **06-technical-challenges.md** - 了解关键技术难点 ⭐
3. **architecture-decisions.md** - 理解设计决策
4. **diagrams/** - 看架构图
5. **05-quickstart.md** - 动手实施

### 阅读顺序 (准备编码)

1. **04-implementation-guide.md** - 完整编码指南
2. **06-technical-challenges.md** - 必须处理的技术难点
3. **03-smart-contract.md** - 智能合约完整代码
4. **02-strategy-design.md** - 算法和公式

## 预期收益

**⚠️ 以下数字已作废。** 保留它们只是为了说明早期设计假设错了在哪里。

实测结果：在 {USDC, USDCe, WETH, WPOL} 四个代币、15 个 Uniswap V3 池子上，
连续 30 个区块内没有任何规模（10–5,000 计价单位）能盈利。0.05% 档的往返手续费地板是 0.15%，
真实价差不足以覆盖。当前正确的预期是 **$0**，直到扩交易对为止。

~~保守估计 (Polygon, $100 本金): 每天 20-50 次机会、日均 $2-10、月收益 $60-300~~

~~乐观估计 (优化后): 月收益 $300-900~~

## 成本分析

### 初始投入
- 合约部署: $0.5-1 (一次性)
- 测试交易: $1-2
- 初始套利资金: $50-100
- **总计**: < $100

### 月运行成本
- RPC节点: $0 (Alchemy 免费版)
- 服务器: $0 (本地运行)
- 失败交易Gas: $2-5
- **总计**: < $5/月

## 风险说明

### 技术风险 (可控)
- ✅ 价格变化导致机会失效 → 发送前二次确认
- ✅ 大整数精度丢失 → 留 10% 安全边际
- ✅ WebSocket 断线 → 自动重连
- ✅ 合约被攻击 → Owner 权限控制

### 市场风险 (不可控)
- ⚠️ 竞争加剧导致利润下降
- ⚠️ Gas 费上涨吞噬利润
- ⚠️ DEX 流动性枯竭

## 法律合规

**这是合法的市场行为**:
- ✅ 通过技术优势捕捉公开的价格差异
- ✅ 类似于传统金融的高频交易 (HFT)
- ✅ 不涉及欺诈、操纵或内幕交易

## 下一步

### 对于开发者

1. **阅读技术难点** → `06-technical-challenges.md`
2. **理解架构** → `diagrams/` + `architecture-decisions.md`
3. **准备环境** → `05-quickstart.md`
4. **开始编码** → `04-implementation-guide.md`

### 对于AI编码模型

**主要输入**: `04-implementation-guide.md`  
**必读补充**: `06-technical-challenges.md`  
**参考代码**: `03-smart-contract.md`

**编码顺序**:
1. Phase 1: 智能合约 (Solidity)
2. Phase 2: 核心模块 (TypeScript)
3. Phase 3: 主程序和测试
4. Phase 4: 部署脚本

预计编码时间: 2-4 小时

---

**项目状态**: 设计完成，等待实施  
**最后更新**: 2024-10-18  
**文档版本**: v1.0
