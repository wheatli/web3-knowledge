---
title: Web3 安全事件大档案（2011–2026 YTD）
module: 08-security-incidents
priority: P0
status: DRAFT
last_verified: 2026-04-23
word_count_target: 3500
primary_sources:
  - https://rekt.news/leaderboard/
  - https://hacked.slowmist.io
  - https://skynet.certik.com
  - https://twitter.com/PeckShieldAlert
  - https://blocksec.com/incidents
  - https://github.com/SunWeb3Sec/DeFiHackLabs
  - https://www.chainalysis.com/crypto-crime-report/
  - https://cyvers.ai
---

# Web3 安全事件大档案（2011–2026 YTD）

> **TL;DR**：本章是 Web3 行业自 2011 年 Mt.Gox 首破到 2026 年 4 月截止 **时间线完整收录的重大安全事件档案**，共 **~55 篇** 独立事件深度复盘（每篇独立 md，统一模板：攻击时间线 / 根因 / 链上痕迹 / 资金流向 / 归因 / 修复与教训）。与 `05-security/` 的教学式综述形成互补：**05 按"漏洞模式"横切，08 按"事件时间线"纵切**。**累计被盗规模**（按 Chainalysis / rekt.news 口径）：2022 峰值 $38 亿，2024 回升至 $22 亿，2025 因 Bybit 单笔 **$14.6 亿**（史上最大单笔被盗）再创纪录。**归因维度**：Lazarus Group（朝鲜 TraderTraitor 子团体）自 2022 年起主导 CEX/托管类大额事件，占 2024–2025 年损失的 ~60%。

---

## 1. 本章定位 vs `05-security/`

| 维度 | `05-security/` | `08-security-incidents/` |
|------|----------------|--------------------------|
| 组织方式 | 按漏洞**模式/类别**（桥攻击、DeFi 漏洞、审计方法论等） | 按事件**时间线**（2011 → 2026，每起独立 md） |
| 目标读者 | 想系统学习"有哪几类漏洞、如何防范"的开发者/审计员 | 想查某一具体事件细节（资金流、归因、链上 tx）的研究员/合规/记者 |
| 深度 | 跨事件**模式综述**（每类 5–8 个典型案例对比） | 每起事件**独立深度复盘**（时间线精确到分钟，含 tx hash） |
| 典型文件 | `bridge-hack-postmortems.md` 综述 Ronin/Nomad/Wormhole/Poly/Harmony/Multichain | `2022-ronin.md`、`2022-nomad.md` … 每个事件独立 |
| 更新频率 | 季度性补充新模式 | 实时追加（PeckShield/BlockSec alert → 72h 内建档） |

**互链**：
- 桥事件跨事件模式综述 → [`05-security/bridge-hack-postmortems.md`](../05-security/bridge-hack-postmortems.md)
- DeFi 协议跨事件模式综述 → [`05-security/defi-exploit-postmortems.md`](../05-security/defi-exploit-postmortems.md)
- 合约漏洞分类词典 → [`05-security/contract-vulnerabilities.md`](../05-security/contract-vulnerabilities.md)

---

## 2. 时间线完整档案（55 events）

> Tier 分档：**T1** = 单笔损失 ≥ $100M 或行业级影响；**T2** = $10M–$100M 或机制新颖；**T3** = < $10M 但具代表性（新型攻击载体）。金额按事发当日 USD 计。

### 2011–2019（13 events）

