---
title: KuCoin 热钱包私钥泄露（2020-09-26, ~$285M）
module: 08-security-incidents
category: CEX-hack
date: 2020-09-26
loss_usd: 285000000
chain: [Ethereum, Bitcoin, BSC, Stellar, Tron, Ontology]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.kucoin.com/news/en-kucoin-security-incident-update-announcement
  - https://www.kucoin.com/news/en-kucoin-ceo-live-stream
  - https://peckshield.medium.com/kucoin-incident-post-mortem-c56ba5dfb36c
  - https://rekt.news/kucoin-rekt/
  - https://blog.chainalysis.com/reports/lazarus-group-kucoin-exchange-hack/
tx_hashes:
  - 未公开完整列表（Ethereum 主嫌疑地址 0xEB31973E0FeBF3e3D7058234a5eBbAe1aB4B8c23；BTC 主嫌疑地址 bc1q...）
---

# KuCoin 热钱包大规模私钥泄露

> **TL;DR**：2020-09-26，新加坡 CEX KuCoin 的 Ethereum / Bitcoin / 多条公链热钱包私钥泄露，攻击者在数小时内将约 **$285M** 资产（ETH、BTC、ERC-20、Stellar、Tron 等）转出。Chainalysis 在 2021-02 的正式报告中将本次攻击归因于朝鲜 **Lazarus Group**。KuCoin 通过自有保险基金 + 项目方冻结/重发 + 执法配合，最终追回约 **84%** 资产，是 CEX 被盗史上最成功的追回案例之一。

## 1. 事件背景

- **主体**：KuCoin，总部塞舌尔、运营在新加坡的大型 CEX；事件前 24h 交易量约 $1.4B，注册用户 > 500 万，支持 200+ 代币。
- **热钱包架构**：为支持大量币种与快速提币，KuCoin 为每条公链维护独立热钱包，部分使用第三方托管方案 BitGo。事件中被泄露的主要为 Ethereum 主热钱包、Bitcoin 热钱包以及若干 ERC-20 聚合钱包。
- **时间轴**：
  - 2020-09-25 19:05 UTC：Ethereum 热钱包异常转出（首笔大额外转给攻击者地址）。
  - 2020-09-25 21:31–23:49 UTC：数十笔批量外转 ETH、LINK、OMG、USDT 等至 `0xeb31...8c23` 等攻击者地址。
  - 2020-09-26 02:51 UTC：BTC 热钱包开始外转 ~1,008 BTC（约 $10.8M）。
  - 2020-09-26 清晨：KuCoin 风控察觉；约早 07:00 内部停止热钱包活动。
  - 2020-09-26 10:25 UTC：KuCoin 官方公告暂停充提。
  - 2020-09-26 15:00 UTC：Johnny Lyu（CEO）直播说明，初步估损 $150M；后修正为 **$285M**。
  - 2020-09-27–10-11：Bitfinex、Binance、Huobi、OKEx、Tether、项目方（Velo、Orion、KardiaChain、Ocean、AMPL 等）相继冻结或重发相应资产。
  - 2020-11-11：KuCoin 宣布已追回 **84%**（~$239M）；剩余部分由保险基金 + 公司储备覆盖，用户未损失。
  - 2021-02-24：Chainalysis 发布报告，将此事件归因于 Lazarus Group。
- **发现过程**：内部风控 + PeckShield 等链上监控先后告警。

## 2. 事件影响

- **直接损失**（2020-09-26 快照）：
  - ETH：~11,480 ETH，约 $4M。
  - BTC：~1,008 BTC，约 $10.8M。
  - ERC-20：价值约 $153M 的 USDT、LINK、OMG、CRO、KCS 等。
  - 其他链（XLM、XRP、TRX、ONT 等）合计约 $117M。
  - 总计 **~$285M**（KuCoin 官方口径）。
