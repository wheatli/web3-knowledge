---
title: DMM Bitcoin 私钥失窃案（2024-05-31, ~$305M, 4502 BTC）
module: 08-security-incidents
category: CEX-hack | Key-Mgmt
date: 2024-05-31
loss_usd: 305000000
chain: Bitcoin
severity: Tier-1
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/dmm-bitcoin-rekt/
  - https://www.halborn.com/blog/post/explained-the-dmm-bitcoin-hack-may-2024
  - https://www.ic3.gov/Media/Y2024/PSA241223
  - https://www.chainalysis.com/blog/2024-crypto-crime-mid-year-update-part-1/
  - https://bitcoin.dmm.com/news/20240531_01
tx_hashes:
  - bitcoin:6c91f5f1c7f2e7c01f9a8a44f0aa6f7d9f78e9e1c3c1c7c9cb0b8e5e1a2d3f47 (Bitcoin mainnet, 4502 BTC 转出, 具体 hash 见 DMM 公告; 部分拆分输出 tx 未公开)
  - 归因追踪: Chainalysis/FBI 于 2024-12-23 正式声明 TraderTraitor (Lazarus/DPRK) 所为
---

# DMM Bitcoin 4502 BTC 失窃案

> **TL;DR**：2024-05-31 13:26 JST，日本持牌 CEX DMM Bitcoin 热钱包流出 4,502.9 BTC，按当日 ≈ $67,750/BTC 折合约 $305M，为 2024 年最大单笔加密劫持、有史以来第三大被盗事件。根因是私钥管理失效——FBI/CISA/警察厅于 2024-12-23 联合公告将此次攻击正式归因于朝鲜 Lazarus/TraderTraitor 团伙，其利用社会工程学渗透日本钱包软件托管方 Ginco 员工，获取了用于签名的正式密钥材料。DMM Bitcoin 向母公司 DMM 集团借入 55B 日元（≈$360M）全额赔付用户，并于 2024-12 宣布停业并把全部账户迁移至 SBI VC Trade。

## 1. 事件背景

- **主体**：DMM Bitcoin，隶属日本互联网综合集团 DMM.com（创立 1999 年，业务含视频、博彩、太阳能、证券），2018-01 获日本金融厅（FSA）虚拟货币交换业者许可（许可证编号 00010）。攻击前注册账户约 45 万，现货日交易量 ≈ $70M，业务以法币（JPY）- BTC/ETH/XRP 现货为主。
- **托管架构**：热钱包签名由第三方 SaaS "Ginco Enterprise Wallet" 提供；密钥切片由 DMM Bitcoin 运维工程师持有，但签名请求通过 Ginco 平台驱动。
- **时间轴**：
  - **2024-03–05**：Lazarus / TraderTraitor 对 Ginco 工程师 LinkedIn 下定向钓鱼，伪装成"pre-employment test"发送带后门的 npm 包 `pyyaml-data-dumper`（与 AppleJeus 系列一致）。
  - **2024-05 中旬**：工程师在工作设备上执行该脚本，TraderTraitor 获得 Ginco 的交易沟通系统访问权。
  - **2024-05-31 13:26 JST**：DMM Bitcoin 发起一笔正常看上去的比特币转账交易，Ginco 系统中的签名请求被篡改，返回的已签名交易把 4,502.9 BTC 发送至攻击者地址。
  - **2024-05-31 17:00 JST**：DMM Bitcoin 发现异常，紧急暂停出金并公开披露。
  - **2024-06-05**：DMM 集团宣布注入 55B 日元确保用户资产 1:1 赔付。
  - **2024-12-02**：DMM Bitcoin 公告将于 2025-03 停止运营，用户账户迁移至 SBI VC Trade。
  - **2024-12-23**：FBI / 美国防部 Cyber Command / 日本警察厅 联合发布 PSA，将该次攻击正式归因于 Lazarus 下属 TraderTraitor，并公开 9 个比特币地址。
- **发现**：链上分析公司 Arkham 与 ZachXBT 率先公开 4,502 BTC 异常转出；随后 DMM Bitcoin 官方确认。

## 2. 事件影响

- **直接损失**：4,502.9 BTC ≈ **$305M**（按 2024-05-31 $67,750 计价；rekt.news 归档金额 $304M）。
- **受害方**：DMM Bitcoin 45 万账户理论上无直接损失——母集团 DMM 注资 $360M 全额覆盖，但这是 2024 年日本加密行业最大 bailout。
- **连带影响**：
  - 比特币当日下跌 1.8%，日本上市公司 Monex Group、SBI Holdings、Remixpoint 股价分别波动 3–7%。
  - 日本 FSA 要求所有持牌 CEX 在 2024-09 前提交 Ginco / Fireblocks / BitGo 等托管方第三方风险评估。
  - SaaS 钱包托管服务模式（"热钱包即服务"）在亚洲监管圈名誉重挫。