| 年份 | 事件 | 损失 (USD) | 类别 | Tier | 档案 |
|------|------|-----------|------|------|------|
| 2011 | Mt.Gox First Breach | ~$8.75M | CEX / Key-Mgmt | T3 | [2011-mtgox-first-breach](./2011-mtgox-first-breach.md) |
| 2012 | Linode / Bitcoinica | ~$230k | Supply-Chain | T3 | [2012-linode-bitcoinica](./2012-linode-bitcoinica.md) |
| 2014 | Mt.Gox Collapse | 850k BTC (~$460M 当时) | CEX / Key-Mgmt | T1 | [2014-mtgox-collapse](./2014-mtgox-collapse.md) |
| 2015 | Bitstamp | ~$5M | CEX / Social-Eng | T3 | [2015-bitstamp](./2015-bitstamp.md) |
| 2016 | The DAO | ~$60M (3.6M ETH) | DeFi / 逻辑 | T1 | [2016-the-dao](./2016-the-dao.md) |
| 2016 | Bitfinex | ~$72M (120k BTC) | CEX / Multisig | T2 | [2016-bitfinex](./2016-bitfinex.md) |
| 2017 | Parity Multisig Hack | ~$30M (150k ETH) | Wallet | T2 | [2017-parity-wallet-hack](./2017-parity-wallet-hack.md) |
| 2017 | Parity Freeze (accidental) | 513k ETH 永久冻结 | Wallet / 逻辑 | T1 | [2017-parity-freeze](./2017-parity-freeze.md) |
| 2017 | CoinDash ICO DNS Hijack | ~$7M | Social-Eng | T3 | [2017-coindash-ico-dns](./2017-coindash-ico-dns.md) |
| 2018 | Coincheck NEM | ~$534M | CEX / Key-Mgmt | T1 | [2018-coincheck-nem](./2018-coincheck-nem.md) |
| 2018 | BitGrail (NANO) | ~$170M | CEX | T3 | [2018-bitgrail](./2018-bitgrail.md) |
| 2019 | Binance 7k BTC | ~$40M | CEX / Social-Eng | T2 | [2019-binance-7k-btc](./2019-binance-7k-btc.md) |
| 2019 | Upbit | ~$49M | CEX | T2 | [2019-upbit](./2019-upbit.md) |

### 2020–2021（10 events）

| 年份 | 事件 | 损失 | 类别 | Tier | 档案 |
|------|------|-----------|------|------|------|
| 2020 | LendfMe Reentrancy | $25M | DeFi / 重入 | T2 | [2020-lendfme-reentrancy](./2020-lendfme-reentrancy.md) |
| 2020 | bZx Flashloan Series | $8M+$8M+$55M | DeFi / Oracle | T2 | [2020-bzx-flashloan-series](./2020-bzx-flashloan-series.md) |
| 2020 | Harvest Finance | $24M | DeFi / Oracle | T2 | [2020-harvest-finance](./2020-harvest-finance.md) |
| 2020 | KuCoin | ~$281M | CEX / Key-Mgmt | T1 | [2020-kucoin](./2020-kucoin.md) |
| 2021 | Poly Network | $611M | Bridge / 权限 | T1 | [2021-polynetwork](./2021-polynetwork.md) |
| 2021 | BadgerDAO | $120M | Supply-Chain / 前端 | T1 | [2021-badgerdao](./2021-badgerdao.md) |
| 2021 | Cream Finance | $130M | DeFi / Oracle | T1 | [2021-cream-finance](./2021-cream-finance.md) |
| 2021 | PancakeBunny | $45M | DeFi / Oracle | T2 | [2021-pancakebunny](./2021-pancakebunny.md) |
| 2021 | BitMart | $196M | CEX / Key-Mgmt | T2 | [2021-bitmart](./2021-bitmart.md) |
| 2021 | Vulcan Forged | $140M (大部分追回) | GameFi / Key-Mgmt | T3 | [2021-vulcan-forged](./2021-vulcan-forged.md) |

### 2022（9 events）

| 年份 | 事件 | 损失 | 类别 | Tier | 档案 |
|------|------|-----------|------|------|------|
| 2022 | Ronin Bridge | $625M | Bridge / Multisig | T1 | [2022-ronin](./2022-ronin.md) |
| 2022 | Wormhole | $325M | Bridge / 签名验证 | T1 | [2022-wormhole](./2022-wormhole.md) |
| 2022 | Nomad | $190M | Bridge / 初始化 | T1 | [2022-nomad](./2022-nomad.md) |
| 2022 | Beanstalk Governance | $182M | Governance / Timelock | T1 | [2022-beanstalk-governance](./2022-beanstalk-governance.md) |
| 2022 | Harmony Horizon | $100M | Bridge / Multisig | T2 | [2022-harmony-horizon](./2022-harmony-horizon.md) |
| 2022 | Mango Markets | $117M | DeFi / Oracle | T2 | [2022-mango-markets](./2022-mango-markets.md) |
| 2022 | BNB Chain Bridge | $570M (冻结 $430M) | Bridge / IAVL proof | T1 | [2022-bnb-chain-bridge](./2022-bnb-chain-bridge.md) |
| 2022 | FTX Collapse | ~$477M (被黑) / $8B+ 总盘 | CEX / 欺诈+被黑 | T1 | [2022-ftx-collapse](./2022-ftx-collapse.md) |
| 2022 | Wintermute | $160M | Wallet / Profanity vanity | T2 | [2022-wintermute](./2022-wintermute.md) |

