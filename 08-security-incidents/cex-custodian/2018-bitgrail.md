---
title: BitGrail NANO(XRB) 被盗 ~$170M（2018-02-09）
module: 08-security-incidents
category: CEX-hack
date: 2018-02-09
loss_usd: 170000000
chain: [NANO]
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://bitgrail.net/
  - https://medium.com/@nanocurrency/an-update-on-bitgrail-f4e04fc56b17
  - https://cointelegraph.com/news/italian-exchange-bitgrail-claims-17-mln-nano-tokens-stolen
  - https://www.theblock.co/post/9080/bitgrail-bankruptcy
tx_hashes:
  - 缺口约 17,000,000 NANO；BitGrail 从未发布具体攻击交易 hash（官方内部账本与链上账本长期不对账）
  - 链上 NANO（XRB）账本可查，但攻击 tx 未被 BitGrail 明确标记
---

# BitGrail NANO 被盗 $170M

> **TL;DR**：2018 年 2 月 9 日，意大利小型交易所 BitGrail 宣布其 NANO（当时符号 XRB）热钱包被盗约 17,000,000 NANO，按当时 ~$10 估值约 $170M。CEO Francesco "The Bomber" Firano 将责任推给 NANO 协议层"预重组"问题；NANO 开发团队强烈否认并出示链上证据表明问题在 BitGrail 内部账本/提现逻辑。事后意大利佛罗伦萨法院裁定 Firano 本人需对大部分损失负责；BitGrail 进入破产清算。事件为 Web3 早期"链上 UTXO/账本对账失控 + CEO 推诿"的典型案例。

## 1. 事件背景

- **主体**：BitGrail 由 Francesco Firano 个人运营，最初是唯一大规模上线 NANO / RaiBlocks 的交易所，2017 年末随着 NANO 涨幅 100 倍享有巨大交易量。
- **时间轴**：
  - 2017-10 起：BitGrail 内部报告陆续出现"余额不一致"问题，Firano 本人在 Discord 承认。
  - 2018-01：NANO 开发团队与 Firano 关于软件漏洞的归属开始公开争执。
  - 2018-02-09：BitGrail 官方公告遭黑，宣布 17M NANO 丢失，暂停所有提现。
  - 2018-02-10：NANO Foundation 发布 Medium 声明，提供链上证据否认 NANO 协议或官方钱包存在漏洞。
  - 2018-05：意大利佛罗伦萨法院受理集体诉讼，BitGrail 被宣告破产。
  - 2019-01：法院初步判决 Firano 需返还用户资金并承担约 1,700 万欧元的个人责任。
  - 2023：意大利破产程序仍在进行，债权人回收率低。
- **发现过程**：用户发现提现长期延迟 → Firano 公开承认漏洞 → NANO 基金会介入 → 破产。

## 2. 事件影响

- **直接损失**：约 17,000,000 NANO。按 2018-02-09 ~$10 估 ~$170M；若按 NANO 历史高点 ~$33 估算则 ~$561M。
- **受害方**：约 230,000 BitGrail 用户。
- **连带影响**：
  - NANO 代币信誉受损，尽管基金会证明协议本身无漏洞，价格仍从 $10 跌至 2018 末 $1 以下。
  - 成为小型 CEX "单人运营 + 交易所自研后端 + 无任何审计"风险的代表案例。
- **资金去向**：BitGrail 内部账本差额从未完整对账上链，具体是"被黑"还是"长期提现逻辑 bug 双花"至今存在争议；部分链上地址被 NANO 社区标记，但官方无权冻结（NANO 无账户冻结机制）。
- **归因**：归因未公开。法院判决重点是 Firano 未履行安全责任，而非定性某个外部黑客。

## 3. 技术根因