- **受害方**：KuCoin 公司（用户资产未被动用保险基金以外的部分，最终 100% 赔付）。
- **连带影响**：
  - **项目方连锁冻结**：数十个代币项目在 24–72 小时内冻结或"硬分叉/重发"被盗资产——Velo Labs 冻结 12.7M VELO、Orion Protocol 冻结 3.8M ORN、KardiaChain 暂停并迁移 1.2B KAI、Ocean Protocol 冻结 21.5M OCEAN、AMPL 冻结 3.4M AMPL、NOIA/COTI/DIA 等多家跟进。
  - **DEX 被盗清洗路径暴露**：攻击者第一次大规模在 Uniswap / 1inch / Kyber 上把几十种 ERC-20 砸盘换 ETH，后世因此把 "DEX 洗币" 纳入 CEX 事件监控重点。
  - **去中心化争议**：关于"项目方是否应为了救援 KuCoin 修改合约"引发社区辩论，Algorand、Tether 式中心化权力被重新讨论。
- **资金去向**：
  - 追回路径：项目方 freeze/migrate（约 $129M）+ 交易所冻结 + 白帽谈判 + 司法协助（约 $110M）。
  - 未追回：部分 BTC、ETH 通过 Tornado Cash / ChipMixer / 跨链桥混币后进入 Lazarus 常用洗钱基础设施（Chainalysis 追踪）。

## 3. 技术根因（代码级分析）

- **漏洞分类**：Key-Mgmt / 运营安全（非合约漏洞）。
- **受损范围**：KuCoin 多条链的热钱包私钥同时失窃。官方未公开具体路径，但 Chainalysis 2021-02 报告及社区分析指向以下组合：
  1. **社工 + 鱼叉攻击**：Lazarus Group 2020 年在加密行业大规模使用"虚假招聘（LinkedIn 伪装 HR + 投递含宏恶意 Word/PDF）"与"虚假业务合作"模式。多家同期被攻破（如 2020-06 CoinJoin Exchange CoinMetro、DragonEx 2019-03、Liquid 2021-08）。
  2. **Watering-hole / 内部员工终端入侵**：攻击者在 KuCoin 运维 / DevOps 终端植入木马，窃取私钥文件或内存中的签名凭证。
  3. **热钱包签名服务被劫持**：非多签/HSM 的热钱包在运维机器上以文件或 KMS 可解密形式保存私钥。
- **为何一次能掏空多条链**：KuCoin 彼时部分热钱包采用"同一组运维 / 同一身份凭证访问多条链 KMS 密钥"的架构，单点突破带来跨链损失。
- **代码/架构层面**：非合约漏洞，不涉及 Solidity bug；但攻击者在清洗阶段调用了 Uniswap V2 `swapExactTokensForETH`、1inch aggregator、Kyber、Bancor 等 DEX 的标准接口——这推动行业在 2021 年开始要求 CEX 热钱包**对外转交易加白名单 + 数额速率限制**，因为 DEX 清洗在技术上完全合规。

### 3.1 Lazarus 归因（严格依据官方报告）

Chainalysis 《Lazarus Group and the KuCoin Exchange Hack》报告（2021-02-09 发布）明确指出：

> "Based on our investigation, the North Korean Lazarus Group — which was previously implicated in the $250 million hack of an unnamed cryptocurrency exchange — is likely the culprit behind the KuCoin hack."

归因证据链：
1. 钱包聚类与 2019 年 DragonEx 事件、2020 年其他 Lazarus 归属事件共享地址簇。
2. 洗钱模式匹配 Lazarus 标志性的 "peel chain + Bitcoin mixer + P2P OTC" 三段式。
3. Bitcoin 阶段使用 ChipMixer 与 Wasabi CoinJoin；以太坊阶段大量使用 Tornado Cash。

**严格遵守归因规则**：以上归因引自 Chainalysis 正式报告，其他未见于 Chainalysis / FBI / ZachXBT 的细节按"归因未公开"处理。

## 4. 事后响应

- **KuCoin 官方**：
  - 2020-09-26 15:00 UTC：CEO Johnny Lyu 直播承诺"用户资产 100% 由 KuCoin 与保险基金覆盖"。
  - 2020-09-30–10-04：逐链恢复充提。
  - 2020-11-11：宣布追回 84% 资产；事件基本平息，未有用户实际损失。