### 2023（8 events）

| 年份 | 事件 | 损失 | 类别 | Tier | 档案 |
|------|------|-----------|------|------|------|
| 2023 | Euler Finance | $197M (已归还) | DeFi / 逻辑 | T1 | [2023-euler](./2023-euler.md) |
| 2023 | Multichain | ~$126M | Bridge / Key-Mgmt | T1 | [2023-multichain](./2023-multichain.md) |
| 2023 | Mixin Network | $200M | CEX-like / 云DB | T1 | [2023-mixin](./2023-mixin.md) |
| 2023 | Curve Vyper Reentrancy | $73M (大部分追回) | DeFi / 编译器 | T1 | [2023-curve-vyper-reentrancy](./2023-curve-vyper-reentrancy.md) |
| 2023 | Atomic Wallet | $100M | Wallet / 未知 vector | T2 | [2023-atomic-wallet](./2023-atomic-wallet.md) |
| 2023 | Poloniex | $125M | CEX / Key-Mgmt | T2 | [2023-poloniex](./2023-poloniex.md) |
| 2023 | Ledger Connect Kit | ~$600k | Supply-Chain / CDN | T2 | [2023-ledger-connect-kit](./2023-ledger-connect-kit.md) |
| 2023 | HECO Bridge | $86M | Bridge / Key-Mgmt | T3 | [2023-heco-bridge](./2023-heco-bridge.md) |

### 2024（8 events）

| 年份 | 事件 | 损失 | 类别 | Tier | 档案 |
|------|------|-----------|------|------|------|
| 2024 | PlayDapp | $290M | Game / 权限 | T1 | [2024-playdapp](./2024-playdapp.md) |
| 2024 | DMM Bitcoin | $305M | CEX / Key-Mgmt | T1 | [2024-dmm-bitcoin](./2024-dmm-bitcoin.md) |
| 2024 | WazirX | $234M | CEX / 前端+Multisig | T1 | [2024-wazirx](./2024-wazirx.md) |
| 2024 | Radiant Capital | $58M | DeFi / Multisig 前端 | T2 | [2024-radiant-capital](./2024-radiant-capital.md) |
| 2024 | Orbit Chain | $82M | Bridge / Key-Mgmt | T2 | [2024-orbit-chain](./2024-orbit-chain.md) |
| 2024 | Munchables | $62M (全额归还) | Supply-Chain / 内鬼 | T3 | [2024-munchables](./2024-munchables.md) |
| 2024 | Penpie Reentrancy | $27M | DeFi / 重入 | T3 | [2024-penpie-reentrancy](./2024-penpie-reentrancy.md) |
| 2024 | Gala Games Mint | $21M (追回) | Game / 权限 | T2 | [2024-gala-games-mint](./2024-gala-games-mint.md) |

### 2025–2026（11 events）

