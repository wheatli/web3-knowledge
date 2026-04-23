---
title: FTX 交易所倒闭与并发链上盗窃事件（2022-11, ~$8B 客户资金缺口 + ~$477M 被盗）
module: 08-security-incidents
category: CEX-hack
date: 2022-11-11
loss_usd: 477000000
chain: [Ethereum, Solana, BSC]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://pacer.uscourts.gov/  # In re FTX Trading Ltd. Case No. 22-11068 (Bankr. D. Del.)
  - https://www.elliptic.co/blog/ftx-hack-investigation
  - https://rekt.news/ftx-rekt/
  - https://www.justice.gov/usao-sdny/pr/samuel-bankman-fried-found-guilty-seven-counts
  - https://www.chainalysis.com/blog/ftx-exploit-lazarus-group/
tx_hashes:
  - Ethereum 攻击者主地址: 0x59ABf3837Fa962d6853b4Cc0a19513AA031fd32b
  - 2022-11-11 多笔 FTX hot wallet 出账 tx（Etherscan 攻击者账户下有 300+ tx）
  - Solana 攻击者账户与 SPL token 清扫交易签名（Solscan 归档）
---

# FTX 交易所倒闭与并发链上盗窃事件

> **TL;DR**：2022-11-11，全球第二大加密交易所 FTX 在流动性挤兑与储备造假暴露后申请 Chapter 11 破产，当日深夜（UTC 晚）其 hot wallet 遭未授权提款约 **$477M**（ETH、ERC-20、Solana 生态资产等混合）。两起事件并行发生：商业崩塌由客户资金挪用至姐妹公司 Alameda 所致（2023-11-02 SBF 被纽约南区陪审团裁定 7 项罪名全部成立，2024-03 判 25 年监禁）；而链上 $477M 的"黑客"身份在长达 2 年争议后，**2024-02 DOJ 指控前 FTX 员工、前 Bahamas 司法部 IT 与 Chainalysis 归因 Lazarus 的混合路径**——但**正式归因至今仍未完全公开**。本文聚焦链上盗窃技术事件，并给出商业崩塌背景。

## 1. 事件背景

### 1.1 FTX 与 Alameda

FTX Trading Ltd. 由 Sam Bankman-Fried (SBF) 于 2019 年创立，2021-2022 高峰期日成交超 $100B，估值 $32B。姐妹做市公司 Alameda Research 由 SBF 2017 年创立，在链上是 FTX 最大做市商。两家间的资金、用户资产混同是此次崩塌的核心。

### 1.2 时间轴

- **2022-11-02**：CoinDesk 报道 Alameda 资产负债表大部分为 FTX 自发代币 FTT，流动性脆弱；
- **2022-11-06**：Binance 宣布抛售其持有的 ~$580M FTT，市场挤兑启动；
- **2022-11-08**：FTX 停止提现；
- **2022-11-09**：Binance 拟收购 FTX 的 LOI 次日被撤销；
- **2022-11-10**：SBF 内部信承认缺口达 ~$8B；
- **2022-11-11 UTC 06:00 左右**：FTX Trading、Alameda 等 130+ 实体申请 Chapter 11 破产 (Bankr. D. Del. Case 22-11068)，John J. Ray III 接任 CEO；
- **2022-11-11 UTC 晚 ~22:00 起**：FTX Ethereum 与 Solana hot wallet 出现大规模异常出账，资产流向从未见过的外部地址；数小时内被盗 ~$477M；
- **2022-11-12**：@zachxbt、@tayvano_、Elliptic 等并行监控异常地址，FTX General Counsel Ryne Miller 在 Twitter 确认"未经授权的交易"；
- **2022-12-12**：SBF 在巴哈马被捕，次日被引渡美国；
- **2023-11-02**：SDNY 陪审团裁定 SBF 7 项罪名全部成立（电信欺诈、共谋、洗钱等）；
- **2024-03-28**：SBF 被判 **25 年监禁**；
- **2024-02**：DOJ 指控前 FTX 员工 Ross Rheingans-Yoo 的关联叙事，Chainalysis 2023-10 报告称攻击者地址与 Lazarus Group 有链上聚类关联（部分资产经 Sinbad.io、Railgun 洗净）；归因**仍有争议**。

### 1.3 发现过程