- **项目方联动**：
  - Velo Labs 通过合约 blacklist 冻结被盗 VELO。
  - Orion Protocol 回滚 ORN 合约并空投新代币。
  - KardiaChain、Ocean Protocol、COTI、AMPL、NOIA、DIA、Covesting 等多家通过合约升级 / 硬分叉进行救援。
  - Tether 冻结相关 USDT 地址（约 $22M）。
- **执法**：新加坡警察部队（SPF）、美国 FBI、韩国警方共同参与；部分追回通过交易所冻结 + 司法协助完成。
- **事后审计**：KuCoin 未公开具体架构升级细节，但声明将切换至更严格的 BitGo 多签 + 自研 MPC 混合方案，冷热比例调整为 98/2。
- **行业连锁**：
  - 2020 年 Q4 起，DeversiFi、Liquid、AscendEX 等多家 CEX 重新评估热钱包架构。
  - Chainalysis、Elliptic 将 KuCoin 事件纳入"国家级威胁案例库"，成为此后多年 CEX 尽调样板。

## 5. 启发与教训

- **交易所（核心）**：
  - **热钱包必须采用 MPC / HSM + 多签门限**；单机保留明文私钥是最高风险；跨链热钱包禁止共享凭证。
  - **外转白名单 + 速率限制 + 人工复核**：任何热钱包在单位时间内超过阈值的外转应自动阻断；尤其需识别 `DEX router` 作为目的地的批量 swap 模式。
  - **供应链安全**：LinkedIn/招聘链路是 Lazarus 2019–2022 反复使用的入口；DevOps、财务、安全团队成员应进行反钓鱼训练、使用专用隔离设备。
  - **Incident Response**：KuCoin 从"发现到公告"用时 < 12 小时，从"事件到 100% 用户赔付承诺"用时 < 24 小时——这是本事件能维持用户信任的关键。
- **项目方**：
  - 发行代币时应评估"遇交易所被盗是否有能力冻结/重发"的权力面；这是去中心化与风险之间的真实 trade-off。
  - Velo / Orion / KardiaChain 的救援行动在业界有争议，但也证明了"黑名单/升级权"在极端事件中的实用性。
- **用户**：
  - 长期持仓不应留在 CEX 热钱包；使用硬件钱包或合规托管（Coinbase Custody、Anchorage 等）。
  - 评估 CEX 时看 3 个指标：Proof of Reserves、保险基金规模、历史事件响应速度。
- **行业**：
  - Chainalysis 归因技术（地址聚类 + 行为模式）自此成为重大事件的"事实标准"。
  - 本案是"DEX 洗币"进入主流意识的标志性事件，推动 Tornado Cash 在 2021–2022 年被密集讨论与最终 OFAC 制裁。

## 6. 参考资料

- KuCoin 事件更新：<https://www.kucoin.com/news/en-kucoin-security-incident-update-announcement>
- KuCoin CEO 直播说明：<https://www.kucoin.com/news/en-kucoin-ceo-live-stream>
- Chainalysis Lazarus 归因报告：<https://blog.chainalysis.com/reports/lazarus-group-kucoin-exchange-hack/>
- PeckShield 分析：<https://peckshield.medium.com/kucoin-incident-post-mortem-c56ba5dfb36c>
- rekt.news：<https://rekt.news/kucoin-rekt/>
- SlowMist 被盗数据库：<https://hacked.slowmist.io/>（KuCoin 2020-09-26）
- Elliptic 技术追踪：<https://www.elliptic.co/blog/kucoin-hack>
- 攻击者以太坊地址之一：<https://etherscan.io/address/0xeb31973e0febf3e3d7058234a5ebbae1ab4b8c23>
- Velo 冻结公告：<https://velo.org/kucoin-security-incident/>
- Orion Protocol 事件总结：<https://blog.orionprotocol.io/security-incident-update/>

---

*Last verified: 2026-04-23*