| 年份 | 事件 | 损失 | 类别 | Tier | 档案 |
|------|------|-----------|------|------|------|
| 2025 | Bybit (史上最大) | **$1.46B** | CEX / UI 钓鱼 + Multisig | T1 | [2025-bybit](./2025-bybit.md) |
| 2025 | Abracadabra / Magic Internet Money | $13M | DeFi | T3 | [2025-abracadabra](./2025-abracadabra.md) |
| 2025 | Infini | $50M | DeFi/Stablecoin | T3 | [2025-infini](./2025-infini.md) |
| 2025 | zkLend | $9.6M | DeFi | T3 | [2025-zklend](./2025-zklend.md) |
| 2025 | Cetus (Sui) | $223M | DeFi / 流动性数学 | T1 | [2025-cetus-sui](./2025-cetus-sui.md) |
| 2025 | Mantra OM Collapse | ~$6B 市值蒸发 | Token / 流动性 | T2 | [2025-mantra-om-collapse](./2025-mantra-om-collapse.md) |
| 2025 | Hyperliquid JELLY | ~$12M HLP 损失 | Perp / Oracle | T2 | [2025-hyperliquid-jelly](./2025-hyperliquid-jelly.md) |
| 2025 | Phemex | $85M | CEX / Key-Mgmt | T2 | [2025-phemex](./2025-phemex.md) |
| 2025 | Four.meme | ~$3M | DeFi / Launchpad | T3 | [2025-four-meme](./2025-four-meme.md) |
| 2025 | KiloEx | ~$7M | DeFi / Perp Oracle | T3 | [2025-kiloex](./2025-kiloex.md) |
| 2026 | YTD Notable Incidents (Q1–Q2) | 聚合 | 多类 | N/A | [2026-ytd-notable-incidents](./2026-ytd-notable-incidents.md) |

---

## 3. Top-10 单笔损失榜（2011–2026 YTD）

> 按事发当日 USD 计价。被追回/归还不影响该榜（表征为"事件量级"）。

| # | 事件 | 年份 | 金额 | 类别 |
|---|------|------|------|------|
| 1 | **Mt.Gox Collapse** | 2014 | 850k BTC（后续牛市估值 $45B+） | CEX |
| 2 | **FTX Collapse** | 2022 | $8B+ 客户资金 / 被黑 $477M | CEX / 欺诈 |
| 3 | **Bybit** | 2025 | $1.46B | CEX / UI 钓鱼 |
| 4 | **Ronin Bridge** | 2022 | $625M | Bridge |
| 5 | **Poly Network** | 2021 | $611M（全部归还） | Bridge |
| 6 | **BNB Chain Bridge** | 2022 | $570M（冻结 $430M） | Bridge |
| 7 | **Coincheck NEM** | 2018 | $534M | CEX |
| 8 | **Wormhole** | 2022 | $325M | Bridge |
| 9 | **DMM Bitcoin** | 2024 | $305M | CEX |
| 10 | **PlayDapp** | 2024 | $290M | Game / 权限 |

（紧随其后：KuCoin $281M 2020、Cetus $223M 2025、Mixin $200M 2023、BitMart $196M 2021、Euler $197M 2023、Nomad $190M 2022、Beanstalk $182M 2022 ……）

---

## 4. 分类 × 年份矩阵

> 数字为本档案内收录事件数（非行业总数）。

| 类别 \ 年 | 2011–15 | 2016 | 2017 | 2018 | 2019 | 2020 | 2021 | 2022 | 2023 | 2024 | 2025–26 | 合计 |
|-----------|---------|------|------|------|------|------|------|------|------|------|---------|------|
| CEX       | 3       | 1    | 0    | 2    | 2    | 1    | 1    | 1    | 2    | 2    | 2       | 17   |
| Bridge    | 0       | 0    | 0    | 0    | 0    | 0    | 1    | 4    | 2    | 1    | 0       | 8    |
| DeFi      | 0       | 1    | 0    | 0    | 0    | 3    | 2    | 1    | 2    | 1    | 6       | 16   |
| Oracle    | 0       | 0    | 0    | 0    | 0    | 3    | 2    | 1    | 0    | 0    | 2       | 8    |
| Wallet    | 0       | 0    | 2    | 0    | 0    | 0    | 0    | 1    | 1    | 0    | 0       | 4    |
| Governance| 0       | 0    | 0    | 0    | 0    | 0    | 0    | 1    | 0    | 0    | 0       | 1    |
| Social-Eng| 1       | 0    | 1    | 0    | 1    | 0    | 0    | 0    | 0    | 1    | 1       | 5    |
| Supply-Chain | 1    | 0    | 0    | 0    | 0    | 0    | 1    | 0    | 1    | 1    | 0       | 4    |
| Key-Mgmt  | 2       | 1    | 0    | 2    | 1    | 1    | 2    | 0    | 3    | 2    | 1       | 15   |

