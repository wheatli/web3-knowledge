---
title: Harvest Finance 闪贷操纵 Curve y Pool（2020-10-26, ~$33.8M）
module: 08-security-incidents
category: DeFi
date: 2020-10-26
loss_usd: 33800000
chain: [Ethereum]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://medium.com/harvest-finance/harvest-flashloan-economic-attack-post-mortem-3cf900d65217
  - https://peckshield.medium.com/harvest-finance-incident-root-cause-analysis-c7b22cfcc6e3
  - https://rekt.news/harvest-finance-rekt/
  - https://slowmist.medium.com/slowmist-analysis-of-harvest-finance-a-33m-attack-5e0c6cd6a137
tx_hashes:
  - 0x35f8d2f572fceaac9288e5d462117850ef2694786992a8c3f6d02612277b0877 (Ethereum, USDC vault)
  - 0x0fc6d2ca064fc841bc9b1c1fad1fbb97bcea5c9a1b2b66ef837f1227e06519a6 (Ethereum, USDT vault)
---

# Harvest Finance 闪贷攻击

> **TL;DR**：2020-10-26，收益聚合器 Harvest Finance 的 `fUSDC` / `fUSDT` 金库被闪贷攻击，攻击者在 Curve `y` 池（yDAI/yUSDC/yUSDT/yTUSD）内反复 `exchange_underlying`，把 `USDC` 的虚拟价格短暂压低 / 抬高，借此在 Harvest 金库 `deposit`/`withdraw` 时以不对称的 `pricePerShare` 抽取价差，净获利约 **$33.8M**。属于"闪贷操纵第三方池 → 套利目标 vault 的 share 定价"范式。

## 1. 事件背景

- **项目**：Harvest Finance 是 2020 年 DeFi Summer 期间崛起的收益聚合器，将用户存款路由到 Curve、Compound、Idle 等策略。事件前 TVL 约 $1B。
- **金库机制**：用户存 USDC → Harvest 策略把 USDC 存入 Curve `y pool` 换 `yUSDC`，按池子 `virtual_price` 记 `pricePerFullShare`。
- **时间轴**：
  - 2020-10-26 09:24 UTC：攻击者从 Uniswap V2 闪贷 50M USDT + 18.3M USDC + 11.8M USDC。
  - 09:24:12–09:38：共发起 17 笔攻击交易，分别针对 fUSDC 与 fUSDT 两个金库。
  - 09:38 UTC：PeckShield、1inch、samczsun 社区几乎同时发出警报。
  - 约 1 小时后 Harvest 关闭相关策略。
- **发现过程**：PeckShield 链上报警；Harvest 团队在 Discord 公开确认；事后由 Harvest + PeckShield 联合复盘。

## 2. 事件影响

- **直接损失**：~13.44M USDC + 11.89M USDT + 少量其他资产 = **$33.8M**（按 2020-10-26 快照）。
- **受害方**：fUSDC、fUSDT 金库储户；FARM 代币价格事件后 1 小时从 ~$240 跌至 ~$90。
- **连带影响**：DeFi 收益聚合器整体质押量下滑约 20%；Yearn、Pickle 等同类协议连夜加入"deposit fee"或"withdrawal fee"抵御同块套利。
- **资金去向**：攻击者将资金转换为 renBTC，跨链到比特币并送入混币，无追回；事后 0xf224ab... 地址向 Harvest 归还了约 $2.5M。

## 3. 技术根因（代码级分析）

- **漏洞分类**：预言机操纵（第三方 AMM 的 `get_virtual_price` / `calc_token_amount`）+ 同块存取无保护。
- **受损合约**：`Vault.sol`（USDC: `0xf0358e8c3CD5Fa238a29301d0bEa3D63A17bEdBE`），关键函数：

```solidity
// Harvest Vault.deposit
function deposit(uint256 amount) external defense {
    _deposit(amount, msg.sender, msg.sender);
}

function _deposit(uint256 amount, address sender, address beneficiary) internal {
    // 核心：份额 = amount * totalSupply / underlyingBalanceWithInvestment
    uint256 toMint = totalSupply() == 0
        ? amount
        : amount.mul(totalSupply()).div(underlyingBalanceWithInvestment());
    _mint(beneficiary, toMint);
    IERC20(underlying()).safeTransferFrom(sender, address(this), amount);
    // 然后 invest() 把资金路由到 Curve yUSDC 策略
}
```

