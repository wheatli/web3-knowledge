---
title: 币安（Binance）热钱包 7,000 BTC 被盗（2019-05-07, ~$41M 按当时价）
module: 08-security-incidents
category: CEX-hack
date: 2019-05-07
loss_usd: 41000000
chain: [Bitcoin]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.binance.com/en/support/announcement/binance-security-breach-update-9c280ab664a144fdb10c2df4ff0bbd7a
  - https://www.coindesk.com/markets/2019/05/08/binance-lost-40-million-in-bitcoin-hack
  - https://slowmist.medium.com/binance-exchange-hacked-how-it-happened-and-what-we-can-learn-from-it-e4f26c7c0d52
  - https://www.chainalysis.com/blog/lazarus-group-crypto-exchange-hacks/
tx_hashes:
  - 攻击者主要接收地址：bc1qu3muzz8sze0l69pf2wrqwxq2qwer68lyc4d8ma（Binance 官方标注）
  - 单笔集中转出 tx：44b1cca0...（Binance CZ AMA 提及；完整 hash 未官方列出）
---

# 币安热钱包 7,000 BTC 被盗

> **TL;DR**：2019 年 5 月 7 日 17:15 UTC，币安（Binance）宣布其 BTC 热钱包被盗 7,000 BTC（约 4,100 万美元），为 CZ 时代币安最严重的安全事件。攻击者精心策划长达数月，以"钓鱼 + 恶意软件 + API key 滥用 + 2FA 绕过"多手法组合绕过风控，在单一交易内一次性转出。币安动用 SAFU 基金全额兜底用户；CEO 赵长鹏一度公开讨论"是否通过 Bitcoin rollback 恢复"，引发社区对去中心化原则的辩论，最终放弃。慢雾在事后分析认为攻击者掌握多名用户 API key 并同步调用提现，部分行为与 Lazarus 此后洗钱路径一致，但 Binance 官方未给出定性归因。

## 1. 事件背景

- **主体**：Binance（币安）成立 2017-07，事件时为全球现货交易量最大的 CEX 之一。
- **时间轴**：
  - 2019 年初 – 5 月初：攻击者长期潜伏，通过钓鱼、恶意软件等多渠道积累大量有效 API key 与 2FA 凭证。
  - 2019-05-07 17:15:24 UTC：币安风控系统检测到异常，但已有一笔包含大量 UTXO 的 tx 通过风控并广播，合计 7,074 BTC 被转出至 7 个攻击者地址。
  - 2019-05-07 17:17 UTC：币安暂停所有提现。
  - 2019-05-08 03:00 UTC：CZ 在 Twitter 与官方公告披露事件并介绍 SAFU 兜底。
  - 2019-05-08：CZ 公开考虑"与矿工合作回滚 Bitcoin 链以追回 BTC"，遭 Jeremy Rubin 等开发者与社区强烈反对后作罢。
  - 2019-05-15：BTC/ETH 提现恢复。
- **发现过程**：币安风控在单笔出金后数秒内告警；人工在数分钟内暂停所有资产出金。

## 2. 事件影响

- **直接损失**：7,074 BTC（~$41M 按 ~$5,800 估）。
- **受害方**：币安自有资金承担全部损失（SAFU 兜底）；用户资金不受影响。
- **连带影响**：
  - SAFU（Secure Asset Fund for Users，2018-07 设立，以 10% 交易手续费累积）首次大规模启用，证明了其必要性。
  - "链回滚"争论短暂但激烈，强化了比特币不可篡改的社会共识。
  - 业界对"CEX 热钱包即使 2% 资产也可能是巨额"的风险管理重新评估。
- **资金去向**：
  - 攻击者将 BTC 分散至数百地址并进入 ChipMixer 等混币服务。
  - Chainalysis 2019–2022 多份报告将部分资金流向归入 Lazarus 洗钱簇，但**Binance 官方未正式确认归因**。

## 3. 技术根因

- **漏洞分类**：Social-Eng + API-Key Leakage + 2FA-Bypass + Behavior-Risk 风控绕过。
- **关键事实**（综合币安官方公告 + 慢雾事后分析）：
  1. 攻击者积累了大量用户的 API key 与 2FA code（通过钓鱼网站、SMS 诈骗、浏览器恶意扩展、第三方服务端泄漏等综合渠道）。
  2. 攻击者"预习"多个用户的交易行为，确保提现请求在风控侧"看起来正常"。
  3. 在某一瞬间同步调用多个用户 API key 执行提现，每笔金额不超过用户单账号限额。
  4. 币安 API 提现虽需 2FA，但 2FA code 在攻击者掌握 API key 的同时被同步获取（SIM swap / malicious browser extension / phishing proxy）。
  5. 由于是合法用户账号发起，单笔金额合规，多笔合计才构成大额；币安风控在"单笔/单账户"维度未拦截；在"全局总和"维度告警但已被突破。
