# 智能合约设计

> **本文档的合约已作废（2026-10）。** 不要照抄下面的代码。
> 权威接口与安全不变量见 [`07-final-design.md`](./07-final-design.md) 第 7 节。
>
> 三处根本性变化：
> 1. **去掉 `onlyOwner`**。改为任何人可调用，安全性由「归还闪电贷后计价代币余额增量 ≥ minProfit」保证。
>    见 [ADR-0003](./adr/0003-permissionless-executor.md)。
> 2. **不再只调 `IUniswapV2Router02`**。可交易的流动性几乎全在 Uniswap V3，
>    路由步骤需支持 `SwapRouter02.exactInputSingle`。见 [ADR-0002](./adr/0002-v3-primary.md)。
> 3. **利润门槛不再是固定值**。改为 `max(0.02 计价单位, 2 × Gas)`，且合约侧有 `ownerMinProfit` 下限。

## 合约架构

```
FlashArbitrage.sol (主合约)
├── 继承: Ownable (OpenZeppelin)
├── 接口依赖:
│   ├── IUniswapV2Router02
│   ├── IUniswapV3Router
│   └── IERC20
└── 核心功能:
    ├── triangularArbitrage() - 三角套利
    ├── crossDexArbitrage() - 跨DEX套利
    └── withdraw() - 紧急提款
```

## 合约代码框架

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";

contract FlashArbitrage is Ownable {
    // 事件
    event ArbitrageExecuted(
        string indexed strategy,
        uint256 profitAmount,
        uint256 gasUsed
    );
    
    event EmergencyWithdraw(
        address indexed token,
        uint256 amount
    );
    
    // 状态变量
    bool private locked;  // 重入锁
    bool public paused;   // 暂停开关
    
    // 构造函数
    constructor() Ownable(msg.sender) {}
    
    // ============ 修饰器 ============
    
    modifier noReentrant() {
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Contract paused");
        _;
    }
    
    // ============ 核心函数 ============
    
    /**
     * @notice 三角套利: tokenA → tokenB → tokenC → tokenA
     * @param routers 三个Router地址
     * @param tokens 四个token地址 [A, B, C, A]
     * @param amountIn 输入金额
     */
    function triangularArbitrage(
        address[3] calldata routers,
        address[4] calldata tokens,
        uint256 amountIn
    ) 
        external 
        onlyOwner 
        nonReentrant 
        whenNotPaused 
    {
        uint256 balanceBefore = IERC20(tokens[0]).balanceOf(address(this));
        require(balanceBefore >= amountIn, "Insufficient balance");
        
        uint256 currentAmount = amountIn;
        
        // Step 1: tokenA → tokenB
        currentAmount = _swap(
            routers[0],
            tokens[0],  // tokenA
            tokens[1],  // tokenB
            currentAmount
        );
        
        // Step 2: tokenB → tokenC
        currentAmount = _swap(
            routers[1],
            tokens[1],  // tokenB
            tokens[2],  // tokenC
            currentAmount
        );
        
        // Step 3: tokenC → tokenA
        currentAmount = _swap(
            routers[2],
            tokens[2],  // tokenC
            tokens[3],  // tokenA (same as tokens[0])
            currentAmount
        );
        
        uint256 balanceAfter = IERC20(tokens[0]).balanceOf(address(this));
        uint256 profit = balanceAfter - balanceBefore;
        
        require(profit > 0, "No profit");
        
        emit ArbitrageExecuted("triangular", profit, gasleft());
    }
    
    /**
     * @notice 跨DEX套利: 在DEX1买入,在DEX2卖出
     * @param buyRouter 买入的Router
     * @param sellRouter 卖出的Router
     * @param tokenIn 输入token
     * @param tokenOut 中间token
     * @param amountIn 输入金额
     */
    function crossDexArbitrage(
        address buyRouter,
        address sellRouter,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) 
        external 
        onlyOwner 
        nonReentrant 
        whenNotPaused 
    {
        uint256 balanceBefore = IERC20(tokenIn).balanceOf(address(this));
        require(balanceBefore >= amountIn, "Insufficient balance");
        
        // 在buyRouter买入tokenOut
        uint256 tokenAmount = _swap(buyRouter, tokenIn, tokenOut, amountIn);
        
        // 在sellRouter卖出tokenOut
        uint256 amountOut = _swap(sellRouter, tokenOut, tokenIn, tokenAmount);
        
        uint256 balanceAfter = IERC20(tokenIn).balanceOf(address(this));
        uint256 profit = balanceAfter - balanceBefore;
        
        require(profit > 0, "No profit");
        
        emit ArbitrageExecuted("cross-dex", profit, gasleft());
    }
    
    // ============ 内部函数 ============
    
    function _swap(
        address router,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) 
        private 
        returns (uint256 amountOut) 
    {
        // 授权router
        IERC20(tokenIn).approve(router, amountIn);
        
        // 构造路径
        address[] memory path = new address[](2);
        path[0] = tokenIn;
        path[1] = tokenOut;
        
        // 执行swap
        uint256[] memory amounts = IUniswapV2Router02(router)
            .swapExactTokensForTokens(
                amountIn,
                0,  // 暂不设最小输出,由外层检查利润
                path,
                address(this),
                block.timestamp + 300  // 5分钟deadline
            );
        
        return amounts[1];
    }
    
    // ============ 管理函数 ============
    
    /**
     * @notice 紧急提款
     */
    function withdraw(address token) external onlyOwner {
        uint256 balance = IERC20(token).balanceOf(address(this));
        require(balance > 0, "No balance");
        
        IERC20(token).transfer(owner(), balance);
        
        emit EmergencyWithdraw(token, balance);
    }
    
    /**
     * @notice 暂停/恢复合约
     */
    function setPaused(bool _paused) external onlyOwner {
        paused = _paused;
    }
    
    /**
     * @notice 接收ETH (用于支付Gas)
     */
    receive() external payable {}
}
```

## 合约优化要点

### 1. Gas优化

**使用calldata而非memory**
```solidity
// ✅ 好
function f(address[3] calldata routers) external {}