`underlyingBalanceWithInvestment()` 在 Curve 策略分支中最终以 `yUSDC.balanceOf(vault) * yUSDC.getPricePerFullShare() / 1e18` 或等价方法折回 USDC。由于 `y pool` 是 AMM，单块内可被 `exchange_underlying` 打歪 —— `virtual_price` / 池内组成发生倾斜，使得 `underlyingBalanceWithInvestment()` **瞬时偏低**。

错在：`deposit` 和 `withdraw` 在同一块内的 pricePerShare 完全取决于被操纵的 Curve 状态，且合约没有 `onlyEOA`、`sameBlockGuard`、`deposit fee` 等任何保护。

- **攻击步骤**（fUSDC 分支，循环执行 ~32 次）：
  1. Uniswap V2 `flashloan(USDT=50M, USDC=0)`。
  2. 在 Curve y pool 中 `exchange_underlying(USDT → USDC)`，推高池内 USDC 占比、降低 USDC 有效价（即 `get_virtual_price` 在 USDC 方向被低估）。
  3. 在 Harvest fUSDC `deposit(USDC)`：此时 `underlyingBalanceWithInvestment()` 被低估，算出的 `shares` 偏多。
  4. 在 Curve y pool 反向 `exchange_underlying(USDC → USDT)`，把池子恢复至接近原状，USDC 有效价回升。
  5. 在 Harvest fUSDC `withdraw(shares)`：此时 `underlyingBalanceWithInvestment()` 恢复正常，赎回的 USDC > 存入的 USDC。
  6. 把利润累计，循环 32 次，同块偿还 Uniswap 闪贷。
  7. 关键 tx：`0x35f8d2f572fceaac9288e5d462117850ef2694786992a8c3f6d02612277b0877`。

- **审计盲区**：Harvest 金库代码本身由 Haechi 审计过，但审计聚焦于 `rebalance` 与权限，未对"策略层外部价格源可操纵"做单块 fuzz。DeFiHackLabs 复现路径：<https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/src/test/2020-10-26-HarvestFinance>，可 `forge test -vv` 直接复现收益。

## 4. 事后响应

- **项目方**：Harvest 立刻关闭 fUSDC/fUSDT 策略；上线"同块 deposit→withdraw"手续费；引入 `commit-reveal` 式延迟。
- **赔付**：Harvest 用 FARM 金库未分配部分与团队份额发起治理投票，部分用户在数月内按比例获赔；完整补偿见 Harvest 官方 post-mortem。
- **行业连锁**：Yearn 随后升级 `v2 Vault`，加入 `deposit/withdraw` 单块差价保护；Curve 团队公开声明 "`get_virtual_price` 不是实时公允价"，此后多个协议误用 `get_virtual_price` 均被 rekt（Curve LP 操纵范式延续到 2023 年）。
- **归因**：攻击者归因未公开；未进入 FBI / Chainalysis 正式报告。

## 5. 启发与教训

- **开发者**：
  - 金库定价禁止直接读取第三方 AMM 的即时 `get_virtual_price` 或 reserves；必须引入时间加权或延迟确认。
  - 任何允许同块 `deposit → withdraw` 的金库必须加入 deposit fee 或 cooldown。
- **审计方**：
  - 单块闪贷 fuzz：`assert(shareAfter * priceAfter >= shareBefore * priceBefore)` 是 vault 类合约的核心不变量。
  - Slither 需新增 `external-price-source-read` 在同函数内与状态写入的耦合检查。
- **用户**：收益聚合器的 TVL 高不等于安全；应关注策略对第三方池的依赖度。
- **协议**：治理应建立"紧急暂停 + 白帽通道 + 资产保险基金"三件套。

## 6. 参考资料

- Harvest 官方 post-mortem：<https://medium.com/harvest-finance/harvest-flashloan-economic-attack-post-mortem-3cf900d65217>
- PeckShield：<https://peckshield.medium.com/harvest-finance-incident-root-cause-analysis-c7b22cfcc6e3>
- rekt.news：<https://rekt.news/harvest-finance-rekt/>
- SlowMist：<https://slowmist.medium.com/slowmist-analysis-of-harvest-finance-a-33m-attack-5e0c6cd6a137>
- DeFiHackLabs 复现：<https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/src/test/2020-10-26-HarvestFinance>
- 关键 tx：<https://etherscan.io/tx/0x35f8d2f572fceaac9288e5d462117850ef2694786992a8c3f6d02612277b0877>

---

*Last verified: 2026-04-23*
