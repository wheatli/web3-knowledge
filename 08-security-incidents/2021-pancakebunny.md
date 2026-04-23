---
title: PancakeBunny 闪贷 + BUNNY 无限增发（2021-05-19, ~$45M）
module: 08-security-incidents
category: DeFi
date: 2021-05-19
loss_usd: 45000000
chain: [BSC]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://pancakebunny.medium.com/hello-bunny-fam-our-post-mortem-of-the-hack-is-below-b30c1e6ff48
  - https://peckshield.medium.com/bunny-incident-root-cause-analysis-2e3f5c2e9ea9
  - https://rekt.news/pancake-bunny-rekt/
  - https://slowmist.medium.com/slowmist-analysis-of-pancakebunny-45m-hack-e4bcc63b2c94
tx_hashes:
  - 0x897c2de73dd55d7701e1b69ffb3a17b0f4801ced88b0c75fe1551c5fcce6a979 (BSC, 主攻击 tx)
---

# PancakeBunny 闪贷 + 铸币攻击

> **TL;DR**：2021-05-19，BSC 上 PancakeBunny（Yearn-style 聚合器）VaultFlipToFlip 金库被闪贷攻击，攻击者用 PancakeSwap 闪贷操纵 USDT-BNB / BUNNY-BNB 池价格，让 `getBunnyPrice()` 返回畸形值，触发 `_minterMint()` 超额铸 ~697 万 BUNNY，随即抛售，BUNNY 价格从 $146 闪崩至 $2，社区损失约 **$45M**。

## 1. 事件背景

- **项目**：PancakeBunny 是 BSC 上类 Yearn 的收益聚合器，TVL 峰值约 $7B；BUNNY 为平台治理/奖励代币。
- **时间轴**：
  - 2021-05-19 18:34 UTC：主攻击 tx 上链（BSC 区块 #7,163,607 附近）。
  - 19:00 UTC：BUNNY 价格从 $146 跌至 ~$2。
  - 约 1 小时后 PancakeBunny 与 PeckShield 公开披露。
- **发现过程**：BSC 链上监控（PeckShield、Beosin）几乎实时捕捉；团队 Discord 同步确认。

## 2. 事件影响

- **直接损失**：约 **$45M**（2021-05-19 快照），其中攻击者实际带走约 114,631 BNB + 697 万 BUNNY 抛售所得 ~$45M 等值资产。
- **受害方**：BUNNY 持有人（价格闪崩）+ 金库 LP。
- **连带影响**：BSC 生态连锁——AutoShark、Merlin Labs、Bogged Finance 等 PancakeBunny fork 协议在随后两周内相继出事；BSC "同质化 fork + 闪贷范式"成为审计方警示典型。
- **资金去向**：攻击者将 BNB 跨 Anyswap 桥到以太坊，分批通过 Tornado Cash 清洗；部分资金被 Binance 冻结但未明确返还数额。

## 3. 技术根因（代码级分析）

- **漏洞分类**：预言机（Pair reserves 直读）+ 业绩挂钩铸币（performance fee → mint BUNNY）。
- **受损合约**：`VaultFlipToFlip.sol` / `ZapBSC.sol` / `PriceCalculatorBSC.sol`，核心计算在 `BunnyMinterV2` 与 `PriceCalculatorBSC.getUnderlyingPrice()` / `valueOfAsset()`。
- **关键代码（简化）**：

```solidity
// PriceCalculatorBSC.valueOfAsset  —— 旧版本
function valueOfAsset(address asset, uint amount) public view returns (uint valueInBNB, uint valueInUSD) {
    // 对 LP token：直接读 PancakePair reserves —— 单块可被闪贷打歪
    if (isFlip(asset)) {
        (uint reserve0, uint reserve1, ) = IPancakePair(asset).getReserves();
        ...
    } else {
        // 对单币：PancakePair(BNB/asset).getReserves() 折算
        valueInBNB = reserve0.mul(amount).div(reserve1);   // ← 同块可操纵
    }
}

// BunnyMinterV2.mintFor —— 把 performance fee 折算为 BUNNY
function mintFor(address asset, uint _feeSum, uint _profit, address to, uint) external onlyMinter {
    uint bnbValue = priceCalculator.valueOfAsset(asset, _feeSum);
    uint mintBunny = amountBunnyToMint(bnbValue);       // ← bnbValue 虚高 → 铸币量虚高
    if (mintBunny == 0) return;
    _mint(to, mintBunny);
}

function amountBunnyToMint(uint bnbProfit) public view returns (uint) {
    return bnbProfit.mul(priceCalculator.priceOfBNB()).div(priceCalculator.priceOfBunny());
    // priceOfBunny 也来自 PancakeSwap BUNNY-BNB 池 —— 闪贷可砸下去
}
```

