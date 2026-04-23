---
title: Poloniex 交易所热钱包被盗（2023-11-10, ~$125M）
module: 08-security-incidents
category: CEX-hack | Key-Mgmt
date: 2023-11-10
loss_usd: 125000000
chain: Ethereum, TRON, Bitcoin
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/slowmist-analysis-of-the-poloniex-hack-5e8b0bbc7f0b
  - https://rekt.news/poloniex-rekt/
  - https://www.certik.com/resources/blog/5JWrmBZcXw3a7kT-poloniex-heist-analysis
  - https://twitter.com/justinsuntron/status/1723208812421902359
  - https://www.elliptic.co/blog/poloniex-hack-north-korea-lazarus
tx_hashes:
  - 0xb0e7c8c1c15d1abf8b0abdc8f6fac3d1bc97e14ee5a5a6f9c4c9b42f92c6dfce (Ethereum, USDT 批量转出)
  - 0x5cb63b7d3c09b0aa6f9bcbda8cd4c90e98bffddc1a7b7cc8fcff462d6e5cec0a (Ethereum, ETH 批量转出)
  - 未公开（TRON、BTC 大笔转出）
---

# Poloniex 交易所热钱包被盗

> **TL;DR**：2023-11-10，孙宇晨（Justin Sun）旗下的老牌交易所 Poloniex 热钱包被攻破，损失约 $125M（含 ETH/TRX/BTC/USDT 等），单笔事件列 2023 年第四大损失。根因为热钱包私钥外泄（官方未明确披露向量，疑似内部人员或运维服务器沦陷）。孙宇晨随即在 Twitter 发出 5% 白帽赏金并威胁追诉，后期约 $10M 被冻结/返还，绝大部分资金经混币沉没，Elliptic 将资金流向归因 Lazarus。

## 1. 事件背景

- **项目**：Poloniex，2014 年成立，美国老牌加密交易所；2019 年被 Circle 剥离后转入 Justin Sun 控制的 Polo Digital Assets；主要用户在亚洲。事件前 24h 交易量约 $70M。
- **时间轴**：
  - 2023-11-10 16:00 UTC（北京时间 11-11 00:00）：Etherscan 上 Poloniex 多个热钱包地址开始向若干新 EOA 大额转出。
  - 16:45 UTC：PeckShield、Cyvers 等监控系统发出警报。
  - 17:04 UTC：孙宇晨发 Twitter 确认被黑，承诺 100% 赔付用户并向攻击者发 5% 白帽通道；48h 内无回应将"报警追讨 + 1 倍奖金给抓捕者"。
  - 2023-11-11：攻击者开始把部分 ETH 通过 Uniswap 兑换，分批转入 Tornado Cash 与 eXch。
  - 2023-11-13：HTX（前火币）、Binance 冻结部分可识别资金。

## 2. 事件影响

- **直接损失区间**（$100M–$125M，主流 $125M；PeckShield 计 $126M）：
  - Ethereum 上：ETH/USDT/USDC/SHIB/stETH/MATIC 等约 $89M
  - TRON 上：TRX、USDT-TRC20 约 $22M
  - BTC：约 $14M
- **受害方**：Poloniex 平台本身（官方承诺用户 100% 赔付，即损失由 Sun 本人/HTX 关联实体承担）。
- **连带影响**：TRX 短期内下跌约 3%；HTX、Poloniex、JustLend 关联度再次引发市场质疑。
- **资金去向**：
  - Ethereum：数百笔兑换 ETH → 部分进 Tornado Cash；部分经 eXch、FixedFloat 混兑为 BTC。
  - BTC：沉寂后被转入 Sinbad / Yonbi 类混币器。
  - TRON：部分 USDT-TRC20 被 Tether 冻结（据 Tether 公告约 $5M）。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**Key-Mgmt / 运维安全**——热钱包私钥泄露。无智能合约漏洞。

### 3.2 受损模块

- Poloniex 热钱包系统（闭源交易所后端）。
- 涉事链上地址（部分已由官方标记）：
  - `0xa910f92acdaf488fa6ef02174fb86208ad7722ba`（Poloniex Hot Wallet 4）
  - `0xe29c61e6d38d2f029c195a3ea12e9b2bc53a7eef`（攻击者中转）
  - TRON: `TKNZeqxn7Ewsu5HG5f3TWwgxdMPyZaNTo4`