链上监控 @zachxbt、@tayvano_、Etherscan community 在 11-11 晚同时发现 FTX hot wallet 异常。FTX 破产团队 Sullivan & Cromwell 随后与 Chainalysis、Elliptic 联合取证。

## 2. 事件影响

### 2.1 直接损失（链上盗窃部分）

- Ethereum：约 **$280M** ERC-20 + ETH；
- Solana：约 **$100M** SOL + SPL（大量长尾 memecoin 被迅速抛售）；
- BSC：约 **$50M** BEP-20；
- 其余分散 Avalanche、Polygon 等；
- 合计名义 **~$477M**（按 2022-11-11 价格）。

### 2.2 商业崩塌损失

- 客户存款缺口约 **$8B**（FTX.com + FTX US 合计）；
- 100 万+ 客户被冻结资金；
- 投资人包括 Sequoia、SoftBank、Ontario Teachers 等数十亿美元股权归零。

### 2.3 受害方

- 全球散户与机构客户（破产表列约 900 万用户注册）；
- 链上受害：储备中用户 ETH、SOL、USDT 等；
- 市场情绪：BTC 从 $20,600 跌至 $15,600 低点（11-21）；Solana 跌超 50%；FTT 近乎归零。

### 2.4 连带影响

- BlockFi、Genesis、Voyager 等中心化信贷实体随之破产；
- 整个行业进入"加密寒冬"；监管收紧；
- 引发 "Proof-of-Reserves" 运动（Binance、Kraken、OKX 等公布 PoR）。

### 2.5 资金去向

- 攻击者 Ethereum 地址 `0x59ABf3837Fa962d6853b4Cc0a19513AA031fd32b` 在 2022-11-20 起通过 RenBridge 把 ETH 跨到 Bitcoin，然后进入 Sinbad.io 混币；
- 2023-01 Elliptic 报告：部分资金路径与 Ronin、Harmony 等 Lazarus 案件重合；
- 2023-10 Chainalysis 报告正式指出"可能与朝鲜相关 actor 有关"，但用词保留；
- 破产团队陆续追回部分稳定币（Tether 冻结数千万 USDT）。

## 3. 技术根因（链上盗窃部分，核心）

### 3.1 漏洞分类

**私钥 / 热钱包访问控制失守**（非智能合约漏洞）。可能路径：
1. 破产混乱期内部人非法转移；
2. 外部攻击者掌握 hot wallet key（通过入侵 FTX 基础设施或社工员工）；
3. Bahamas 当局在法律混乱期自行移动资产的说法（被 FTX 破产方公开否认）。

三类路径不互斥，DOJ / Chainalysis 追踪倾向于"外部 actor 主导 + 内部访问条件被利用"。

### 3.2 受损模块

- FTX 的多链 hot wallet 基础设施（内部自研 + Fireblocks 托管的混合架构，按破产文件披露以 AWS + HSM 存钥）；
- Ethereum hot wallet 群组：`0x2FAF487A4414Fe77e2327F0bf4AE2a264a776AD2`、`0xC098B2a3Aa256D2140208C3de6543aAEf5cd3A94` 等；Solana hot wallet：`5Q544fKrFoe6tsEbD7S8EmxGTJYAKtTVhAW5Q5pge4j1` 等多。
- Gas funding 账户 `0x59ABf3837Fa962d6853b4Cc0a19513AA031fd32b` 成攻击者主地址。

### 3.3 关键"代码"层面

本案无合约漏洞。关键问题是 **key management 操作流程**：
- Fireblocks 与内部系统未做最小权限 / 多签白名单；
- 破产启动时的混乱期中，多名高管持 admin 级别访问；
- 钱包批量提币的签名逻辑使用 hot wallet 单签 EOA，而非 MPC / multisig（部分钱包如此配置）。

### 3.4 攻击步骤分解

1. **立足**：攻击者（内部或外部）在 2022-11-11 UTC 晚已具备 FTX hot wallet 的签名访问（或其 Fireblocks 自动化 rule 被篡改）；
2. **批量扫币**：向数十个 FTX 用户 hot wallet 地址发出 `transfer` / `transferFrom`，目标地址 `0x59ABf...` 等；Solana 侧批量 SPL `transferChecked`；
3. **快速换币**：在 CowSwap、Uniswap、Curve 做成 ETH / stETH / DAI 等流动性好资产；
4. **Ren Bridge 跨 BTC**：几天内把 ETH 换成 renBTC 跨到 Bitcoin；
5. **Sinbad 混币**：2022-12 起多轮 Sinbad.io / Railgun 混币；
6. **长期静默**：部分资产至今仍未移动。