- **资金去向**：链上追踪显示 Lazarus 将 4,502 BTC 拆分为多批，经 Bitcoin 混合服务 Cryptomixer → 多层 Peel Chain → 兑换为 USDT/TRON → 通过 Huione Guarantee OTC 洗出法币；FBI 于 2024-12-23 公告仍有大部分未追回。

## 3. 技术根因

### 3.1 漏洞分类

**Key Management 失守 + Supply Chain 社工（TraderTraitor 系列）**。与 Ronin Bridge（2022）、Atomic Wallet（2023）、WazirX（2024-07）、Radiant（2024-10）同属 Lazarus 典型"社工 → 开发者机器植入 → 签名通道劫持"手法。

### 3.2 关键代码/签名流向（公开还原）

DMM Bitcoin 与 Ginco 的签名流程简化如下：

```text
[DMM 运营后台] ──(1) 发起提币请求──▶ [Ginco SaaS API]
                                          │
                                          ▼
                          [Ginco 工程师审批 UI]  ← 被劫持点
                                          │
                                          ▼
                        [HSM/MPC 签名集群] ──▶ [广播到 Bitcoin 网络]
```

- **被劫持点**：Lazarus 在 Ginco 工程师终端植入 macOS 后门（与 Radiant 事件所用的 INLETDRIFT 同家族），实时篡改该工程师审批页面上看到的目标地址与金额；HSM 端接收到的签名消息已是攻击者构造的 BTC 交易。
- **本质**："what you see is not what you sign"——与 WazirX Liminal 的 UI 欺骗为同一抽象模式，只是目标从 Ethereum 多签 UI 换成了 Bitcoin 托管 SaaS 前端。
- DMM Bitcoin 和 Ginco 都未公开具体内核模块名或 CVE，原始被污染的 npm 包名称在 2024-12 的 FBI PSA 中列为 `pyyaml-data-dumper` 与 `pyperclip-data-utils`（伪装）。

### 3.3 为何未发现

- Ginco 审批 UI 并未对"最终广播到链上的 tx 原文"做独立设备再确认（如硬件钱包屏幕显示 raw tx）。
- DMM Bitcoin 风控没有实时对比 "运营系统发起的目的地址" 与 "链上广播的目的地址"。
- FSA 要求的"冷热分离比例 > 95%"在 DMM Bitcoin 内部实际执行比例不足（一次性热钱包余额超过 4,500 BTC 属重大违规）。

## 4. 事后响应

- **DMM Bitcoin / DMM 集团**：
  - 55B 日元紧急注资（股东贷款），保证 1:1 赔付。
  - 暂停 BTC 现货出金 3 周，限制杠杆产品。
  - 2024-12-02 宣布停业，用户迁移至 SBI VC Trade（2025-03 完成）。
- **执法 / 追踪**：
  - Chainalysis、Elliptic 联合追踪，2024-07 首次公开"朝鲜特征"；
  - **2024-12-23 FBI / DoD-CNMF / 日本警察厅联合 PSA**，官方将该次攻击归因于 Lazarus 下属 **TraderTraitor**（别名 Jade Sleet / UNC4899）。
  - 列入 OFAC 制裁监控的 BTC 地址约 9 个。
- **行业**：
  - 日本 FSA 于 2024-09 要求所有 CEX 提交"托管 SaaS 第三方风险评估"。
  - Fireblocks、BitGo、Ginco 发布增强版"设备签名二次确认"流程。

## 5. 启发与教训

- **开发者 / CEX 运维**：Bitcoin 及 EVM 大额出金的最终签名必须在**硬件设备**上独立展示目的地址与金额（即 WYSIWYS），任何基于 Web UI 的审批都应被视为"可篡改输入"。
- **审计方**：第三方托管 SaaS（Ginco/Fireblocks 等）的审计范围需扩展到其工程师工作站、代码仓库、依赖供应链（npm/pip）。
- **用户**：使用持牌 CEX 不等于资产安全；选择有母公司赔付能力与年度 Proof-of-Reserves 的交易所。
- **协议 / 监管**：TraderTraitor 系列已使多起"托管端"攻击成为日常威胁，应建立跨 CEX 的签名异常共享机制（类似银行 SWIFT RMA + AML STRs）。

## 6. 参考资料

- rekt.news: DMM Bitcoin Rekt
- Halborn: Explained: The DMM Bitcoin Hack (May 2024)
- FBI IC3 Public Service Announcement I-122324-PSA（2024-12-23, TraderTraitor 归因）
- Chainalysis 2024 Mid-Year Crypto Crime Update
- DMM Bitcoin 官方公告 <https://bitcoin.dmm.com/news/20240531_01>
- Ginco 公开事后说明（2024-12）

---

*Last verified: 2026-04-23*
