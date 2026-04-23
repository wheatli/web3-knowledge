---
title: Mango Markets 预言机操纵事件（2022-10-11, ~$117M）
module: 08-security-incidents
category: Oracle
date: 2022-10-11
loss_usd: 117000000
chain: [Solana]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://blog.mango.markets/mango-incident-post-mortem-2c1c6b0c3b94
  - https://rekt.news/mango-markets-rekt/
  - https://www.justice.gov/usao-sdny/pr/avraham-eisenberg-found-guilty-manipulating-decentralized-cryptocurrency-exchange
  - https://twitter.com/avi_eisen/status/1580191065781366784
  - https://www.certik.com/resources/blog/mango-markets-incident-analysis
tx_hashes:
  - Solana 操纵 / 借款关键交易签名（攻击者账户公开，见 Solscan）
  - 攻击者 Solana 账户: CQvKSNnYtPTZfQRQ5jkHq8q2sJHrZTvZbjZ3u2A7GMGN
  - Ethereum 部分地址: 0x57452ADa44Bd9c1f5bA88B5BE8147bCF4A2d6A06
---

# Mango Markets 预言机操纵事件

> **TL;DR**：2022-10-11 UTC，交易员 Avraham Eisenberg 在 Solana DeFi 永续合约平台 Mango Markets 上通过现货拉盘推高 MNGO 价格，使 Pyth / Switchboard 预言机采信的价格从 $0.03 飙升至 $0.91（3000%+），其 MNGO-PERP 多头仓位账面盈利暴涨，抵押能力对等膨胀，随后借出协议内 **$117M** 资产（USDC、BTC、SOL、mSOL 等）并提走。攻击后 Eisenberg 公开承认身份，称"这是高盈利交易策略"，后于 **2023-04** 被 FBI 逮捕，**2024-04-18** 在纽约南区联邦法院被陪审团裁定商品操纵、商品欺诈、电信欺诈三项罪名成立，**2025-01-30** 被判处约 **6.5 年监禁**（以及民事赔偿）。

## 1. 事件背景

### 1.1 Mango Markets

Mango Markets 是 Solana 上的去中心化衍生品交易所（现货 + 永续 + 借贷），治理代币 MNGO，2021-08 上线，2022-10 事件前 TVL 约 $200M，Serum orderbook 为现货撮合后端，Pyth Network 与 Switchboard 提供预言机喂价。用户可用组合保证金（cross-margin）同时持有现货和永续，MNGO-PERP 为其明星市场之一。

### 1.2 时间轴

- **2022-10-11 UTC 22:23**：Eisenberg 使用两个 Mango 账户 `A`（做空）、`B`（做多）分别注资约 $5M USDC。
- **UTC 22:25-22:38**：通过 AOB/Serum 同时在账户 A/B 开立 **483M MNGO** 的 PERP 多/空对冲头寸。
- **UTC 22:38-22:50**：账户 B 在 Serum / Raydium / AscendEX（中心化，低流动性）现货市场大量吃单买入 MNGO，把 MNGO/USDC spot 价从 ~$0.03 拉升至 > $0.91。Pyth 的 MNGO/USD 聚合价跟随。
- **UTC 22:50–23:15**：账户 B 账面未实现盈利放大 ~$400M+，可用保证金随之放大；借出协议金库内 USDC、BTC、SOL、mSOL 等共价值 **$117M**，提至其他地址。
- **UTC 23:30**：@0xNox、@joshua_j_lim、@mangomarkets 公开披露，协议 borrow/lend 被暂停。
- **2022-10-15**：Mango DAO 达成 Settlement Proposal，Eisenberg 返还 ~$67M BTC/ETH/SOL，留下 ~$47M USDC 作为"bug bounty"（争议巨大）。
- **2022-10-15**：Eisenberg 发推公开承认身份，称"高盈利交易策略"。
- **2022-12-27**：Eisenberg 在波多黎各被 FBI 逮捕。
- **2024-04-18**：纽约南区陪审团裁定商品操纵 (18 U.S.C. § 1348 / CEA)、商品欺诈、电信欺诈三项罪名成立。
- **2025-01-30**：判刑约 6.5 年（**78 月**，根据 SDNY 判决文件）。

### 1.3 发现过程

