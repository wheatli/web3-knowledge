---
title: Cream Finance V2 预言机操纵（2021-10-27, ~$130M）
module: 08-security-incidents
category: DeFi
date: 2021-10-27
loss_usd: 130000000
chain: [Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://medium.com/cream-finance/post-mortem-exploit-oct-27-ea6e0f5d11c
  - https://peckshield.medium.com/cream-finance-incident-root-cause-analysis-6d6f8daa12b6
  - https://rekt.news/cream-rekt-2/
  - https://slowmist.medium.com/slowmist-analysis-of-the-cream-finance-hack-6e7c0e0e5d1f
  - https://mudit.blog/cream-hack-analysis/
tx_hashes:
  - 0x0fe2542079644e107cbf13690eb9c2c65963ccb79089ff96bfaf8dced2331c92 (Ethereum, 主攻击 tx)
---

# Cream Finance yUSDVault LP 定价操纵

> **TL;DR**：2021-10-27，借贷协议 Cream Finance（基于 Compound V2 fork）第三次被攻击，损失约 **$130M**。攻击者利用 Cream 把 `yearn yUSDVault`（4Crv 策略）的 LP 代币作为抵押资产、并用 `pricePerShare` 作为其美元定价的设计：通过巨额闪贷在 Curve 3Pool 中把 yVault 的 `pricePerShare` 短暂拉高一倍以上，从而让少量 LP token 拥有天文数字的抵押价值，借空 Cream 池内几乎所有资产。这是 2020–2021 年"闪贷操纵 vault 定价"范式的集大成之作。

## 1. 事件背景

- **项目**：Cream Finance 是 Compound V2 的多资产 fork，作为 Yearn Ecosystem 成员之一；事件前 TVL 约 $1.5B，支持上百种资产抵押。2021 年内已经发生过两次大规模被盗（2021-02 AMP 重入 ~$37M、2021-08-30 Fork from Iron Bank ~$18M），本次是第三次。
- **关键设计**：Cream 允许 `crYUSDVault`—— 以 yearn yUSDVault (yDAI+yUSDC+yUSDT+yTUSD Curve LP 的 Yearn 封装，俗称 yUSD) 作为抵押资产，并通过 Cream 的 PriceOracleProxy 把 yUSD 的美元价格取为 `yVault.pricePerShare() * priceOfUnderlying4Crv`（其中 `priceOfUnderlying4Crv` 近似 $1，因此 yUSD 价格近似 `pricePerShare`）。
- **时间轴**：
  - 2021-10-27 13:54 UTC：主攻击 tx 上链（区块 #13,499,798 附近）。
  - 14:10 UTC：PeckShield、SlowMist、mudit.blog 陆续披露。
  - 14:30 UTC：Cream 官方宣布暂停 Ethereum 市场合约，保留 Fantom、BSC 市场。
  - 2021-10-28 起：Yearn、C.R.E.A.M.、MakerDAO 社区就"LP 代币作为抵押"展开广泛反思。
  - 2021-11：Cream 官方发布 post-mortem。
- **发现过程**：链上监控与 Mudit Gupta、Igor Igamberdiev 等研究者几乎实时分析。

## 2. 事件影响

- **直接损失**：按 2021-10-27 快照，攻击者净获利约 **$130M**，包括 ETH、DAI、USDC、USDT、WBTC 以及一些小币种，攻击者总共借出并保留了约 92,630,145 DAI + 3,686 WETH + 等值资产（rekt 与 Cream 官方统计口径略有差异）。
- **受害方**：
  - Cream Ethereum 市场存款人（部分被坏账覆盖，只能分批赔付）。
  - Yearn yUSDVault 储户（yVault 本体未损，但因 `pricePerShare` 操纵引发短期冲击）。
  - CREAM 代币持有人（事件后 CREAM 价格大跌，协议事实性衰退）。
- **连带影响**：
  - "LP 代币做抵押 + `pricePerShare` 直读"定价范式被宣判死刑，这是 DeFi 借贷设计的重大转折点。
  - Compound fork 普遍进行审计复查；Rari Capital、Iron Bank 等收紧资产上架门槛。
  - Cream Finance 事件后基本退出头部协议行列。
- **资金去向**：攻击者通过多个 EOA 将资产兑换为 ETH / WBTC，随后通过 Tornado Cash 混币；归因未公开。

## 3. 技术根因（代码级分析）

### 3.1 关键合约

- Cream Finance `PriceOracleProxy`：负责为每种 cToken 返回底层资产 USD 价格。
  - 地址：`0x6B1ce89E86F1BbeE17fa02A07f2E1C6C90Bb6c0c`（PriceOracleProxyUSD 事发版本）。
- Cream `crYUSD` (Compound V2 CToken fork)：以 yUSDVault (`0x5dbcF33D8c2E976c6b560249878e6F1491Bca25c`) 为底层。
- Yearn `yUSDVault`：策略把 DAI/USDC/USDT/TUSD 存入 Curve Y pool 换 yDAI/yUSDC/yUSDT/yTUSD，然后 4Crv LP → yVault shares。其 `pricePerShare()` = `totalAssets() / totalSupply()`。

### 3.2 关键代码（简化，PriceOracleProxyUSD 的 yVault 分支）

```solidity
// Cream Finance PriceOracleProxyUSD，对 yVault 类底层资产的定价
function getUnderlyingPrice(CToken cToken) external view returns (uint) {
    address underlying = CErc20(address(cToken)).underlying();

    if (isYearnVault[underlying]) {
        IYearnVault yVault = IYearnVault(underlying);
        // 关键：直接调用 yVault.pricePerShare()
        uint pricePerShare = yVault.pricePerShare();
        // 乘以底层 token (4Crv) 的价格（近似 $1）
        uint underlyingPrice = getUnderlying4CrvPrice();
        return pricePerShare * underlyingPrice / 1e18;
    }
    ...
}
```

问题本质：
- `pricePerShare = totalAssets / totalSupply`。
- `totalAssets` 是 Curve 4Crv LP 价值的实时计算，依赖 Curve 的 `virtual_price` 与 LP 余额。
- 任何能在单块内把 Curve 池内 4Crv LP 对应资产**多** / totalSupply 保持**不变**的操作，都会让 `pricePerShare` 飙升。
- **"多存底层 + 不 mint shares" 的直接捐赠路径**：attacker 直接 `transfer` 资产到 yVault 或 Curve pool，不经过 `deposit` → yVault 的 `totalAssets` 增加、`totalSupply` 未变 → `pricePerShare` 上升。

### 3.3 攻击步骤（Mudit Gupta / PeckShield 分析）

1. **准备**：借助 MakerDAO 闪贷 500M DAI + AAVE 闪贷 ETH；在多个中间账户铺设。
2. **建立 Cream 抵押**：把小额 yUSD 存入 Cream 作为 `crYUSD` 抵押。
3. **把 yVault 的 pricePerShare 打上天**：
   - 通过 Curve Y pool 反复 `add_liquidity/remove_liquidity`、直接向 yVault 转入 4Crv LP，使 `totalAssets` 上升。
   - 同时避免触发 `_deposit` → 不增发 shares。
   - 结果：`pricePerShare` 从 ~$1 附近翻倍到 $2+，进而 yUSD 在 Cream 中的单位美元价翻倍。
4. **借出资产**：
   - Cream 依据操纵后的 `pricePerShare`，认为攻击者的 yUSD 抵押价值翻倍。
   - 借出 Cream Ethereum 市场中的所有流动性：ETH、DAI、USDC、USDT、WBTC 等。
5. **归还闪贷 + 利润出场**：闪贷偿还，净利润为借出资产。
6. **关键 tx**：`0x0fe2542079644e107cbf13690eb9c2c65963ccb79089ff96bfaf8dced2331c92`（一条长 tx 内串联 68 步操作；Etherscan 展开即为经典范本）。
7. **复现**：DeFiHackLabs 有可 `forge test` 的复现——<https://github.com/SunWeb3Sec/DeFiHackLabs>（Cream Finance 2021-10-27 子目录）。

### 3.4 为何审计未发现

- Cream 继承 Compound V2 审计 + 自有第三方审计，但审计员对 "yVault.pricePerShare() 可被捐赠式操纵" 这一跨协议组合风险理解不足。
- Yearn yVault 文档彼时未明确警示 "`pricePerShare` 不是抗操纵的公允价"。
- 预言机 proxy 被设计成"自动识别 yVault 类底层资产并直读 pricePerShare"——这对 Cream 运营方而言"节省了手动维护 feed 的麻烦"，但事实上是自爆开关。
- 2021 年上半年 Alpha Finance × Cream Iron Bank 攻击（~$37M）已是预警，但 Cream 未在该事件后系统性下架 LP-as-collateral 资产。

## 4. 事后响应

- **项目方**：Cream Finance 第一时间暂停 Ethereum 市场；通过治理下架所有 yVault / LP 类抵押资产；官方 post-mortem 承认"定价源未抗操纵"是根因。
- **赔付**：Cream 提议用未来协议收入分批偿还受影响储户（Iron Bank "SAFE" 计划），实际执行中进展缓慢；许多用户承受了事实性坏账。CREAM 代币价格此后长期低迷。
- **Yearn 连带**：Yearn 团队声明 yVault 本身未被攻破，但公开呼吁所有把 yVault.pricePerShare 作为预言机的协议立即下线该路径；yVault 文档随后加入"pricePerShare 可被捐赠式操纵，禁止作为外部 oracle"警示。
- **法律 / 执法**：无公开追回；攻击者未归还。归因未公开。
- **行业连锁**：
  - Aave、Compound、Venus 等主流借贷协议对"LP-as-collateral" 要求使用多预言机（Chainlink + 加权 + 时延）。
  - 2022 年 Mango Markets、Moola Market、0vix、Midas Capital 等多起事件仍延续此范式 —— 说明教训并未被所有新项目吸收。

## 5. 启发与教训

- **开发者 / 协议设计（核心）**：
  - **永远不要直接使用 `pricePerShare` / `getPricePerFullShare` / `get_virtual_price` 作为借贷抵押物的预言机来源**——它们是可被捐赠式操纵的协议内部计数，而非抗操纵公允价。
  - LP 代币若要作抵押物，需要：1) 基于底层资产的市场价 × LP 份额比例重新推导；2) 结合多 oracle 和 TWAP；3) 做抵押率保守折价（haircut）。
  - 任何引入新抵押资产必须经过"同块闪贷能否把价格打歪"的 fuzz 测试。
