---
title: Bitfinex 120,000 BTC 被盗（2016-08-02, ~$72M 按当时价 / $4.5B 按 2022 追回时价）
module: 08-security-incidents
category: CEX-hack
date: 2016-08-02
loss_usd: 72000000
chain: [Bitcoin]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.bitfinex.com/posts/200
  - https://www.justice.gov/opa/pr/two-arrested-alleged-conspiracy-launder-45-billion-stolen-cryptocurrency
  - https://www.bitgo.com/newsroom/statements/statement-on-bitfinex
  - https://www.coindesk.com/markets/2016/08/03/bitfinex-loss-is-119756-btc-worth-65-million/
  - https://www.chainalysis.com/blog/bitfinex-hack-2016-recovery/
tx_hashes:
  - 被盗 BTC 聚合地址（由 FBI 在 2022-02 扣押）`bc1qazcm763858nkj2dj986etajv6wquslv8uxwczt`
  - 2022-02 美司法部扣押 ~94,000 BTC
---

# Bitfinex 120,000 BTC 被盗

> **TL;DR**：2016 年 8 月 2 日，当时全球最大 BTC/USD 现货交易所 Bitfinex 遭入侵，攻击者通过未公开的方法绕过 Bitfinex 与 BitGo 共同构建的"每个客户一个多签地址"的架构，在 2 小时内执行约 2,072 笔交易，共转出 119,756 BTC，时价约 7,200 万美元。事件独特之处在于：Bitfinex 当时采用 2-of-3 BitGo 联合签名多签并被视为"业界标杆"，但在实际 API 调用层被击穿，暴露了多签架构在"签名 API 调用方被攻陷"时失效的系统性风险。2022-02，美司法部逮捕 Ilya Lichtenstein 与 Heather Morgan 夫妇，追回约 9.4 万 BTC（时价约 36 亿美元），为当时美国史上最大规模的资产追缴。

## 1. 事件背景

- **主体**：Bitfinex 由 iFinex Inc. 运营，总部位于英属维京群岛（后迁至香港、欧洲），事件时承担全球 BTC/USD 现货交易量约 ~40%。
- **技术架构**：事件前 Bitfinex 与 BitGo 合作部署"每账户独立多签地址"：每个客户充值地址由 3 把私钥的 2-of-3 多签控制（1 把 Bitfinex 在线、1 把 Bitfinex 离线、1 把 BitGo co-signer）；BitGo 会根据 Bitfinex 传入的策略在 API 层做风控校验。该架构设计目标是"即使热钱包被盗，无 BitGo 共签也无法提币"。
- **时间轴**：
  - 2016-08-02 18:11 UTC 起：约 2,072 笔提现 tx 被广播，BitGo co-signer 均正常签名。
  - 2016-08-02 20:00 UTC：Bitfinex 暂停所有服务。
  - 2016-08-03：官方公告确认损失 119,756 BTC，按比例在用户账户中削减约 36%（所谓 "generalized haircut"），以 BFX token 代偿。
  - 2017-04：BFX token 由 iFinex 回购清偿完毕（Bitfinex 自称用交易所收益偿付）。
  - 2020-08：美司法部公开约 28 枚 BTC 从攻击者钱包流出被识别。
  - 2022-02-08：Ilya Lichtenstein、Heather Morgan（"Razzlekhan" 说唱 MV 本人）在纽约被捕；美司法部查扣 94,636 BTC（时价 ~$3.6B）。
  - 2023-08：两人在华盛顿联邦法院对共谋洗钱认罪。
  - 2024-11：Lichtenstein 被判 5 年监禁，Morgan 被判 18 个月。
- **发现过程**：Bitfinex 自动告警系统检测到异常出金率；运维在约 2 小时内完成系统下线。

## 2. 事件影响

- **直接损失**：119,756 BTC（~$72M 按 2016-08-02 价 ~$600）。
- **受害方**：Bitfinex 全体用户账户被 "socialized" 按比例削减约 36%，补发 BFX token；BitGo 的"多签即安全"叙事受挫。
- **连带影响**：
  - BTC 当日下跌约 20%。
  - 加密圈对"多签也可能被攻破"产生警觉，HSM / MPC / Fireblocks 等方案此后兴起。
  - BFX token → iFinex 股权 → Tether 发行公司 iFinex 的复杂资本结构自此建立，与 Tether 争议长期纠缠。
- **资金去向**：
  - 攻击者将 BTC 分散至数千地址静置。
  - 2017–2022 年间仅少量被转至 AlphaBay、Hydra 等暗网市场出金。
  - 2022-02 美司法部一次性扣押约 94,636 BTC；剩余约 2.5 万 BTC 下落部分未公开。

## 3. 技术根因