由于操纵行为在 Solana 公链 + Mango 前端实时可见，社区在 30 分钟内识别。Mango Labs 的 Daffy Durairaj 第一时间发推预警。

## 2. 事件影响

### 2.1 直接损失

- Mango 金库流失 **$117M**（USDC、SOL、mSOL、BTC、ETH、USDT、MNGO、SRM 等混合）；
- 借款方为攻击者账户 B，抵押品为被操纵的 MNGO-PERP 盈利，属"协议允许但预言机失真"的合法借款。

### 2.2 受害方

- Mango 的存款人、USDC 流动性池 LP；
- 使用 MNGO-PERP 做空的对手方在短暂时间内也被操纵影响；
- 协议治理代币 MNGO 价短暂拉升，事后崩盘。

### 2.3 连带影响

- Solana DeFi 再次受重创（距离 Wormhole 事件仅 8 个月）；
- 暴露"cross-margin + 低流动性 perp + 现货喂价预言机"组合的系统性漏洞，Drift、01、Zeta 等协议纷纷更新预言机与保证金模型；
- 引发"代码即法律 vs 金融操纵法"的大讨论；SDNY 的判决标志着**链上经济学型攻击**可被刑事定罪（美国语境下）。

### 2.4 资金去向

- 约 $67M 按 Settlement 条款返还 DAO；
- $47M 由 Eisenberg 保留（后续被执法部门部分追回 / 冻结）；
- Chainalysis 与 Arkham 追踪账户活动用于举证。**归因：Avraham Eisenberg 本人，非 APT**。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**预言机操纵 + 保证金模型风控盲区**（非合约漏洞，是经济学 / 风险管理缺陷）。

### 3.2 受损模块

- Mango v3 program (Solana Rust program)，`lib.rs` health check / borrow 逻辑
- 预言机：Pyth Network MNGO/USD（聚合器）、Switchboard 备选
- MNGO-PERP 市场

### 3.3 关键逻辑（伪代码）

```rust
// 简化：Mango v3 的健康值计算
// 价格来源于 Pyth 聚合器
fn account_health(acc: &MangoAccount, cache: &OracleCache) -> I80F48 {
    let mut health = I80F48::ZERO;
    for pos in acc.spot_positions() { /* 现货抵押 × price */ }
    for pos in acc.perp_positions() {
        let mark_price = cache.price(pos.market.oracle); // Pyth MNGO/USD
        // 未实现盈亏直接纳入 health
        let upnl = (mark_price - pos.avg_entry) * pos.base_size;
        health += upnl * init_asset_weight;              // <-- 关键：upnl 作抵押
    }
    health
}

fn borrow(ctx, amount, token) {
    let h = account_health(&ctx.accounts.mango_account, &cache);
    require!(h >= amount * liquidity_factor, NotEnoughCollateral);
    // 放款
    transfer_from_vault(token, amount, ctx.accounts.user);
}
```

**错在哪**：
- `account_health` 把**永续未实现盈亏** (upnl) 直接作为可借抵押品；
- upnl 取决于 `mark_price` = 预言机 MNGO/USD 即时价；
- Pyth 聚合的 MNGO/USD 源于 Serum / FTX / AscendEX 等**低流动性现货市场**的中位数，被几百万美元现货买单即可拉升 30x；
- `init_asset_weight` 对 MNGO 未做"高度操纵风险"调降（事件时权重过于宽松）；
- 无单市场借款上限 / 无冷却期：upnl 立即可借光全库。

### 3.4 攻击步骤分解

1. **分账户**：Eisenberg 在 Mango 创建账户 A（做空）、账户 B（做多），各存入约 $5M USDC。
2. **自成交建多/空 PERP**：在 Mango PERP orderbook 上，A 挂卖、B 挂买，成交 **483M MNGO-PERP**。此步骤 A/B 对冲 UPNL = 0，但双方都有巨额名义仓位。
3. **现货拉盘**：从 AscendEX、Mango 现货、Serum 等低流动性市场用 USDC 买入 MNGO，把价格从 $0.03 拉至 $0.91（约 30x，15 分钟内）。
4. **Pyth 聚合**：由于 AscendEX、FTX、Serum 等喂价源跟随，Pyth MNGO/USD 聚合价进入 $0.80–$0.91 区间。
5. **账户 B upnl 爆炸**：483M MNGO × ($0.91 - $0.038) ≈ $421M 账面未实现盈利。
6. **借空金库**：账户 B 借出 USDC、SOL、mSOL、BTC、ETH、USDT、MSOL 等合计价值 $117M，并把资产转出 Mango 账户到其他钱包。
7. **账户 A 留守**：账户 A 的 PERP 空头保证金被爆仓，但由于金库已被借空，无法实际清算出等值偿付。
8. **曝光**：Eisenberg 对外承认，开启 Settlement 谈判。

