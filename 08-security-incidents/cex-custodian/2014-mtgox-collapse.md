---
title: Mt.Gox 破产事件（2014-02-07, ~$450M 按 2014-02 价 / ~$51B 按 2025 峰值）
module: 08-security-incidents
category: CEX-hack
date: 2014-02-07
loss_usd: 450000000
chain: [Bitcoin]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.mtgox.com/img/pdf/20140228-announcement_eng.pdf
  - https://bitcoinmagazine.com/business/the-inside-story-of-mt-gox-bitcoins-460-million-disaster-1424864436
  - https://www.wired.com/2014/03/bitcoin-exchange/
  - https://blog.wizsec.jp/2017/07/breaking-open-mtgox-1.html
  - https://www.ccn.com/chainalysis-mt-gox-did-not-lose-650000-bitcoin/
tx_hashes:
  - 攻击者早期提币聚合地址 1Feex3Dfk5y6yCE9BE7ChsJmmFEmwGvPP6（WizSec 追踪，待核实）
  - 部分 BTC 流向 BTC-e（2017 年 Alexander Vinnik 被捕时冻结）
---

# Mt.Gox 破产事件

> **TL;DR**：2014 年 2 月 7 日，曾占据全球 BTC 交易量 70% 的日本交易所 Mt.Gox 停止提币；2 月 28 日其在东京地方法院申请民事再生（破产保护），公告显示约 850,000 BTC（其中客户 750,000 + 自有 100,000）"因 transaction malleability 攻击"失踪，时价约 4.5 亿美元。后续 WizSec、Chainalysis 等独立调查表明，malleability 只是官方借以解释的表层原因——实际 80% 以上的 BTC 早在 2011 年中至 2013 年间被长期盗取，主犯通过持续入侵 Mt.Gox 的热钱包私钥将 BTC 搬至 BTC-e 洗白。这是加密行业第一次 Tier-1 级系统性崩溃，直接催化了"Not your keys, not your coins"的文化，并深刻影响了此后十年 CEX 监管与自托管叙事。

## 1. 事件背景

- **主体**：Mt.Gox 于 2010 年由 Jed McCaleb 创立，2011 年 3 月转让给法籍 CEO Mark Karpelès 运营的 Tibanne K.K.。高峰期（2013 年）处理全球约 70% BTC 交易量。
- **时间轴**：
  - 2011-03（背景）：MtGox wallet.dat 被前任持有者/员工复制（WizSec 证据），导致约 79,956 BTC 热钱包最早被逐步转出，此后盗取持续至 2013 年。
  - 2011-06-19：首次数据库泄露（见 `2011-mtgox-first-breach.md`），管理端账户被入侵。
  - 2013-05：美国国土安全部查封 Mt.Gox 在 Dwolla 的美元账户（原因：未注册资金转移业务）。
  - 2014-02-07：Mt.Gox 暂停所有 BTC 提现，官方称原因为 transaction malleability 导致内部账本与链上不一致。
  - 2014-02-24：网站下线。
  - 2014-02-28：东京法院申请民事再生，公告 850,000 BTC + ¥28 亿日元现金失踪。
  - 2014-03-20：Mt.Gox 宣布在旧冷钱包"意外发现" 200,000 BTC，实际缺口修正为 650,000 BTC。
  - 2015-08：日本警方以业务侵占、数据篡改等罪名逮捕 Karpelès；2019 被判业务数据篡改有罪但侵占无罪。
  - 2017-07：美国执法部门逮捕俄罗斯人 Alexander Vinnik（BTC-e 运营者），起诉书指其经手 ~530,000 BTC 来源于 MtGox。
  - 2021–2024：Mt.Gox 破产信托进入"民事复原"（rehabilitation）阶段，开始分批向债权人分发 BTC/BCH/日元。
- **发现过程**：2014-02-07 用户提币持续失败；2 月中社区通过链上分析发现 Mt.Gox 热钱包余额长期与账面不符；WizSec 于 2015–2017 年通过地址聚类最终完成攻击还原。

## 2. 事件影响

- **直接损失**：
  - 客户 BTC 约 750,000（按 2014-02-28 ~$545 估 ~$409M）
  - Mt.Gox 自有 BTC 约 100,000（~$55M）
  - 现金 ¥28 亿（~$28M）
  - 合计当时价约 4.5 亿美元；若按 2025 年 BTC ~$60k 估值，缺口名义上达 $51B 级（仅作规模对比参考）。
