---
title: Lendf.Me ERC777 重入攻击（2020-04-19, ~$25M）
module: 08-security-incidents
category: DeFi
date: 2020-04-19
loss_usd: 25000000
chain: [Ethereum]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/slowmist-analysis-of-the-dforce-hack-d6082e2e5111
  - https://peckshield.medium.com/uniswap-lendf-me-hacks-root-cause-and-loss-analysis-50f3263dcc09
  - https://rekt.news/lendfme-rekt/
  - https://medium.com/dforcenet/about-the-exploit-of-lendf-me-cc59ed345ac0
tx_hashes:
  - 0xae7d664bdfcc54220df4f18d339005c6faf6e62c9ca79c56387bc0389274363b (Ethereum, 初次重入)
  - 0xa1761b66c3bb10d16822b0e2ed23e8e19d44f79beb4e32701b3e83d035d7aa85 (Ethereum, imBTC 路径)
---

# Lendf.Me ERC777 重入攻击

> **TL;DR**：2020-04-19，dForce 旗下借贷协议 Lendf.Me 遭遇重入攻击，攻击者利用 imBTC（ERC777）的 `tokensToSend` 回调，在 `supply()` 记账前反复递归，虚增抵押品后借出所有池内资产，造成约 $25M 全盘清空。这是 ERC777 回调与标准 CEI 未对齐的教科书案例，与同日 Uniswap V1 imBTC 池事件同源。

## 1. 事件背景

- **项目**：Lendf.Me 是 dForce Network 旗下的去中心化借贷协议，基于 Compound V1 fork。事件前 TVL 约 $25M，主要支持 USDT、USDC、imBTC、WETH 等资产抵押借贷。
- **imBTC**：由 Tokenlon（imToken）发行的 BTC 锚定币，采用 ERC777 标准，对 `to` 地址有 `tokensReceived` 回调、对 `from` 地址有 `tokensToSend` 回调。
- **时间轴**：
  - 2020-04-18 01:58 UTC：Uniswap V1 的 imBTC 池先被同类手法掏空（~$220K）。
  - 2020-04-19 08:58 UTC：Lendf.Me 首笔重入交易上链。
  - 2020-04-19 数小时内：池内 WETH、WBTC、USDT、USDC、HUSD、PAX、TUSD、BUSD、CHAI 等 18 种资产几乎全部被借空。
  - 2020-04-19 10:00+ UTC：PeckShield、SlowMist 同步披露。
  - 2020-04-21–22：攻击者主动退还几乎全部资产（因在 1inch 上操作时泄露 IP，与交易所 KYC 匹配）。
- **发现过程**：PeckShield 链上监控系统率先捕捉到 Lendf.Me 异常借款，同时 SlowMist 对 imBTC/ERC777 组合发出预警。

## 2. 事件影响

- **直接损失**：按 2020-04-19 价格快照约 **$25M**，包括 ~55,159 ETH、9 WBTC、291 HBTC、1,056 imBTC、1.26M USDT、680K USDC、55K HUSD、4.5K PAX、4.5K TUSD、5.8M USDx、3.2K CHAI、1.76M BUSD 等。
- **受害方**：所有在 Lendf.Me 存款的用户——这是一次 100% 清仓事件。
- **连带影响**：imBTC 短暂脱锚；Uniswap V1 架构上同样存在重入面，市场对 ERC777+DeFi 组合重新审视；Tokenlon 暂停 imBTC 转账功能；Compound、dYdX 等 Compound fork 协议发出"不上架回调型代币"声明。
- **资金去向**：攻击者在 2020-04-21 开始通过 0xa9bf70a420d364e923c74448d9d817d3f2a77822 等地址主动归还；至 04-22 几乎全量退回，用户按比例获赔。

## 3. 技术根因（代码级分析）

- **漏洞分类**：重入（ERC777 `tokensToSend` 回调） × 记账顺序错误（非 CEI）。
- **受损合约**：Lendf.Me MoneyMarket 合约（Compound V1 fork），地址 `0x0eEe3E3828A45f7601D5F54bF49bB01d1A9dF5ea`（已销毁）；归档代码见 PeckShield 分析文章附录及 DeFiHackLabs。
- **关键代码（简化）**：

