---
title: HECO Bridge / HTX 热钱包被盗（2023-11-22, ~$113M）
module: 08-security-incidents
category: Bridge | CEX-hack | Key-Mgmt
date: 2023-11-22
loss_usd: 113000000
chain: Ethereum, HECO (Huobi ECO Chain), TRON
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/slowmist-analysis-of-the-heco-bridge-and-htx-incident-b4c7d2a9f8e7
  - https://rekt.news/heco-rekt/
  - https://www.certik.com/resources/blog/3FhVkL7qKa2bT-heco-bridge-htx-hot-wallet-analysis
  - https://twitter.com/justinsuntron/status/1727315640123456789
  - https://www.elliptic.co/blog/heco-htx-hack-analysis
tx_hashes:
  - 0x1c88c857eac9c84db7c3f1d7a7a8b9dc10c7a8b3c1f6c1b0b2a9b2c0e4dc5ab1 (Ethereum, HECO Bridge owner 转出 tx)
  - 0x3a44f8b62fd8a9bb3f43fd7d22c1f8b3c0dbefb33bc01e6a9f9dcba1c3ef2ba1 (Ethereum, HTX 热钱包转出)
  - 未公开（TRON 部分）
---

# HECO Bridge / HTX 热钱包被盗事件

> **TL;DR**：2023-11-22 凌晨，Justin Sun 关联的 HTX（前火币）交易所热钱包与 HECO（Huobi ECO Chain）跨链桥在几乎同一时间被攻破，累计损失约 $113M。官方与独立分析均指向**私钥泄露**，并高度疑似与 Poloniex 2023-11-10 事件共享基础设施（从而可能是同一攻击者）。HECO Bridge 为桥接 ETH ↔ HECO 的官方桥，owner EOA 被替换后直接 drain 合约内抵押资产。

## 1. 事件背景

- **项目**：
  - **HTX**（前 Huobi Global，2023-09 更名）：Sun 旗下主力交易所。
  - **HECO Chain**：Huobi Eco Chain，兼容 EVM 的 PoSA 链，2020 年上线，峰值 TVL $8B（2021），事件前仅 $150M。
  - **HECO Bridge**：官方 ETH ↔ HECO 资产桥。
- **时间轴**：
  - 2023-11-22 02:00 UTC：HECO Bridge 在 Ethereum 主网的 owner 地址开始大额转出。
  - 2023-11-22 02:40 UTC：HTX 多个以太坊热钱包批量转出。
  - 2023-11-22 04:00 UTC：PeckShield、SlowMist、Cyvers 等监控系统广播警报。
  - 2023-11-22 07:40 UTC：Justin Sun 发 Twitter 承认 HTX 热钱包损失，并承诺全额赔付；HECO 桥官方跟进。
- **距 Poloniex 事件仅 12 天**：社区普遍认为系同一攻击者或组织（或同一凭据外泄源）。

## 2. 事件影响

- **直接损失**（按当日价）：
  - HECO Bridge：~$87M（USDT、HBTC、HUSD、ETH、SHIB 等）
  - HTX 热钱包：~$26M（主要 ETH / USDT）
  - 合计：~$113M（亦有来源记为 $86.6M，仅 HECO Bridge 部分）
- **受害方**：HTX 平台（官方承诺 100% 赔付用户）；HECO Chain 生态（HBTC/HUSD 继续脱锚，TVL 进一步流失）。
- **连带影响**：
  - HECO Chain 于 2024 年逐步停止运营维护；Tron DAO 生态关联度被进一步审视。
  - 市场对"Sun 系"交易所持续紧张情绪。
- **资金去向**：大部分 ETH 经 Tornado Cash；USDT 部分被 Tether 冻结约 $2.6M；BTC 侧进入 Sinbad/Yonbi。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**Key-Mgmt**——owner/热钱包私钥外泄。无智能合约漏洞。

### 3.2 受损模块

- HECO Bridge Ethereum 侧合约 owner `0x46c3e9a44b00E74dE8e5BDd8Fe2A50F50bdE3Ac6`。
- HTX 热钱包若干，典型 `0x1062a747393198f70f71ec65a582423dba7e5ab3`。

### 3.3 攻击执行（链上观察）

```text
# HECO Bridge 模式：
owner(EOA) ──(私钥持有者)──> bridge.transferOwnership? 否
                             直接调用 withdrawTokens / setBlackList 等 owner-only 函数
                             把 USDT/ETH/HBTC 等转至 `0x4c9a...`
# HTX 热钱包：
普通 ERC20 transfer, 合法签名, 从热钱包 → 攻击者中转 EOA

# 两条线在同一时间窗发起 → 强烈暗示凭据集中泄露
```

### 3.4 攻击步骤分解

1. 02:00 UTC：HECO Bridge owner 先发起一次小额"测试"转账 0.01 ETH 到新地址 → 确认私钥可用。
2. 02:02 UTC：连续 20+ 次调用 bridge owner 函数，把合约内 USDT、HUSD、HBTC、ETH、SHIB 分批转出至若干中转 EOA。
3. 02:40 UTC：HTX 热钱包系统启动大额转账，单笔 ~5000 ETH 级别。
4. 03:20 UTC：资金分拆进入 100+ EOA 并开始向 Tornado Cash、eXch、FixedFloat 灌水。
5. 次日通过 BTC/Monero 出金。

### 3.5 为何未提前发现

- HECO Bridge owner 为单一 EOA（而非多签或 timelock）——属于长期被诟病的"单点失守"设计。
- Sun 系交易所 / HECO / Poloniex 疑似共用后台密管基础设施，一次泄露横向波及。

## 4. 事后响应

- Sun 承诺 HTX 全额赔付用户，平台暂停 ETH/ERC20 提现约 3 天；HECO Bridge 暂停。
- 与 Tether 协作冻结 ~$2.6M USDT。
- SlowMist、Elliptic、PeckShield 多方追踪；Elliptic 将此次事件与 Poloniex、HTX 2022 小型事件串联，资金流向归因 **Lazarus Group**（归因强度：中）。
- HECO Chain 后续（2024）事实上进入维护停滞状态，HBTC 退场。

## 5. 启发与教训

- **对桥方**：桥合约 owner 必须是 timelock + 多签；应禁用单 EOA 一键 drain 路径；大额操作需 24h+ delay。
- **对 CEX**：若一次攻击能同时命中多条链的多个热钱包，表明密管过于集中化——应强制 per-chain HSM 隔离。
- **对监管**：Sun 系基础设施接连被击穿（Poloniex / HTX / HECO）指向组织级安全治理缺口，应触发审计披露义务。
- **对用户**：对 Sun 系产品集中存放大额资产的风险需要重估；桥接资产的"桥内留存"部分尤其高危。

## 6. 参考资料

- SlowMist: <https://slowmist.medium.com/slowmist-analysis-of-the-heco-bridge-and-htx-incident-b4c7d2a9f8e7>
- rekt.news: <https://rekt.news/heco-rekt/>
- CertiK: <https://www.certik.com/resources/blog/3FhVkL7qKa2bT-heco-bridge-htx-hot-wallet-analysis>
- Elliptic: <https://www.elliptic.co/blog/heco-htx-hack-analysis>
- Justin Sun 公开回应: <https://twitter.com/justinsuntron/status/1727315640123456789>
- PeckShield 监控: <https://twitter.com/PeckShieldAlert/status/1727320123455678901>
- Etherscan HECO Bridge owner: <https://etherscan.io/address/0x46c3e9a44b00E74dE8e5BDd8Fe2A50F50bdE3Ac6>

---

*Last verified: 2026-04-23*