- **受害方**：127,000+ 债权人（日本、美国、欧洲、中国用户均众多）；BTC 现货市场整体信心。
- **连带影响**：
  - BTC 从 2013-11 高点 $1,150 跌至 2014 年底 $200 的长熊，与 MtGox 事件高度相关。
  - 推动 BitLicense（NY DFS 2015）、FinCEN MSB 登记严格化、日本《资金决济法》修订（2017 年将交易所纳入登记制）。
  - BitGo、Xapo、Coinbase Vault 等托管方案在 2014–2016 年崛起。
- **资金去向**：
  - WizSec 追踪发现约 630,000 BTC 于 2011–2013 年间陆续转入 BTC-e 被 Vinnik 洗白。
  - 部分被美国政府在 2017 年 Vinnik 被捕时冻结。
  - 信托在 2017–2018 年出售部分 BTC/BCH 换回约 ¥430 亿日元。
  - 2024-07 开始分发残余 142,000 BTC（其中一部分以 BTC/BCH 形式还债权人）。

## 3. 技术根因（核心章节）

- **漏洞分类**：Key-Mgmt（主因） + Protocol-Bug（malleability，但仅为表层掩护） + Social-Eng + Governance。
- **三个关键技术事实**：

### 3.1 Transaction Malleability 被官方错误归因

Bitcoin 在 SegWit 之前（BIP-141，2017-08 激活）允许第三方在广播前对已签名 tx 的 `scriptSig` 做非关键性修改（如 ECDSA 签名 `s` 值取 curve order 的补），修改后 tx 仍然有效，但 txid 改变。

Mt.Gox 的运维逻辑大致为：

```python
# 简化还原 Mt.Gox 提现处理（基于 Karpelès 法庭陈述与 Wizsec 分析）
def process_withdraw(user, amount, to_addr):
    txid_internal = bitcoind.sendtoaddress(to_addr, amount)  # 本地计算 txid
    db.mark_sent(user, txid_internal)
    # 缺陷：此后仅按 txid 查询链上是否 confirmed
    while not bitcoind.gettransaction(txid_internal).confirmations:
        time.sleep(60)
    # 若攻击者 malleate 了 tx，链上以新 txid 落账，旧 txid 查询 "not found"
    # Mt.Gox 系统以为交易失败，重发一次 —— 实际用户收到两次！
```

攻击者可以通过 malleate 攻击让 Mt.Gox "重发提币"，官方 2014-02-10 的技术说明以此解释缺口。但随后 Decker & Wattenhofer（2014 ETH Zurich 论文）分析 2014-02 区块链数据，发现 malleable tx 累计至多只能解释几千 BTC，**不可能解释 65 万 BTC 的缺口**。

### 3.2 长期热钱包私钥盗窃（主因）

WizSec（Kim Nilsson）2015–2017 通过地址聚类给出更可信的还原：

1. 2011 年某时（可能是 2011-09 前后），攻击者获得 Mt.Gox 主热钱包的 `wallet.dat`，包含大量私钥。
2. 攻击者从 2011 末至 2013 年中，持续以每日数千 BTC 的节奏将 BTC 从 Mt.Gox 钱包转出到自控地址（主流向 BTC-e）。
3. Mt.Gox 由于账本（MySQL）与链上钱包余额对账极不频繁，多年未察觉累积差额。
4. 2013 年末 Mt.Gox 余额见底，才被迫在 2014 年初制造 malleability 叙事公告。
5. 关键聚合地址包括 `1Feex...wGvPP6` 等（WizSec 命名 "MtGoxHacker"），链上行为特征高度一致。

### 3.3 运营层面的多重失效

- **冷热钱包边界不清**：根据 Karpelès 法庭陈述，Mt.Gox 并无严格的冷热比例守则，CEO 单人拥有冷钱包绝大多数私钥。
- **对账频次低**：链上余额与内部 MySQL 账本对账周期以月甚至年计。
- **私钥访问无审计日志**：谁在何时读取 `wallet.dat`、从哪个 IP，均无完整日志。
- **代码质量**：Mt.Gox 交易引擎核心为 PHP，数据一致性与并发处理广受内部工程师诟病（见 2014 年 Wired、Ars Technica 采访）。