（注：一起事件可能同时归入多个类别，如 Ronin 同时 Bridge+Multisig+Social-Eng，矩阵按"主因"单计一次。）

---

## 5. 年度损失趋势（Chainalysis 口径，含本档案外事件）

```
USD (Billion)
4.0 |                                      ██                        
3.8 |                                      ██                        
3.5 |                                      ██                        
3.0 |                        ██            ██                      ██
2.5 |                        ██            ██            ██        ██
2.2 |                        ██     ██     ██     ██     ██        ██
2.0 |                        ██     ██     ██     ██     ██     ██ ██
1.5 |                 ██     ██     ██     ██     ██     ██     ██ ██
1.0 |   ██     ██     ██     ██     ██     ██     ██     ██     ██ ██
0.5 |   ██     ██     ██     ██     ██     ██     ██     ██     ██ ██
    +---2014--2015--2016--2017--2018--2019--2020--2021--2022--2023-2024-2025(YTD)
```

**关键节点**：
- **2022 峰值**（$38 亿）：Ronin+Wormhole+Nomad+BNB Bridge+FTX 叠加
- **2023 低谷**（$17 亿）：熊市 + 大额追回（Euler、Poly）
- **2024 回升**（$22 亿）：CEX 重新成为主战场（DMM/WazirX/PlayDapp）
- **2025 YTD**（截至 Q1 约 $18 亿）：Bybit 一起即占 80%

---

## 6. 头部攻击者归因：Lazarus Group（TraderTraitor）

> 仅收录 **Chainalysis / FBI / ZachXBT** 正式归因过的事件。参考：FBI IC3 公告、Chainalysis 2024/2025 Crypto Crime Report、ZachXBT investigation thread。

| 年份 | 事件 | 金额 | 归因来源 |
|------|------|------|---------|
| 2022 | Ronin Bridge | $625M | FBI 2022-04-14 官方公告、OFAC SDN list |
| 2022 | Harmony Horizon | $100M | FBI 2023-01-23 公告 |
| 2023 | Atomic Wallet | $100M | Elliptic / ZachXBT 资金链追踪至 Ronin 混合钱包 |
| 2023 | CoinEx / Stake.com / Alphapo | 合计 ~$200M | Chainalysis 2024 Crypto Crime Report |
| 2024 | DMM Bitcoin | $305M | FBI + Japan NPA 2024-12-23 联合公告 |
| 2024 | WazirX | $234M | Elliptic / ZachXBT 地址关联 |
| 2024 | Radiant Capital | $58M | Mandiant / Radiant post-mortem 2024-12 |
| 2025 | Bybit | $1.46B | FBI 2025-02-26 公告（TraderTraitor 子团体） |

**模式特征**：
1. 针对 **托管 / Multisig / DeFi 前端** 的长期钓鱼 / LinkedIn 假面试 / 恶意 npm 包；
2. 资金在 1–2 小时内切 THORChain/Chainflip → BTC → Mixer；
3. 持有期 6–24 个月后通过 OTC 市场出金，偏好泰国/柬埔寨柜台。

---

## 7. 学习主题 Top 10（每类给出代表事件）

