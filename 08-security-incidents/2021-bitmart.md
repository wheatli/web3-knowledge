---
title: BitMart 热钱包私钥泄露（2021-12-04, ~$196M）
module: 08-security-incidents
category: CEX-hack
date: 2021-12-04
loss_usd: 196000000
chain: [Ethereum, BSC]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://peckshield.medium.com/bitmart-hack-a-case-study-9f8a28a06c4f
  - https://rekt.news/bitmart-rekt/
  - https://twitter.com/peckshield/status/1467196477035024384
  - https://support.bmx.fund/hc/en-us/articles/4416490619291
tx_hashes:
  - 未公开（PeckShield 披露的主嫌疑地址：0x39fb0dcd13945b835d47410ae0de7181d3edf270）
---

# BitMart 热钱包私钥泄露

> **TL;DR**：2021-12-04，CEX BitMart 的以太坊 / BSC 热钱包遭受大规模提款异动，PeckShield 监控系统首先发现 ~$100M 以太坊资产 + ~$96M BSC 资产被转出，判定为私钥泄露类事件。BitMart 创始人 Sheldon Xia 随后确认，承诺用公司自有资金赔付用户。总损失按当日快照约 **$196M**，是 2021 年规模最大的 CEX 热钱包被盗事件之一。

## 1. 事件背景

- **主体**：BitMart 为中等规模中心化交易所，注册地开曼，事件前 24h 交易量约 $2B。
- **热钱包架构**：Ethereum / BSC 各维护一个或多个 EOA，用以承接充值、聚合小额。事发主钱包（Ethereum）`0x68b22215...`、BSC `0x68b22215ff...`（同私钥）。
- **时间轴**：
  - 2021-12-04 17:30 UTC（约）：Ethereum 热钱包资产开始被批量 `swap → withdraw`。
  - 2021-12-04 19:00 UTC：PeckShield 推特披露异常转账，初步估算 $100M。
  - 2021-12-05 05:00 UTC：PeckShield 补充 BSC 侧 $96M 损失。
  - 2021-12-05：BitMart 公告暂停提币；创始人发推确认事件。
  - 2021-12-10：BitMart 官方恢复提币并开始分批赔付。
- **发现过程**：PeckShield 链上监控（非 BitMart 内部告警）首先拉响警报。

## 2. 事件影响

- **直接损失**：
  - Ethereum：约 **$100M**，包含 SAFEMOON、SHIB、BabyDoge、FLOKI、BSCPAD 等多达 20+ 种 ERC-20 + 少量 ETH。
  - BSC：约 **$96M**，包含 BSC-PEG 资产、BNB、CAKE、SAFEMOON、BPAY 等。
  - 合计 **~$196M**（2021-12-04 快照）。
- **受害方**：BitMart 公司自掏腰包赔付所有用户；短期 BitMart 交易量与平台币 BMX 显著下跌。
- **连带影响**：2021 年 Q4 CEX 热钱包事件再现高峰（AscendEX 12-12、BitMart 12-04、Crypto.com 2022-01）；业界重新审视"热钱包资产比例 + 多签门限 + HSM 托管"。
- **资金去向**：攻击者迅速把小市值代币在 1inch / PancakeSwap 上卖成 ETH / BNB，随后通过 Tornado Cash 清洗。SAFEMOON 等项目方短暂讨论是否冻结/黑名单，最终未集体行动。

## 3. 技术根因

- **漏洞分类**：私钥管理 / 运营安全（非合约漏洞）。
- **受损范围**：BitMart 的 Ethereum / BSC 热钱包私钥。两条链使用**同一把私钥**导致单点被攻破即跨链损失。
- **根因推测**（BitMart 未披露细节，以下为业界通行判断）：
  1. 热钱包私钥可能以明文或可被运维访问的形式存放在运维服务器上，未使用 HSM。
  2. 热钱包余额过高——在交易所中约占整体储备的 ~2%，但在 $196M 量级仍远超必要流动性。
  3. 内部告警与提币阻断阈值未及时触发。
- **代码/架构层问题**：热钱包不是智能合约漏洞，因此没有"代码片段"可引用；但架构缺陷清晰——缺乏 MPC/门限签名、缺乏跨链私钥隔离、缺乏实时异常外转阻断。PeckShield 分析中指出攻击者在 30 分钟内完成了对 20+ 资产的 Uniswap/PancakeSwap 兑换，但无任何风控触发——说明 BitMart 缺少"单位时间外转金额上限"的自动阻断。
- **归因**：归因未公开（Chainalysis / FBI 未发布针对 BitMart 的正式归因报告；社区有对 DPRK-linked 的猜测但缺乏官方确认）。

## 4. 事后响应

- **项目方**：BitMart 暂停所有提币约 5 日；Sheldon Xia 12-05 发推承诺"使用公司自有资金全额赔付"，12-10 重启提币并启动赔付流程。
- **安全加固**（官方声明）：升级钱包管理系统、加入多签、提高冷热分离比例；但未公开具体架构方案。
- **法律 / 执法**：未见 FBI / Chainalysis 针对本次事件的公开起诉；部分资金路径被纳入 Elliptic、Chainalysis 标签库。
- **行业连锁**：AscendEX 于 2021-12-12 发生同类 ~$77M 事件，业内普遍认为并非同一攻击者但方法相似——提示整个 2021 年底 CEX 热钱包安全姿态集体不足。

## 5. 启发与教训

- **交易所**：
  - 热钱包与冷钱包私钥分离；跨链热钱包绝不可共用同一把私钥。
  - 采用 MPC / 门限签名 / HSM；消除"运维可触达明文私钥"的路径。
  - 对单位时间外转金额、非白名单地址外转、新代币兑换等设置实时阻断与人工复核。
  - 热钱包余额应以"未来 N 小时预期提币 + 安全边际"动态调整，不应堆积数亿美元。
- **用户**：评估 CEX 时关注其冷热分离比例、是否公开 Proof of Reserves、是否有保险基金。
- **监管/行业**：CEX 应强制披露 incident response SLA 与储备构成；2022 年 FTX 事件后 Proof of Reserves 成为行业标配，BitMart 事件是推动这一标准化的前序案例之一。

## 6. 参考资料

- PeckShield 分析：<https://peckshield.medium.com/bitmart-hack-a-case-study-9f8a28a06c4f>
- PeckShield 推特首发：<https://twitter.com/peckshield/status/1467196477035024384>
- rekt.news：<https://rekt.news/bitmart-rekt/>
- BitMart 官方公告：<https://support.bmx.fund/hc/en-us/articles/4416490619291>
- SlowMist 被盗数据库：<https://hacked.slowmist.io/>（检索 BitMart 2021-12-04）
- 嫌疑地址：<https://etherscan.io/address/0x39fb0dcd13945b835d47410ae0de7181d3edf270>

---

*Last verified: 2026-04-23*
