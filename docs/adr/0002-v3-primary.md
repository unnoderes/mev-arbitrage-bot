# 以 Uniswap V3 为主，V2 只保留步骤类型

状态: accepted（取代 `docs/architecture-decisions.md` 的 ADR-006）

原设计要监控「3 个 DEX × 3 个交易对 = 9 个 V2 池子」。上链核实后发现这张表是空的：SushiSwap 的 USDC/WETH 池只有 0.08 USDC 和 0 WETH；QuickSwap 的 USDC/WPOL 池只有 671 USDC。而 Uniswap V3 的 0.05% 档在 USDC 侧深 11 倍、WETH 侧深 20 倍。执行路径因此改为以 V3 为主。

## 为什么不是纯 V3

合约里保留 V2 步骤类型（`Step.kind = 0`）只多一个分支和一组测试，而删掉它会让「以后某个 V2 池子变深了」变成一次合约改动。注册表按深度下限自动过滤，因此实际部署后 V2 步骤会是空集合——这是注册表的输出，不是写死的假设。

## 代价

- V3 的定价不能用 `x*y=k`，必须读 `slot0` 的 `sqrtPriceX96`，并用 `QuoterV2` 校验可执行价格。
- 多出 tick 跨越、费率档枚举、`SwapRouter02` 的 `exactInputSingle` 编码三类复杂度。
- 原 `docs/03-smart-contract.md` 里只调用 `IUniswapV2Router02` 的合约不再成立。
