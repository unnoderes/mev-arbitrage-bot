# 用 Balancer V2 的零费率闪电贷，而非 Aave V3

状态: accepted（取代 `docs/architecture-decisions.md` 的 ADR-003）

套利机会的利润很薄，任何按借款额百分比收取的闪电贷溢价都会变成一道硬性利润地板：Aave V3 收 0.05%，借 10,000 美元就要付 5 美元，足以吃掉绝大多数机会。因此资金方选用 Balancer V2 Vault 的 `flashLoan`，其费率在 Polygon 主网实测为 0。

## 代价

- 可借规模受 Vault 实际持币量限制，不是无限。实测（区块 95,797,764 前后）：34,636 USDC、540,762 USDCe、234.9 WETH、796,070 WPOL。超出这个量的机会做不了。
- Vault 的余额会随他人存取而变化，需要在运行时读取而不是硬编码。
- Aave V3 保留为备选。若 Balancer 不可用，门槛必须改成「价差 > 0.05% + Gas + 边际」。

## 依据

- `ProtocolFeesCollector(0xce88686553686DA562CE7Cea497CE749DA109f9F).getFlashLoanFeePercentage()` 返回 `0`。
- 注意：`Vault.getFlashLoanFeePercentage()` 在该部署上会 revert，费率要从 collector 读。
