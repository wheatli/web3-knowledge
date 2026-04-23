---
title: Atomic Wallet 批量盗币（2023-06-03, ~$100M+）
module: 08-security-incidents
category: Wallet | Supply-Chain
date: 2023-06-03
loss_usd: 100000000
chain: Ethereum, Bitcoin, TRON, BNB Chain, XRP, Polygon, Tezos
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/slowmist-analysis-of-the-atomic-wallet-theft-5cab7a8e0ef8
  - https://rekt.news/atomic-wallet-rekt/
  - https://www.elliptic.co/blog/lazarus-group-identified-as-perpetrator-of-atomic-wallet-heist
  - https://www.chainalysis.com/blog/atomic-wallet-hack-lazarus-group/
  - https://twitter.com/AtomicWallet/status/1665337483542011907
tx_hashes:
  - 0xa68b4d9f3b2ab6b1bdf2c2dc07acf85b56a0707f5d77d8bc04b7d3dc0cfc2db4 (Ethereum, 一位典型受害者的转出 tx)
  - 未公开（多链数千受害者）
---

# Atomic Wallet 批量盗币事件

> **TL;DR**：2023-06-03 起，非托管多链钱包 Atomic Wallet 数千名用户资产被盗，累计损失 $100M+。攻击向量至今存在多种说法：恶意更新包（供应链污染）、弱随机数导致助记词可预测、或 RPC 端点被劫持。Chainalysis 与 Elliptic 后续独立将资金流向归因给朝鲜 Lazarus 集团。无单一链上合约漏洞，事件属客户端 / 种子层失守。

## 1. 事件背景

- **项目**：Atomic Wallet（由爱沙尼亚公司 Atomic Protocol OÜ 运营），非托管多链桌面 + 移动端钱包，支持 500+ 币种，peak MAU ~5M。
- **时间轴**：
  - 2023-06-02 下午起：链上监控（ZachXBT、MistTrack）出现多地址同步"整仓转出"，覆盖 BTC、ETH、USDT-TRC20、XRP、LTC、DOGE。
  - 2023-06-03 11:00 UTC：Atomic Wallet 官方 Twitter 首次承认"less than 1% of our users"受影响。
  - 2023-06-05：ZachXBT 公开 58 名受害者损失合计 $35M（仅可识别部分）。
  - 2023-06-14：Elliptic 发布归因报告指向 Lazarus。
  - 2023-07-18：Chainalysis 发布独立分析确认归因。
- **发现过程**：社区与独立研究员先于官方察觉；MistTrack（SlowMist 旗下）与 MetaSleuth 先后建立受害者自报表，最终可核实受害约 5,500 个地址。

## 2. 事件影响

- **直接损失**（按当日价汇总区间 $60M–$130M，主流引用 ~$100M）：
  - USDT-TRC20 约 $37M
  - BTC 约 $30M
  - ETH 约 $22M
  - 其他（XRP、DOGE、LTC、ADA、SOL 等）合计 ~$15M
- **受害方**：Atomic Wallet 普通用户（多为长期持币者，含早期 AWC 投资人）；官方从未公布完整赔付。
- **连带影响**：非托管钱包"客户端安全"议题回热；Trust Wallet、Exodus 等竞品紧急发布安全 FAQ。
- **资金去向**：资金通过 Sinbad（彼时 Lazarus 偏好的 BTC 混币服务）、后续转向 Yonbi / Railgun；部分兑换为 XMR。美国 OFAC 后将 Sinbad 列入制裁名单，部分原因即为此案。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**钱包客户端安全**——私钥/助记词在用户设备层失守。具体向量**至今公开说法不统一**：

候选假设（Atomic Wallet、SlowMist、Least Authority、独立研究员分别提出过）：

1. **供应链 / 恶意更新**：Atomic 更新服务器被入侵，向部分用户推送了植入 exfiltration 逻辑的客户端版本。
2. **弱 RNG / 助记词熵不足**：早期版本生成助记词时 RNG 依赖 `Math.random` 或系统熵源不足，攻击者可在特定时间窗批量恢复。
3. **密钥加密薄弱**：本地存储的 keyfile 使用可暴破的加密方案（低迭代 PBKDF2 + 仅口令派生），用户数据库通过某种方式外泄后被离线暴破。
4. **钓鱼/假更新网页 + 广告劫持**。

