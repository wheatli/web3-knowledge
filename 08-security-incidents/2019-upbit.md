---
title: Upbit 热钱包 342,000 ETH 被盗 $49M（2019-11-27）
module: 08-security-incidents
category: CEX-hack
date: 2019-11-27
loss_usd: 49000000
chain: [Ethereum]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://upbit.com/service_center/notice?id=1681
  - https://www.coindesk.com/markets/2019/11/27/south-korean-crypto-exchange-upbit-loses-49-million-in-ether-hack
  - https://slowmist.medium.com/upbit-was-attacked-an-update-from-slowmist-inc-91cb4e7b3796
  - https://www.chainalysis.com/blog/north-korea-cryptocurrency-hacks-2022/
  - https://www.justice.gov/opa/pr/department-justice-seizes-330000-stolen-virtual-currency-proceeds-massive-hack-south-korean
tx_hashes:
  - Upbit 热钱包地址：0x390de26d772D2e2005C6D1d24afC902bae37a4bB
  - 攻击 tx（聚合）：0xa09d73b0aae4a8aa65a34cfe9f277d36884bd23e3fe10b03fe8f8fa0cd09eabf
  - 攻击者目标地址：0xa09c4EE...（后续多地址分散）
---

# Upbit 热钱包 342k ETH 被盗

> **TL;DR**：2019 年 11 月 27 日 13:06 KST，韩国最大交易所之一 Upbit 的 ETH 热钱包被一次性提走 342,000 ETH（约 $49M）。Upbit 当日公告承认损失，称为"异常提现"，并宣布以公司自有资金全额兜底用户。链上 ETH 迅速在数千地址分散，2020–2022 年间经混币器 Wasabi、Tornado Cash 与韩国 P2P 交易所分散出金。2020-03 美国司法部、2023 FBI 与韩国警察厅先后在公开文件中将 Upbit 事件归因于朝鲜 Lazarus Group / APT38，Chainalysis 在《North Korean Crypto Hacks》系列报告中多次引用此案。

## 1. 事件背景

- **主体**：Upbit 由韩国 Dunamu 运营，2017-10 上线，由于其与 Kakao 生态绑定和本土化合规，2019 年已是韩国成交量第一。
- **时间轴**：
  - 2019-11-27 13:06 KST（04:06 UTC）：Upbit 热钱包地址 `0x390De…a4bB` 发起一笔 tx，转出 342,000 ETH 至攻击者地址。
  - 2019-11-27 13:37 KST：Upbit 官方紧急公告承认"异常转账"，暂停所有虚拟资产存取款。
  - 2019-11-28：Upbit 承诺 2 周内用自有资金全额兜底。
  - 2020-01：韩国警察厅与 Upbit 联合发布初步调查，认为是"外部黑客"作案。
  - 2020-11：Upbit 完成系统迁移并恢复全部服务。
  - 2022-02：Chainalysis 在《Crypto Crime Report 2022》中将 Upbit 入金流向 Lazarus 专属地址簇。
  - 2024-11：美国司法部宣布追回约 $330,000 的 Upbit 被盗资金（via Hydra 扣押链路）。
  - 2023-11：韩国警察厅正式公布调查结果，归因 Lazarus / Andariel。
- **发现过程**：Upbit 风控系统发现单笔大额出金；运维在 30 分钟内暂停全站。

## 2. 事件影响

- **直接损失**：342,000 ETH（~$49M 按 2019-11-27 ~$145 估）。
- **受害方**：Upbit 自担全部损失，用户资金完整。
- **连带影响**：
  - ETH 价格短时下跌 ~4%。
  - 韩国金融监督院（FSS）加强对虚拟资产交易所监管，催化后续《特定金融信息法》2021 年修订。
  - 多家韩国 CEX 加强了与 HSM 厂商、MPC 厂商的合作。
- **资金去向**：
  - ETH 分散至数百地址 → ERC-20 转换为 WBTC、DAI → 桥至 BTC 链 → 混币器 → 多家 P2P 交易所出金。
  - Chainalysis 与链上侦探 ZachXBT 指出其中大量 BTC 经 ChipMixer、Wasabi CoinJoin 清洗。
  - 美国司法部 2024-11 扣押约 $330k（远小于全额）。
- **归因**：Chainalysis 2022 Crypto Crime Report + FBI 2023 联合公告 + 韩国警察厅 2023-11 正式报告均指向 **Lazarus Group**。

## 3. 技术根因