### 3.5 为何审计未发现

- OtterSec 等对 Mango v3 代码的审计覆盖重入、账户校验、权限等"常规"漏洞，但**保证金经济学缺陷**被归为治理参数；
- 预言机操纵抵抗是 `DEX + Perp` 架构的经典问题，事件时没有 TWAP / confidence-interval 保护；
- Pyth 当时未对低流动性资产打分，Mango 未对 MNGO 做 self-listing 限制。

## 4. 事后响应

### 4.1 项目方动作

- 暂停 borrow/lend；
- 2022-10-15 社区投票通过 Settlement：返还 ~$67M 资产；
- 2022-11 公布 Mango v4 roadmap：修正保证金权重、引入 listing asset 更严格 oracle confidence 门槛。

### 4.2 资产追回

- DAO 获 ~$67M；
- 美国 SDNY 提起刑事诉讼后冻结 Eisenberg 部分加密资产；
- 截至 2025-01 判决时，民事赔偿令要求返还剩余金额。

### 4.3 法律 / 执法

- **2022-12-27**：FBI 波多黎各逮捕 Eisenberg；
- **2024-04-18**：陪审团裁定三项罪名（商品操纵、商品欺诈、电信欺诈）成立；
- **2025-01-30**：SDNY 宣判约 **78 月（6.5 年）** 监禁；
- CFTC、SEC 并行民事诉讼。**是链上经济学攻击被联邦刑事定罪的首批标志性案例。**

### 4.4 复查与审计

- Mango v4 改用 Pyth 的 `confidence interval` 过滤，引入权重分级；
- OtterSec 发文呼吁所有 Solana perp 协议引入 TWAP 喂价或 confidence-band。

## 5. 启发与教训

### 5.1 对开发者

- **UPNL 作抵押必须 throttled**：不可完全按 mark price 即时采信；加最大 haircut（如 20-50%）或 per-asset 白名单；
- 预言机必须使用 **confidence interval / TWAP / 多源中位**，抗 30 分钟窗口内操纵；
- 单资产 / 单市场借款上限必须硬编码到合约；
- 自成交（wash trading）检测虽难在链上做，但保证金模型不应鼓励此行为。

### 5.2 对审计方

- 审计不能止步于合约正确性；应包含**保证金模型攻击者视角**模拟（如"如果 oracle 上涨 10x，我能借多少？"）；
- 构建场景化 fuzzing：`oracle_price = 30× true_price` 时 `borrow_limit(account) < TVL`？

### 5.3 对用户

- 使用 perp 平台前查预言机源头 / 权重：MNGO/USD 来自 AscendEX 等低流动性市场 = 高风险信号；
- 协议代币充当抵押品本身是高风险模式。

### 5.4 对协议

- 治理代币禁止作为大额借款抵押；
- 快速发行资产的 oracle 权重要分级；
- 社区 Settlement 非长久之计，应建立法律追溯渠道。

## 6. 参考资料

- Mango 官方 post-mortem：https://blog.mango.markets/mango-incident-post-mortem-2c1c6b0c3b94
- rekt.news：https://rekt.news/mango-markets-rekt/
- DOJ 宣判新闻稿：https://www.justice.gov/usao-sdny/pr/avraham-eisenberg-found-guilty-manipulating-decentralized-cryptocurrency-exchange
- Eisenberg 公开推特承认：https://twitter.com/avi_eisen/status/1580191065781366784
- CertiK 分析：https://www.certik.com/resources/blog/mango-markets-incident-analysis
- 攻击者 Solana 账户：`CQvKSNnYtPTZfQRQ5jkHq8q2sJHrZTvZbjZ3u2A7GMGN`
- Eisenberg 关联 Ethereum 地址：`0x57452ADa44Bd9c1f5bA88B5BE8147bCF4A2d6A06`

---

*Last verified: 2026-04-23*