Atomic 官方最终发表的简短声明仅含糊提到"infrastructure virus"，未给代码级 PoC。Least Authority 2022 年早于事件发布的审计报告曾指出 Atomic 客户端在密码派生、种子加密、RNG 来源等方面**存在至少 5 项 Critical/High 问题**并未被完全修复——审计结论是：无法推荐用户信任该客户端存储敏感密钥。

### 3.2 受损模块

- Atomic Wallet Desktop / Mobile（闭源，仅对 2018 年老版部分开源）。
- 签名/种子管理模块及其更新通道。

### 3.3 关键"伪代码"（Least Authority 报告给出的反面模式简化）

```text
# 反面模式（审计报告指出）
seed = entropy()  # 存在时 entropy 依赖较弱来源
encrypted = AES( seed, key=PBKDF2( user_password, iter=~10_000 ) )  # 迭代数偏低
store_on_disk(encrypted)

# 正确做法（BIP-39 + scrypt/Argon2 + 足够迭代）
seed = crypto_secure_random()
key  = argon2id( password, salt, t=3, m=64MB, p=4 )
encrypted = AES-GCM( seed, key )
```

由于客户端闭源，外界无法精确复现攻击链；所有分析均属"外部黑盒 + 受害者钱包共性分析"（例如受害账户的助记词生成时间集中在某一版本区间）。

### 3.4 攻击步骤（高层）

1. 攻击者获取可预测/可解密的种子集合。
2. 建立脚本批量签署转账 tx。
3. 在 2023-06-02/03 之间集中发起链上清仓，24 小时内完成大头。
4. 将资金按资产类型路由至不同混币服务（BTC → Sinbad；USDT → TRC20 复杂分散；ETH → Tornado Cash 受限路径）。

## 4. 事后响应

- Atomic 官方开设有限赔付基金，承诺最多覆盖单用户损失的某固定百分比；大量受害者反馈赔付进度缓慢。
- Atomic 聘请链上侦探与安全公司协作追踪，但未公布完整取证报告。
- Chainalysis、Elliptic 独立分析把资金流向归因 **Lazarus Group (DPRK)**，此归因基于与 Ronin、Harmony 事件的资金轨迹重合。
- 美国财政部 2023-11-29 制裁 Sinbad 时将本案列为主要依据之一。
- 欧盟与爱沙尼亚金融监管对 Atomic Protocol OÜ 启动询查。

## 5. 启发与教训

- **对用户**：非托管不等于安全——客户端实现同样重要。闭源钱包无法外部审计；Least Authority 报告当年的 5 项 Critical 问题应视为重大红旗。
- **对钱包厂商**：必须开源关键种子/签名模块；强制使用 Argon2id/scrypt 密钥派生、HSM-backed secure enclave（iOS Secure Enclave / Android StrongBox）。
- **对行业**：建立钱包客户端安全标准（CryptoCurrency Security Standard，CCSS 类）；鼓励独立审计 + reproducible build。
- **对受害者支持**：社区级"链上受害者登记 + 援助"机制（MistTrack、Chainabuse）是本次事件中唯一有效的数据基础设施。

## 6. 参考资料

- SlowMist: <https://slowmist.medium.com/slowmist-analysis-of-the-atomic-wallet-theft-5cab7a8e0ef8>
- rekt.news: <https://rekt.news/atomic-wallet-rekt/>
- Elliptic 归因: <https://www.elliptic.co/blog/lazarus-group-identified-as-perpetrator-of-atomic-wallet-heist>
- Chainalysis: <https://www.chainalysis.com/blog/atomic-wallet-hack-lazarus-group/>
- Least Authority 2022 审计: <https://leastauthority.com/blog/audit-of-atomic-wallet/>
- Atomic 官方声明: <https://twitter.com/AtomicWallet/status/1665337483542011907>
- ZachXBT 公开追踪: <https://twitter.com/zachxbt/status/1665365322116755457>

---

*Last verified: 2026-04-23*