1. **初始化漏洞**：`initialize` 未加 `onlyOwner` 或重入式 init → **Nomad 2022**（acceptableRoot[0]=0）、**Parity 2017-11 freeze**。
2. **中心化多签**：阈值过低 / 密钥集中运营端 → **Ronin 2022** (5/9)、**Harmony 2022** (2/5)、**Multichain 2023** (CEO 独控)。
3. **预言机闪贷**：spot price 被单池操纵 → **bZx 2020**、**Harvest 2020**、**Cream 2021**、**Mango 2022**、**KiloEx 2025**。
4. **治理无 timelock**：提案即刻生效 → **Beanstalk 2022**（闪贷 66% vote → 清空金库）。
5. **供应链攻击**：前端 / CDN / npm / 内鬼 → **BadgerDAO 2021**（Cloudflare）、**Ledger Connect Kit 2023**（npm phish）、**Munchables 2024**（朝鲜内鬼开发）。
6. **编译器 bug**：语言层 `@nonreentrant` 失效 → **Curve Vyper 0.2.15/0.2.16/0.3.0 2023**。
7. **权限隔离失败**：跨合约权限未白名单化 → **Poly 2021**（EthCrossChainManager owner 被替换）、**PlayDapp 2024**（SuperMinter 被授权）、**Gala 2024**（mint 权限泄露）。
8. **前端钓鱼 / UI 欺骗**：Multisig 签名 UI 被伪造 → **Bybit 2025**（Safe{Wallet} JS 被污染）、**WazirX 2024**、**Radiant 2024**。
9. **TEE / 托管单点**：Key shard 集中于云/人员 → **Coincheck 2018**（热钱包未分片）、**Bitfinex 2016**（BitGo 多签 co-signer 绕过）、**DMM 2024**、**Mixin 2023**（云 DB 被拖）。
10. **逻辑一致性**：同一状态两套读法 → **Euler 2023**（`donateToReserves` vs `liquidate` 健康因子差异）、**The DAO 2016**（splitDAO 重入前状态未结算）。

---

## 8. 使用指南：按漏洞类别快速检索

**场景 A：审计员做新代码库的威胁建模**
→ 先翻 §7 学习主题，对照代码里是否存在对应模式；再到 §4 分类矩阵锁定同类历史事件；最后读对应年份的独立 md 档案。

**场景 B：合规/KYT 团队做 SAR 报告**
→ 先到 §6 Lazarus 归因表定位朝鲜资金；再读具体事件 md 中的"资金流向"章节（链上 tx hash、mixer hop、OTC 出金点）。

**场景 C：研究员做行业回顾**
→ 先看 §2 时间线 + §3 Top-10 + §5 年度趋势，把握宏观；再按 Tier 分档抽样精读（T1 事件是必读）。

**场景 D：开发者做新协议的"别踩的坑"清单**
→ 直接按 §7 学习主题逐条过。每个主题下链接到 2–3 个独立 md 深读。

---

## 9. 参考源

### 事件档案专用源

- **SlowMist Hacked 数据库** — <https://hacked.slowmist.io>（中文友好、慢雾 AML 自研归因）
- **CertiK Skynet** — <https://skynet.certik.com>（实时监控 + 风险评分）
- **PeckShield Alert** — <https://twitter.com/PeckShieldAlert>（事发后 <15 min 首发）
- **BlockSec Incidents** — <https://blocksec.com/incidents>（MetaSleuth 资金流追踪）
- **rekt.news leaderboard** — <https://rekt.news/leaderboard/>（英文深度复盘 + 金额排行）
- **DeFiHackLabs** — <https://github.com/SunWeb3Sec/DeFiHackLabs>（可复现 Foundry POC）
- **Chainalysis Crypto Crime Report**（年度）— <https://www.chainalysis.com/crypto-crime-report/>
- **Cyvers Alerts** — <https://cyvers.ai>（实时 ML 风险信号）

### 归因 / 执法源

- **FBI Internet Crime Complaint Center (IC3)** 公告
- **OFAC SDN List**（朝鲜 / 伊朗制裁地址）
- **ZachXBT investigation thread** — `twitter.com/zachxbt`
- **Mandiant / TRM Labs / Elliptic** 归因报告

### 交叉验证（Tier 2）

- 受害项目官方 post-mortem（优先级 > 媒体报道）
- Etherscan / Solscan / Arkham 链上证据
- Uniswap / Curve DAO forum 讨论帖

---

**维护约定**：每起新 T1 事件在 72h 内建档（至少含时间线 + 已知金额 + tx hash 占位）；T2/T3 每月月末汇总一次。所有金额标注 **事发当日 USD** 与 **查询日期**。归因主张必须附 Tier 1 源（FBI/OFAC/项目方/Chainalysis 正式报告），否则只写"链上地址表现与某事件钱包簇关联"。