- **审计方**：
  - Slither / Semgrep 需新增 "borrowing-market reads pricePerShare/ get_virtual_price" 的高危告警。
  - 不变量 fuzz：`pricePerShare(t+1) <= pricePerShare(t) * maxGrowth(1 block)`，违反即说明可被捐赠式推高。
  - 跨协议组合审计：被审协议对外部协议的"信任面"必须显式列出；本案 Cream 对 yVault 与 Curve 的隐含信任是根因。
- **用户**：
  - 多次被黑的协议存在结构性问题；用户应识别"治理响应速度、审计复查完整度"的历史记录。
  - 把资金存入头部借贷协议（Aave V3、Compound V3）比短期高收益协议更稳妥。
- **行业**：
  - 2021–2022 的 vault-as-collateral 事件（Cream、Rari Fuse、Moola、Mango）构成"预言机范式" 的完整负面案例集；这些事件直接催生了 Chaos Labs、Risk DAO、Gauntlet 等专业风险评估服务。
  - DeFi 抵押资产准入应有"risk committee + cap + isolation mode"三件套（Aave V3 isolation mode、Compound V3 单抵押资产设计正是回应）。

## 6. 参考资料

- Cream 官方 post-mortem：<https://medium.com/cream-finance/post-mortem-exploit-oct-27-ea6e0f5d11c>
- PeckShield：<https://peckshield.medium.com/cream-finance-incident-root-cause-analysis-6d6f8daa12b6>
- rekt.news：<https://rekt.news/cream-rekt-2/>
- SlowMist：<https://slowmist.medium.com/slowmist-analysis-of-the-cream-finance-hack-6e7c0e0e5d1f>
- Mudit Gupta 分析：<https://mudit.blog/cream-hack-analysis/>
- Igor Igamberdiev 技术拆解（The Block Research / Twitter）：<https://twitter.com/frankresearcher/status/1453468600373813254>
- DeFiHackLabs 复现：<https://github.com/SunWeb3Sec/DeFiHackLabs>（Cream 2021-10-27 子目录）
- 关键 tx：<https://etherscan.io/tx/0x0fe2542079644e107cbf13690eb9c2c65963ccb79089ff96bfaf8dced2331c92>
- PriceOracleProxyUSD：<https://etherscan.io/address/0x6B1ce89E86F1BbeE17fa02A07f2E1C6C90Bb6c0c>
- yUSDVault：<https://etherscan.io/address/0x5dbcF33D8c2E976c6b560249878e6F1491Bca25c>

---

*Last verified: 2026-04-23*