- **受损模块**：币安用户端与 API 端（闭源），以及第三方服务（浏览器扩展、SMS 通道）。无开源 commit 可引用。
- **伪代码示意（攻击者的聚合提现脚本）**：

```python
# 攻击者脚本示意（概念）
import binance_client
leaked_credentials = load_phished_creds()  # 数千条 (api_key, api_secret, 2fa_seed)
for cred in leaked_credentials:
    try:
        client = binance_client.Client(cred.key, cred.secret)
        code = totp.now(cred.twofa_seed)
        client.withdraw(
            asset="BTC",
            address=attacker_btc_addr_of(cred),
            amount=cred.max_allowed_btc,
            network="BTC",
            code=code
        )
    except Exception:
        continue
# 聚合效果：单笔风控通过，全局总量爆炸
```

- **攻击步骤**：
  1. 长期（月级）积累海量 API key + 2FA 种子。
  2. 针对性"训练"受害账号的历史提现特征（地址白名单、时间分布）。
  3. 2019-05-07 17:15 瞬间并发调用 提现 API，通过单一 UTXO 聚合 tx 发出 7,074 BTC 至 7 个攻击者地址。
  4. 风控发出全局告警，币安在几秒后暂停系统——已晚。
- **tx hash**：主要聚合地址包括 `bc1qu3muzz8sze0l69pf2wrqwxq2qwer68lyc4d8ma`，币安官方公开链上地址；完整 tx 列表在 Binance Nest 地址标注中可查。
- **为何风控未拦截**：风控基于历史正常行为建模，而攻击者使用的本身就是"账号的合法行为特征"；全局总量规则阈值在攻击发生时刚好没有拦截第一笔聚合。

## 4. 事后响应

- **项目方**：
  - SAFU 全额兜底损失，用户零损失。
  - 全面审查 API 提现体系：
    - 重置所有 API key。
    - 增加"白名单地址 + 地址生效 24h cooldown"机制。
    - 增强 2FA：推广 Google Authenticator / YubiKey，限制 SMS 2FA。
  - 推出 "Anti-Phishing Code"（发送邮件时包含用户自定义防钓鱼标识）。
- **资产追回**：未公开追回信息。
- **执法**：未公开立案结果。Chainalysis 将部分路径归入 Lazarus 但 Binance 未官方确认。
- **行业连锁**：
  - SAFU 模式被多家大所（KuCoin、OKX 等）模仿，成为 CEX 准保险基础设施。
  - API key 权限分层（读-only / trade-only / withdraw-only）成为行业惯例。

## 5. 启发与教训

- **对 CEX**：
  - 热钱包资产应设"24h 全局总量 + 单笔 + 单账户"多层限额，任何一层突破立即自动暂停。
  - API 提现必须绑定 allowlist 地址 + 24h cooldown。
  - 对异常聚合行为（多账号同时在极短时间内提现）进行 graph-based 风控建模。
  - 为用户提供"API key 自动轮转 + 定期过期"机制。
- **对用户**：
  - 不在第三方工具（Bot、面板）保存带提现权限的 API key。
  - 使用 YubiKey / 硬件 2FA；警惕 SMS 2FA 的 SIM swap 风险。
  - 开启 Anti-Phishing Code、withdrawal allowlist。
- **对行业**：币安事件后"CEX Bug Bounty + SAFU"成为标配；但也凸显"链回滚讨论"虽然未成真，却是对去中心化原则的一次公共压力测试。

## 6. 参考资料

- Binance 官方公告：《Binance Security Breach Update》2019-05-07
- CoinDesk：《Binance Lost $40 Million in Bitcoin Hack》2019-05-08
- 慢雾 SlowMist Medium：《Binance Exchange Hacked: How It Happened and What We Can Learn from It》2019-05
- Chainalysis：《Lazarus Group Crypto Exchange Hacks》2020、2022 报告
- CZ Twitter @cz_binance 2019-05-07/08 thread
- Jeremy Rubin Twitter：对回滚提议的反对
- rekt.news 回顾
- Etherscan / mempool.space：攻击者 BTC 地址集群

---

*Last verified: 2026-04-23*