- **攻击 / 事件步骤还原**：
  1. （2011 年）攻击者通过未知手段（可能是前任 CEO 时期员工侧泄漏，或后续服务器入侵）取得 Mt.Gox 热钱包 `wallet.dat`。
  2. 长期以低频次提币模式盗取 BTC，避免触发异常告警。
  3. 2014-02-07：Mt.Gox 钱包基本告罄，以 malleability 为由停止提现。
  4. 2014-02-28：破产公告。
  5. 2014-03-20：以"从旧冷钱包意外发现 200k BTC"为名，将剩余库存补入信托。

- **tx hash**：攻击聚合地址 `1Feex...wGvPP6`（WizSec 公开），BTC-e 入金 hash 过多不逐条列出。
- **审计盲区**：Mt.Gox 2011–2013 年未接受任何主流会计事务所对 BTC 资产的审计；日本 FSA 当时亦无加密货币监管框架。

## 4. 事后响应

- **项目方**：2014-04 提交民事再生计划；公司清算由破产管财人小林信明（2018 年起任 Rehabilitation Trustee）接手。
- **资产追回 / 赔付**：
  - 信托 2014 年接管残余 ~200,000 BTC，2017–2018 年抛售约 4 万 BTC 获得约 ¥430 亿日元，该抛售被社区视为当年 BTC 价格下跌诱因之一。
  - 2021-10：债权人大会通过民事复原计划，改以 BTC/BCH + 日元方式返还。
  - 2024-07 起正式分发；截至 2025 年，超过 90% 债权人收到首轮分发。
- **法律 / 执法**：
  - 2015-08 Mark Karpelès 被捕；2019 东京地方法院裁定"业务数据篡改"有罪（缓期 2 年半），"业务侵占"等指控无罪。
  - 2017-07 Alexander Vinnik（BTC-e 实际控制人）在希腊被 FBI 逮捕，2022 美国引渡成功；2024 在美国联邦法院认罪，承认经手 MtGox 被盗 BTC。
- **行业连锁**：
  - 直接催生日本《资金决済法》2017 修订，要求所有加密交易所在 FSA 登记。
  - 推动 Proof-of-Reserves 概念（Kraken 2014 年首次实施 Merkle-tree PoR）。
  - 冷热钱包比例、保险（BitGo、Coincover）、托管分离成为 CEX 标配。

## 5. 启发与教训

- **对开发者 / CEX**：
  - 链上余额与账本必须每日甚至每块对账，发现不一致立即暂停提现。
  - 热钱包私钥必须受 HSM / 多签保护，任何出金必须多人多设备签名。
  - malleability 风险即使在 SegWit 后也提醒我们：依赖 txid 作为幂等键是危险的；应使用 (to, amount, nonce) 幂等保护。
- **对审计方**：
  - 需建立 Proof-of-Reserves 标准（默克尔树 + 签名挑战 + 负债侧零知识证明）。
  - 需关注多年财务账本的对账历史（MtGox 教训是"问题积累了 3 年才爆发"）。
- **对用户**：
  - "Not your keys, not your coins" 成为行业信条。
  - 分散交易所风险，长期持有资产上硬件钱包。
- **对协议**：SegWit 解决了 malleability 本身，但 Mt.Gox 的主因是私钥管理，协议升级不能替代运营纪律。

## 6. 参考资料

- Mt.Gox 官方 2014-02-28 破产公告 PDF（公司域名）
- WizSec / Kim Nilsson：《Breaking open the MtGox case, part 1》2017-07（blog.wizsec.jp）
- Wired：《The Inside Story of Mt. Gox, Bitcoin's $460 Million Disaster》2014-03（Robert McMillan）
- Bitcoin Magazine：《The Inside Story of Mt. Gox》 2015-02（Daniel Cawrey）
- Decker & Wattenhofer：《Bitcoin Transaction Malleability and MtGox》ESORICS 2014
- Chainalysis 报告：《Mt. Gox Did Not Lose 650,000 Bitcoin》2017
- 美国司法部 Vinnik 起诉书 No. 3:16-cr-00227（N.D. Cal.）
- 慢雾事后回顾：SlowMist 安全月报多篇（2018–2020）

---

*Last verified: 2026-04-23*