// ❌ 差
function f(address[3] memory routers) external {}
```

**避免重复的SLOAD**
```solidity
// ✅ 好
address owner = owner();  // 一次SLOAD
doSomething(owner);
doOtherthing(owner);

// ❌ 差
doSomething(owner());     // 每次都SLOAD
doOtherthing(owner());
```

**使用unchecked (Solidity 0.8+)**
```solidity
unchecked {
    currentAmount++;  // 节省溢出检查的Gas
}
```

### 2. 安全检查

**重入保护**
```solidity
bool private locked;

modifier noReentrant() {
    require(!locked, "Reentrant");
    locked = true;
    _;
    locked = false;
}
```

**滑点保护** (可选,链下已检查)
```solidity
function triangularArbitrage(
    // ...
    uint256 minProfit  // 新增参数
) external {
    // ...
    require(profit >= minProfit, "Slippage too high");
}
```

## 测试策略

### Hardhat测试框架

```typescript
// test/FlashArbitrage.test.ts
import { ethers } from "hardhat";
import { expect } from "chai";

describe("FlashArbitrage", function() {
  let arbitrage: any;
  let owner: any;
  
  beforeEach(async function() {
    [owner] = await ethers.getSigners();
    
    const FlashArbitrage = await ethers.getContractFactory("FlashArbitrage");
    arbitrage = await FlashArbitrage.deploy();
  });
  
  describe("triangularArbitrage", function() {
    it("应该成功执行三角套利", async function() {
      // Fork Polygon主网
      await network.provider.request({
        method: "hardhat_reset",
        params: [{
          forking: {
            jsonRpcUrl: process.env.POLYGON_RPC_URL,
            blockNumber: 50000000  // 固定区块,保证可重复
          }
        }]
      });
      
      // 给合约转USDC
      const usdc = await ethers.getContractAt(
        "IERC20",
        "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174"  // Polygon USDC
      );
      
      // 从富豪地址转账
      await impersonateAccount("0x...");  // 某个USDC大户
      await usdc.transfer(arbitrage.address, parseUnits("100", 6));
      
      // 执行套利
      const tx = await arbitrage.triangularArbitrage(
        [QUICKSWAP_ROUTER, UNISWAP_ROUTER, SUSHISWAP_ROUTER],
        [USDC, WETH, MATIC, USDC],
        parseUnits("100", 6)
      );
      
      const receipt = await tx.wait();
      
      // 验证事件
      const event = receipt.events.find(e => e.event === 'ArbitrageExecuted');
      expect(event.args.profitAmount).to.be.gt(0);
      
      // 验证余额增加
      const finalBalance = await usdc.balanceOf(arbitrage.address);
      expect(finalBalance).to.be.gt(parseUnits("100", 6));
    });
    
    it("应该在无利润时revert", async function() {
      // 构造一个必定亏损的路径
      await expect(
        arbitrage.triangularArbitrage(/* 亏损参数 */)
      ).to.be.revertedWith("No profit");
    });
  });
  
  describe("安全性", function() {
    it("非owner不能调用", async function() {
      const [, attacker] = await ethers.getSigners();