- **漏洞分类**：Key-Mgmt（热钱包私钥泄漏）+ 可能伴随的 Endpoint / Social-Eng；Upbit 官方未公开具体入侵路径。
- **关键事实**（综合 Upbit 公告、慢雾分析、Chainalysis 报告）：
  1. Upbit 的 ETH 热钱包为单地址结构，私钥或签名流程在某种方式下被攻击者获取。
  2. 未观察到智能合约层的漏洞：此为标准的 EOA → EOA 转账，且签名有效。
  3. Upbit 的官方定性是"异常提现"；慢雾与韩国警察厅调查倾向于"内部签名机被入侵"（包括可能的内部人员账户被钓鱼 + 恶意软件植入）。
  4. Chainalysis 归因 Lazarus Group 的理由包括：出金路径与后来 Ronin、Harmony 等 Lazarus 事件的混币链路高度重合；入金交易所地址集群重合；Wasabi CoinJoin peel-chain 模式一致。
- **受损模块**：Upbit 内部签名机（闭源）。
- **伪代码示意（热钱包签名的典型弱点）**：

```python
# 概念模型：Upbit 热钱包签名机架构（假设）
# 一旦签名机被攻陷，攻击者可构造任意 tx 并取得合法签名
def sign_and_broadcast(to_addr, amount):
    nonce = web3.eth.get_transaction_count(hot_wallet_addr)
    tx = { "to": to_addr, "value": amount, "nonce": nonce, "gas": 21000, ... }
    # 缺陷：无业务层"收款地址白名单"最终门槛
    #       无"单笔/单日阈值 + 人工复核"二次确认
    signed = account.sign_transaction(tx, hot_wallet_privkey)
    web3.eth.send_raw_transaction(signed.rawTransaction)
```

- **攻击步骤**（基于公开报告推断）：
  1. 攻击者对 Upbit 员工长期侦察 + 钓鱼（Lazarus 标准套路：LinkedIn 社工 + 恶意 PDF/Excel 文档）。
  2. 植入后门，横向移动获得 ETH 热钱包签名机访问。
  3. 2019-11-27 04:06 UTC 在签名机上构造 342k ETH 的单笔转账 tx。
  4. ETH 在数分钟内分散至多个攻击者地址，开始链上洗白。
- **tx hash**：核心攻击 tx `0xa09d73b0...cd09eabf`（Etherscan 可查），源地址即 Upbit 热钱包 `0x390De26d...a4bB`。
- **为何审计未发现**：韩国当时对大型交易所的合规审计未深入到签名机架构细节；Lazarus 的 APT 级社工在多数防御体系前均难完全防御。

## 4. 事后响应

- **项目方**：
  - Dunamu 宣布以公司自有 ETH 全额兜底，无用户损失。
  - 停机约 2 周升级安全架构（细节未全部公开，但宣布启用更严格的冷热钱包比例）。
  - 与慢雾等安全厂商合作长期追踪资金。
- **资产追回**：2024-11 美国司法部查扣约 $330,000（~3 BTC 级别）作为象征性回收；整体追回率极低。
- **执法**：
  - 韩国警察厅 2023-11 归因 Lazarus / Andariel，但嫌疑人远在朝鲜，无法逮捕。
  - FBI 2023-08 公开 Upbit 追踪地址，列入制裁地址名单。
- **行业连锁**：
  - 韩国 CEX 全面升级多签与 MPC；Dunamu 与 Fireblocks 等合作。
  - 推动 Travel Rule 在韩国 CEX 间落地（2022-03）。

## 5. 启发与教训

- **对 CEX**：
  - 热钱包应严格限额，单一地址不应持有 >$10M 级资产；采用分片多签或 MPC。
  - 签名机前的业务层应设独立审批，而非依赖签名机自身风控。
  - 所有员工终端需 EDR 全覆盖；LinkedIn / 邮件收到的 PDF、Office 文档必须沙箱打开。
- **对用户**：
  - Upbit 事件再次证明 CEX 的 SAFU 能力是关键：Dunamu 自担 $49M 并未破产。
  - 但不是所有交易所都有同等储备能力。
- **对协议 / 生态**：Lazarus 在 Upbit 之后持续升级手法，直到 2022 Ronin（$624M）才让全球真正意识到国家级威胁常态化；Upbit 事件是这个剧本的重要前置。

## 6. 参考资料

- Upbit 官方公告：`upbit.com/service_center/notice?id=1681`
- CoinDesk：《South Korean Crypto Exchange Upbit Loses $49 Million in Ether Hack》2019-11-27
- 慢雾 SlowMist Medium：《Upbit was attacked – an update from SlowMist Inc.》2019-11
- Chainalysis：《Crypto Crime Report 2022》 North Korea 章节
- 美国司法部 2024-11 新闻稿：《Department of Justice Seizes Stolen Virtual Currency Proceeds》
- 韩国警察厅 2023-11 调查结果新闻稿
- rekt.news / Elliptic 等二次分析

---

*Last verified: 2026-04-23*