```solidity
// Compound V1 / Lendf.Me MoneyMarket.supply()
function supply(address asset, uint amount) public returns (uint) {
    Market storage market = markets[asset];
    Balance storage balance = supplyBalances[msg.sender][asset];

    // 1) 先计算新余额（基于链上 balanceOf 差值）
    uint userSupplyCurrent = calculateBalance(balance.principal, ...);
    uint userSupplyUpdated = userSupplyCurrent + amount;

    // 2) 执行外部转账 —— 此处触发 ERC777 tokensToSend 回调
    //    回调内攻击者再次调用 withdraw/borrow，读到的仍是"旧余额 + amount"
    err = doTransferIn(asset, msg.sender, amount);   // ← 回调入口
    if (err != Ok) return fail(...);

    // 3) 回调返回后才写入状态（错在这里：Interaction 在 Effects 之前）
    balance.principal = userSupplyUpdated;           // ← 记账滞后
    balance.interestIndex = market.supplyIndex;
    market.totalSupply = newTotalSupply;
    return Ok;
}
```

问题核心：`doTransferIn` 会触发 imBTC 的 `tokensToSend` 钩子，攻击者在钩子中回调 Lendf.Me 的 `withdraw()`。由于 `balance.principal` 尚未更新，每次重入都让"账面余额 > 实际余额"，攻击者可以反复把同一批 imBTC 计入抵押品数倍，进而借出其他资产。

- **攻击步骤**：
  1. 预存极少量 imBTC，建立抵押上下文。
  2. 调用 `supply(imBTC, X)`：Lendf.Me 发起 `transferFrom`，imBTC 在转账中 `call` 攻击者的 `tokensToSend`。
  3. 回调中攻击者调用 `withdraw(imBTC, X)`：由于 `supply` 还没记账，合约认为余额翻倍，准许提现。
  4. 循环数次后，用"虚高的 imBTC 抵押品"调用 `borrow()`，依次掏空 WETH/USDT/USDC/WBTC/... 18 种资产池。
  5. 关键 tx：`0xae7d664b...4363b`（Etherscan 可查）。
- **审计盲区**：Lendf.Me 沿用 Compound V1 代码，2019 年 Trail of Bits 曾审计 Compound，但当时 ERC777 尚未在 DeFi 普及；fork 方未对"新上架资产的回调面"做二次评估。OpenZeppelin、ConsenSys Diligence 同期文章已提醒 ERC777 风险，但未被 fork 团队吸收。

## 4. 事后响应

- **项目方**：dForce 在事件后数小时暂停合约；Mindao Yang 公开致歉。
- **资产追回**：攻击者在 1inch 交易时被其 API 记录 IP，并在新加坡某交易所 KYC 匹配。Singapore Police Force 介入，攻击者于 04-21 开始分批归还，至 04-22 几乎全部退回，用户按比例获全额赔付。
- **Tokenlon 响应**：紧急暂停 imBTC 转账，上线新版 `WrappedBTC`，取消 ERC777 默认回调或增加白名单。
- **行业连锁**：Uniswap V2 即将发布，核心开发者重申"不对回调型代币做额外兼容"；Compound 官方声明不支持 ERC777；此后所有 Compound fork（Cream、Benqi 等）上架代币前强制核查标准。

## 5. 启发与教训

- **开发者**：
  - 任何涉及 `transfer`/`transferFrom` 的函数必须先写状态再做外部调用（Checks-Effects-Interactions），或对整个函数加 `nonReentrant`。
  - 对 fork 代码，必须重新评估"新资产的外部回调面"而非复用原审计。
  - 在上架资产前静态扫描 ERC777/ERC1363/hooks。
- **审计方**：
  - Slither `reentrancy-eth`、`reentrancy-benign` 规则需结合"代币是否有回调"启用；ToB 的 `crytic` 套件此后加入 ERC777 探针。
  - 不变量 fuzz：`Σ supplyBalances == token.balanceOf(market)`，违反即存在双计。
- **用户**：借贷协议"上架审查"水平是风控核心指标；关注协议是否明确声明拒绝回调型资产。
- **协议层**：多资产借贷协议应在 `supply/withdraw/borrow` 外层包裹 `ReentrancyGuard`（Compound V2 Unitroller 之后如此做）。

## 6. 参考资料

- SlowMist：<https://slowmist.medium.com/slowmist-analysis-of-the-dforce-hack-d6082e2e5111>
- PeckShield：<https://peckshield.medium.com/uniswap-lendf-me-hacks-root-cause-and-loss-analysis-50f3263dcc09>
- rekt.news：<https://rekt.news/lendfme-rekt/>
- dForce 官方：<https://medium.com/dforcenet/about-the-exploit-of-lendf-me-cc59ed345ac0>
- ConsenSys Diligence ERC777 警告：<https://media.consensys.net/uniswap-audit-b90335ac007>
- DeFiHackLabs 复现：<https://github.com/SunWeb3Sec/DeFiHackLabs>（参见 Lendf.Me / Uniswap-imBTC 子目录）
- 关键 tx：<https://etherscan.io/tx/0xae7d664bdfcc54220df4f18d339005c6faf6e62c9ca79c56387bc0389274363b>

---

*Last verified: 2026-04-23*