### 3.5 为何未被阻止

- 破产声明当夜 hot wallet 监控被切换 / 响应团队混乱；
- 部分 Ethereum hot wallet 是单签 EOA，钥匙在受控机器上，一旦机器失守立即空；
- 缺乏 token-level 提币上限 + 白名单；
- Fireblocks 的 TAP（Transaction Authorization Policy）未正确配置紧急冻结。

## 4. 事后响应

### 4.1 破产管理人动作

- 2022-11-12 起新 CEO John J. Ray III 委托 Sullivan & Cromwell 与链上取证团队（Chainalysis、TRM）追踪；
- 部分被盗 stablecoin 在 Tether、Circle、Paxos 协调下冻结；
- 2023-2024 逐步与用户分配剩余资产；
- 2024-10 Chapter 11 Plan 获批，用户按 USD 价值 2022-11-11 快照 + 利息赔付，多数用户 100%+ 回血。

### 4.2 资产追回

- 链上盗窃部分：冻结 + 混币残留中识别到约 10-15% 可追回；
- 商业崩塌部分：破产团队通过 Alameda 投资组合 (Anthropic 股份等) 变现，客户获得远高于最初预期的赔付。

### 4.3 法律 / 执法

- **2023-11-02**：SBF 七项罪名全部成立（包括电信欺诈、洗钱共谋等）；
- **2024-03-28**：25 年监禁 + $11B 没收令；
- Caroline Ellison、Gary Wang、Nishad Singh 等认罪合作；
- 2023-10 Chainalysis 正式提及 Lazarus 关联；FBI 未单独公开归因 FTX hack 为 Lazarus——**归因存在但未完整公开**。

### 4.4 复查与行业影响

- CEX 业界大规模推出 PoR（Merkle-tree 负债证明 + 链上储备），Binance / Kraken / OKX / Bitget 均在 2022-12 至 2023-01 间上线；
- Fireblocks、Copper 等 MPC 托管方案市占提升；
- 美国、欧盟对 CEX 加强 segregation of customer funds 要求（MiCA、SEC 多起 CEX 诉讼）。

## 5. 启发与教训

### 5.1 对开发者 / 交易所工程

- Hot wallet 必须使用 **MPC / 多签 + TAP 策略**，禁止单 EOA 热钱包托管大额用户资产；
- 紧急响应手册必须包含"破产 / 治理危机"情境：立即 lock admin、轮换 keys、启用地址白名单；
- Fireblocks 等托管方 TAP 规则需定期演练 + 单独的 emergency kill-switch。

### 5.2 对审计方

- 交易所审计不仅是合规，也应包括 **operational security review**：key 访问控制、员工权限、rotation；
- PoR 审计必须校验"资产真为交易所所有 + 负债全额披露"，避免 FTX 式挪用。

### 5.3 对用户

- "Not your keys, not your coins"；
- CEX 选择应看 PoR、监管管辖、多签 / MPC 架构；
- 避免在单一 CEX 集中超出自身可承受损失。

### 5.4 对协议 / 行业

- 客户资产 **segregation** 法制化（美国 SEC、CFTC 监管加码；欧盟 MiCA）；
- 市场营销 + VC 光环不是风控指标；
- 行业从"自律"转向"证明 + 监管"。

## 6. 参考资料

- FTX Chapter 11 法庭文件：https://pacer.uscourts.gov/ （Case 22-11068, Bankr. D. Del.）
- Elliptic 链上调查：https://www.elliptic.co/blog/ftx-hack-investigation
- rekt.news：https://rekt.news/ftx-rekt/
- DOJ SBF 判决：https://www.justice.gov/usao-sdny/pr/samuel-bankman-fried-found-guilty-seven-counts
- Chainalysis Lazarus 关联：https://www.chainalysis.com/blog/ftx-exploit-lazarus-group/
- Ryne Miller (FTX GC) 2022-11-12 Twitter 声明
- 攻击者 Ethereum 地址：`0x59ABf3837Fa962d6853b4Cc0a19513AA031fd32b`

---

*Last verified: 2026-04-23*