- **漏洞分类**：CEX-internal Accounting + Race-Condition + Governance / One-Man-Operation。
- **根因分析**（基于 NANO 基金会与 Firano 往来邮件、诉讼文件公开部分）：
  1. **BitGrail 自研提现逻辑的竞态**：NANO 是 account-balance 模型并支持极快确认（~秒级），BitGrail 的账本检查在处理并发提币时存在 TOCTOU（Time-Of-Check / Time-Of-Use）窗口。用户可在内部账本未及时减扣期间发起多笔同额提现。
  2. **长期未对账**：BitGrail 自 2017-10 已发现内外账本不一致，但未暂停服务。
  3. **所有资产存单一热钱包**：NANO 支持的多账户模型未被 BitGrail 有效利用。
  4. **CEO 单人控制所有密钥**：无多签，无外部审计。
- **关键伪代码**（典型并发提现漏洞示意，非 BitGrail 原始代码）：

```python
# BitGrail 类似实现的并发提现漏洞示意
def withdraw(user_id, amount):
    # 缺陷：check 与 update 非原子
    balance = db.get_balance(user_id)       # (T1) 读取 balance = 100
    if balance < amount:
        return "insufficient"
    # 发起真实 NANO 链上转账（毫秒级完成）
    send_nano(user_id.wallet, amount)       # (T2) 链上已扣
    # 更新账本（可能晚于下一次 T1）
    db.set_balance(user_id, balance - amount)  # (T3) 扣账
# 用户并发发起 10 笔 $90 提现，在 T3 执行前 T1 都读到 100，
# 于是 10 次链上转账全部放行
```

- **攻击/事件步骤**：
  - 攻击者（或多名用户）利用上述竞态，持续在高并发时间段发起同额多笔提现。
  - BitGrail 内部账本余额在数月时间内被逐步掏空。
  - Firano 在 2018-02-09 宣告"遭黑"。
- **tx hash**：NANO 链上可查，但 BitGrail 未给出具体 hash 列表（tx 未公开）。
- **为何未发现**：BitGrail 无任何外部审计；Firano 一人掌控技术与运营。

## 4. 事后响应

- **项目方 / 运营方**：
  - 宣布破产并发行 BitGrail Shares（BGS）补偿 token，社区视为无实际价值。
  - Firano 拒绝承担个人责任，转而起诉 NANO 开发团队，后败诉。
- **NANO 基金会**：
  - 发布详细技术反驳，证明 NANO 协议正常。
  - 发起 LegalRaiser 众筹为受害者法律程序筹款。
- **执法**：2019–2023 意大利司法程序；2019 年初审 Firano 败诉；二审维持。
- **行业连锁**：
  - 成为"单人 CEX + 无审计"风险的教科书案例，推动小规模新币上线大 CEX 意愿。
  - NANO 社区此后长期避开单一小交易所中心化风险。

## 5. 启发与教训

- **对开发者 / CEX**：
  - 所有提现逻辑必须使用数据库行级锁或分布式锁，确保 check-and-act 原子。
  - 链上余额与内部账本每日对账；对账差异即停服调查。
- **对审计方**：小 CEX 的合规审计若仅看合约（CEX 没有合约）就完全失去价值；必须覆盖后端账本对账机制。
- **对用户**：小交易所的"单人运营"模式是系统性风险；合规监管下线严于审批上线。
- **对协议 / 生态**：NANO 团队正面处理了归责问题，通过公开证据与法律支持维护协议声誉，成为项目方面对 CEX 崩盘的 PR 教科书。

## 6. 参考资料

- NANO 基金会 Medium：《An Update on BitGrail》2018-02-10
- Cointelegraph：《Italian Exchange BitGrail Claims $170M in NANO Tokens Stolen》
- The Block：《BitGrail Bankruptcy》2018-05
- 意大利佛罗伦萨法院判决书（公开版）：2019 年判决文本
- SlowMist 2018 年报：BitGrail 条目
- NANO Discord / GitHub 早期对账讨论归档

---

*Last verified: 2026-04-23*