错在：`valueOfAsset` 直接读 `PancakePair.getReserves()`，未使用 TWAP；`priceOfBunny` 同样来自即时 AMM 现货；两者组合导致 `mintBunny` 可被单块操纵为任意大。

- **攻击步骤**：
  1. 从 PancakeSwap 闪贷 ~230 万 BNB（多池联动）。
  2. 在 USDT-BNB / BUNNY-BNB 池中大额换出 BUNNY，同时把 BUNNY-BNB 池价格推高 → `priceOfBunny` 在金库视角**也被推高**（但真实市场被预期反向）。
  3. 调用 VaultFlipToFlip `getReward()`：该函数结算 performance fee，以 `valueOfAsset(LP)` 计价。由于 LP 的两端被闪贷扭曲，`bnbValue` 被算得极大。
  4. `BunnyMinter.mintFor` 按"虚高 bnbValue / 虚高 priceOfBunny"仍然算出 697 万 BUNNY —— 因为协议本意是"以 BNB 价值 ÷ BUNNY 价"算奖励，但真正被操纵的是 **reserves 的绝对量**而不是比值。
  5. 攻击者瞬间获得 697 万 BUNNY，砸回 BUNNY-BNB 池换回 BNB。
  6. 偿还闪贷，净利润 ~114,631 BNB。
  7. 关键 tx：`0x897c2de73dd55d7701e1b69ffb3a17b0f4801ced88b0c75fe1551c5fcce6a979`。

- **审计盲区**：PancakeBunny 未在事件前公开权威审计；社区此前有提示"奖励铸币依赖 spot 价"，但未被响应。PeckShield 在后续补审中明确新增 TWAP 要求。

## 4. 事后响应

- **项目方**：上线 pBUNNY 补偿代币 + IOU 机制，通过未来协议收入分批赎回；迁移至 V2 架构，改用 Chainlink + TWAP 双重预言机。
- **赔付**：IOU 模式引发争议，部分用户只拿回小部分；2021-07-16 MOUND（创始团队）遭遇第二次 4.5M 攻击，信任再度受挫。
- **行业连锁**：BSC 所有 Yearn-fork 被迫审查铸币逻辑；Chainlink BSC feed 需求激增。
- **归因**：归因未公开。

## 5. 启发与教训

- **开发者**：
  - "业绩奖励 → 铸币"链路中的每一个价格源都必须是抗操纵的 TWAP / Chainlink。
  - `getReserves()` 永远不得直接用于经济关键决策。
  - 铸币函数应有每块上限（`mintPerBlock`）与治理熔断。
- **审计方**：铸币权限函数应被特别标注；fuzz 场景需包含 "外部 pair reserves 翻倍 → 铸币上限" 的不变量。
- **用户**：BSC 上未经审计的 fork 项目应默认视为高风险；BUNNY 式"高 APR + 平台币铸造奖励"机制本质脆弱。
- **协议**：补偿机制如 IOU 应事前设计，而非事后仓促推出。

## 6. 参考资料

- PancakeBunny 官方：<https://pancakebunny.medium.com/hello-bunny-fam-our-post-mortem-of-the-hack-is-below-b30c1e6ff48>
- PeckShield：<https://peckshield.medium.com/bunny-incident-root-cause-analysis-2e3f5c2e9ea9>
- rekt.news：<https://rekt.news/pancake-bunny-rekt/>
- SlowMist：<https://slowmist.medium.com/slowmist-analysis-of-pancakebunny-45m-hack-e4bcc63b2c94>
- DeFiHackLabs：<https://github.com/SunWeb3Sec/DeFiHackLabs>（PancakeBunny 子目录）
- 关键 tx：<https://bscscan.com/tx/0x897c2de73dd55d7701e1b69ffb3a17b0f4801ced88b0c75fe1551c5fcce6a979>

---

*Last verified: 2026-04-23*