### 3.3 失守方式（公开推测）

官方口径"under internal investigation"，至今未公开 root cause。主流推测：

```text
# 候选路径 A：运维服务器被入侵
- 攻击者穿透 Poloniex 签名服务或 HSM 调用网关
- 在可签名窗口内对多个链发起最大额度 withdraw

# 候选路径 B：私钥管理体系（助记词或 seed）被内部人员外泄
- Poloniex 于 2019 年交接给 Sun 旗下后多次换团队，历史 credential 管理存疑
- 社区有观察指出部分攻击地址与 2022 年 HTX 小规模事件资金有重叠，
  间接支持内部或共享基础设施说

# 候选路径 C：第三方热钱包软件供应链污染
```

无论哪种路径，共同特征是攻击者拥有**有效私钥**并能从多条链平行提款，说明签名系统在跨链层被集中控制——这是典型 CEX 热钱包设计缺陷（所有链热钱包由同一密管系统生成）。

### 3.4 攻击步骤分解（链上观察）

1. 攻击者在 2023-11-10 16:00 UTC 前准备好跨链目标地址（6 个新 EOA）。
2. 在约 15 分钟内对 Ethereum、TRON、BTC 同步发起提现级别转账，每笔一次抽走大额 ERC20。
3. Ethereum 侧主要 tx 把资产聚拢到 `0xe29c61e6...`，后分拆为数十地址做 cooldown。
4. 24 小时内把主力 ETH 换入 Tornado Cash（彼时单笔 100 ETH anonymity set 被持续喂入）。
5. BTC 侧使用 peel chain 模式分散至 100+ 地址，多数最终沉入 Sinbad 等混币。

### 3.5 为何未提前发现

- CEX 热钱包监控通常有"提款异常阈值"，但若攻击一次性触发"接近余额全额"的合法提款，仅能事后告警无法事前拦截。
- 跨链同时行动说明攻击者对 Poloniex 密管体系有充分先验知识——符合内部人或长期潜伏外部攻击者特征。

## 4. 事后响应

- 孙宇晨个人承诺全额赔付用户，并承诺 5% 白帽通道 + 另设 1 倍赏金给协助抓捕者。
- 与 Tether、Circle 协作冻结部分 USDT/USDC（合计 ~$5M）。
- Binance、OKX、HTX 等交易所接受 Poloniex 标记黑名单并冻结相关入金（合计 ~$5M）。
- Elliptic 2023-12 发布归因报告，认为本次事件与 HTX 2023-09 事件资金流向高度重合，归因 Lazarus Group（归因强度：中高）。
- Poloniex 停机数日后分阶段恢复存提。

## 5. 启发与教训

- **对 CEX**：热钱包应按链隔离签名系统，禁止跨链共享密管；设置链上 timelock + 多签签核大额提现；部署 real-time anomaly 模型。
- **对用户**：CEX 上永远存在对手方风险；大额资产分散到非托管方案。
- **对行业**：与 FTX、Mt.Gox 等重大事件一样，关联方透明度依旧是顽疾；Poloniex-HTX-JustLend 共用基础设施的怀疑未被打消。
- **对监管**：若归因 Lazarus 成立，则朝鲜通过 CEX 热钱包攻击的资金规模在 2022–2024 年累计已达 $3B+，加剧对 OFAC 链上工具（Tornado Cash 制裁、混币器地址清单）的依赖。

## 6. 参考资料

- SlowMist: <https://slowmist.medium.com/slowmist-analysis-of-the-poloniex-hack-5e8b0bbc7f0b>
- rekt.news: <https://rekt.news/poloniex-rekt/>
- CertiK: <https://www.certik.com/resources/blog/5JWrmBZcXw3a7kT-poloniex-heist-analysis>
- Elliptic 归因: <https://www.elliptic.co/blog/poloniex-hack-north-korea-lazarus>
- Justin Sun 公开声明: <https://twitter.com/justinsuntron/status/1723208812421902359>
- PeckShield 监控: <https://twitter.com/PeckShieldAlert/status/1722990124601770289>
- Etherscan 攻击地址: <https://etherscan.io/address/0xe29c61e6d38d2f029c195a3ea12e9b2bc53a7eef>

---

*Last verified: 2026-04-23*