- **漏洞分类**：Architecture / Key-Mgmt + Governance（API 权限配置）。
- **根因分析**（综合 Bitfinex 与 BitGo 事后表态）：
  1. BitGo 2-of-3 共签依赖 Bitfinex 调用 BitGo API 时传入的策略参数。Bitfinex 在默认配置下未对"单笔/每小时最大提现总量"设限。
  2. 攻击者在 Bitfinex 内部网络获得可调用 BitGo 共签 API 的凭证（具体入侵路径 Bitfinex 从未公开，多数推测为内部服务器漏洞 + API key 泄露）。
  3. 攻击者通过合法的 API 调用流程，让 BitGo 对数千笔提现 tx 进行共签。BitGo 侧 "符合既定策略"，并未察觉异常——因为策略上没有设置总量上限。
  4. 多签架构本质上未被密码学攻破，而是在"策略配置"与"API 调用凭证"两个层面被击穿。
- **关键设计缺陷（概念模型）**：

```text
用户 -> Bitfinex 前端 -> Bitfinex 后端（签 1 次 + 调 BitGo API）-> BitGo co-signer（签第 2 次）-> 广播

攻击者只要掌握"Bitfinex 后端签名权 + API 调用凭证"，就等同于掌握 2-of-3 中的 2 把。
多签的安全前提是 3 把 key 由 3 个独立失败域持有，
而此架构把 2 个失败域（Bitfinex 热 key + BitGo API 凭证）合并到了 Bitfinex 服务器。
```

- **攻击步骤**（Bitfinex 从未公开细节，以下为公开信息可推断的流程）：
  1. 攻击者入侵 Bitfinex 内部服务器（具体路径未公开）。
  2. 获取调用 BitGo co-signer API 的客户端凭证与风控策略参数。
  3. 构造约 2,072 笔提现 tx，每笔指向攻击者自控地址。
  4. Bitfinex 后端正常签名、调用 BitGo API，BitGo 按现行策略共签。
  5. 2 小时内完成全部广播，BTC 总额 119,756 到达攻击者地址。
- **受损"合约"**：Bitfinex 内部后端代码（闭源），BitGo co-signer 服务代码（闭源）；无开源 commit 可引用。代码归档参考 rekt 未收录，官方事后公告 `bitfinex.com/posts/200`。
- **tx hash**：被盗 BTC 集中地址公开信息丰富，最关键为 2022-02 美司法部查扣时公告的地址 `bc1qazcm763858nkj2dj986etajv6wquslv8uxwczt`；2,072 笔原始 tx 完整列表在 Bitfinex 社区分析贴与 Chainalysis 报告中逐条可查。
- **为何审计未发现**：多签架构本身是安全的；问题在"策略配置"与"运营 API 凭证"层，属于非标审计域。

## 4. 事后响应

- **项目方**：
  - 全账户 36% 比例削减，发行 BFX token 代偿；2017-04 全部赎回。
  - 架构改造：不再使用"每客户独立多签地址"，转向"少量大额冷钱包 + 小量热钱包 + 自建 co-signer"模式。
  - 为此后与 BitGo 的合作变化铺路，多签厂商普遍开始推 MPC 方案（Fireblocks 2018 年成立，Curv 同年）。
- **资产追回**：
  - 2022-02-08 美司法部查扣 94,636 BTC，宣布按破产程序将 BTC 返还给 iFinex/Bitfinex，最终用于偿还 RRT token 持有者（Bitfinex 2016 发行的"追回权"代币）。
  - 2024-03：Bitfinex 完成 RRT 清偿。
- **执法**：Lichtenstein、Morgan 于 2022 被捕，2023 认罪，2024 分别判 5 年与 18 个月。
- **行业连锁**：
  - MPC（Multi-Party Computation）签名成为 2018 年后 CEX 与托管商的主流方案。
  - "策略层"风险审计被列入 SOC2 类评估。

## 5. 启发与教训

- **对开发者 / CEX**：
  - 多签 / MPC 必须将多把 key 分散到不同失败域（不同公司、不同网络、不同硬件）。
  - 风控策略必须设置硬编码的"每小时 / 每日总提现上限"，超出立即人工审批。
  - API 凭证要绑定 IP allowlist、短 TTL、且独立于业务数据库。
- **对审计方**：密码学正确的多签架构并非终点；"策略 + API 凭证 + 运营流程"需三位一体审计。
- **对用户**：再次证明 CEX 不可完全替代自托管；Bitfinex 的"socialized loss"模型虽罕见但合规上可行。
- **对行业**：即便 6 年后资产追回，用户体验上的 36% 削减与多年无偿等待仍是沉重代价。追回成功归功于链上不可篡改——这也从反面证明比特币反洗钱难度对攻击者是系统性不利。

## 6. 参考资料

- Bitfinex 官方公告：《Security Breach》2016-08-02（bitfinex.com/posts/200）
- BitGo 官方声明：《Statement on Bitfinex》2016-08-02
- CoinDesk：《Bitfinex Loss Is 119,756 BTC, Worth $65 Million》2016-08-03
- 美国司法部 2022-02-08 新闻稿：《Two Arrested for Alleged Conspiracy to Launder $4.5 Billion in Stolen Cryptocurrency》
- Chainalysis：《Bitfinex Hack Recovery》2022-02 blog
- Bitfinex Leo Whitepaper（2019）：描述 RRT 与 LEO 用于未追回部分补偿
- SlowMist 年度回顾：2016 CEX 安全事件章节

---

*Last verified: 2026-04-23*
