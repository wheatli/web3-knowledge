---
title: Multichain 跨链桥资金异常转出（2023-07-06, ~$125M+）
module: 08-security-incidents
category: Bridge | Key-Mgmt
date: 2023-07-06
loss_usd: 125000000
chain: Ethereum, Fantom, Moonriver, Polygon, zkSync, Arbitrum, Optimism, BNB Chain
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/multichain-exploit-analysis-c6c5b6b9b4a0
  - https://rekt.news/multichain-rekt/
  - https://www.certik.com/resources/blog/3KJQUJuFQK3VApQjMY6eIb-multichain-exit-scam-incident-analysis
  - https://twitter.com/MultichainOrg/status/1677180114227056641
  - https://www.elliptic.co/blog/multichain-suspected-125-million-hack
tx_hashes:
  - 0x9a6230c12340a34bd23e6bb70d9ca5a4bce1d0eb1afa71a66cba3d0ab9b95f3b (Fantom USDC 桥向未知地址转出)
  - 0xef6d4c1aa54e6a3e42f43e92a44b4fb29eac19d0bd99edd6c7b8e6c3e2d0d47f (Ethereum, WBTC 转出)
  - 未公开（跨 8 条链多笔）
---

# Multichain 跨链桥资金异常转出（"退出骗局式"事件）

> **TL;DR**：2023-07-06，跨链桥龙头 Multichain（前身 AnySwap）在无预告下从旗下多个链的 MPC 锁仓地址向未知外部地址转出 $125M+ 资产。根因并非智能合约漏洞，而是 MPC 私钥托管实际上高度中心化：CEO 赵均（Zhaojun）在 2023-05 已被中国警方拘留，其个人云端掌握全部密钥碎片，资金被其本人或接触到密钥的关联方转走。Multichain 此后永久关停。

## 1. 事件背景

- **项目**：Multichain（2020 fork 自 Anyswap，后改名）是 2021-2023 年 TVL 最大的跨链桥之一，支持 92+ 条链、4000+ 资产，峰值 TVL 超 $20B。自称采用去中心化 SMPC（Secure Multi-Party Computation）节点网络保管资产。
- **前兆**：
  - 2023-05-24：CEO 赵均及其姐姐在中国江苏被公安带走（Multichain 官方在 7-14 确认）。
  - 2023-05-31：Multichain 官方 Twitter 首次出现"部分跨链路径异常停滞"。
  - 2023-06-11：Fantom 上 Multichain 桥部分资产无法 redeem，Fantom 创始人 Andre Cronje 公开对抗。
- **资金转出**：
  - 2023-07-06 15:00 UTC 起，Fantom-Multichain 桥地址开始外转 USDC、DAI、WBTC、WETH。
  - 连续数小时内蔓延至 Moonriver、Dogechain、Kava、zkSync 等链。
- **官方披露**（2023-07-14）：Multichain 官方发推承认 CEO 被拘，所有运维服务器和 MPC 密钥均由 CEO 个人控制（"all the operational keys were under the CEO's control"），团队无法访问，宣布立即停止服务。

## 2. 事件影响

- **直接损失**（按当日价）：
  - Fantom 桥：~$122M（USDC 约 $30.9M、USDT $13.6M、DAI $7.5M、WBTC $19M 等）
  - Moonriver：~$6.5M
  - Dogechain：~$700k
  - Kava / zkSync / 其他：~$1M
  - 粗略合计：$125M – $130M（部分来源统计 $210M，含后续二次转出）
- **受害方**：
  - Fantom 链生态：USDC/USDT 几乎全部脱锚，Fantom TVL 单周跌 60%+。
  - Dogechain、Kava 等依赖 Multichain 做原生桥的链近乎丧失美元稳定币流动性。
  - 所有持有 anyToken 资产的 DeFi 协议（包括 Curve、SushiSwap）不得不下架 anyUSDC/anyUSDT 池。
- **资金去向**：
  - 转出地址集中在 6 个新建 EOA，资金被分拆转入若干中心化交易所（被 OKX、Binance 部分冻结约 $6M）。
  - 其余资金沉默，截至 2026-04 仍未流入主流混币器，疑为仍被控制状态。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**密钥管理（Key-Mgmt）/ 治理中心化**——非合约 bug，而是托管架构声称去中心化但实际单点控制。

### 3.2 受损模块

- Multichain SMPC 节点软件（未完全开源，部分代码见 `https://github.com/anyswap/FastMulThreshold-DSA`）。
- 链上锁仓合约（例：Fantom 端 `anyUSDC` MPC-owner `0x27564B8CaAb4CbEf760b0B11267E31EdD33B9E36`），owner 为 MPC 生成的 ECDSA 地址。

### 3.3 架构"号称 vs 实际"

```text
# 号称（Multichain 白皮书 / 官网 2022 版）
- 私钥以 t-of-n 门限切片分散在独立节点运营商
- 任何单一节点损毁不影响资产
- 节点运营商分布在多个司法辖区

# 实际（官方 2023-07-14 声明 + Ankr / Binance 后续披露）
- 所有 MPC 节点部署在 CEO 赵均个人名下阿里云账号
- 运维密码、SSH 私钥、MPC share 备份均托管于同一云账户
- "独立节点" 实为同一人管理的多个进程
- CEO 被拘 = 私钥事实上被一并扣押
```

由于没有真正的门限 + 地理分散，任何掌握 CEO 云账号凭据的人都可以重组私钥并发起任意跨链交易。这一事件成为"MPC 伪去中心化"最高代价的教训。

### 3.4 攻击步骤分解

因并无合约漏洞，所谓"攻击"是一系列**有效签名的正常提现交易**：

1. 某方（官方口径："unknown party"）在 2023-07-06 访问了 CEO 控制的 MPC 环境。
2. 在 Fantom 端用合法的 MPC 签名调用 `anyUSDC.Swapout` / 直接 owner 转账，将底层 USDC 从桥合约转至 EOA `0x9d57...`。
3. 同一签名权限被用于 Moonriver、Dogechain 等链的桥合约。
4. 部分资金 5 分钟内跨回 Ethereum，经多层 hops 沉淀。

链上无"异常"——所有交易都是合约期望的 owner 操作，因此监控系统（BlockSec、Forta）只能在**资金流出模式**维度告警，无法阻止签名。

### 3.5 为何审计未发现

- 合约审计能证明"合约按 owner 签名行事"；不能证明"owner 私钥去中心化"。
- MPC 运营属于 operational security 范畴，未包含在 PeckShield / SlowMist 等对 Multichain 合约的审计 scope 内。
- 多次社区质疑"节点运营商身份"均被以"商业机密"为由拒答。

## 4. 事后响应

- 2023-07-14：Multichain 官方宣布永久关停所有服务，团队已解散。
- Binance / OKX 根据链上追踪冻结约 $6M。
- Fantom Foundation 为用户设立应急赔付通道，但未能覆盖全部损失。
- 中国警方未公布具体案情；资金归属至 2026-04 仍在诉讼中。
- 行业后果：Stargate、LayerZero、Wormhole 等桥协议竞相发布"MPC 节点身份与证明"材料以挽回信任。

## 5. 启发与教训

- **对用户**：跨链桥的安全边界等于其**最脆弱的私钥托管者**。TVL 大 ≠ 安全。
- **对桥协议**：必须公开节点运营商身份、地域、审计证明；设置链上 timelock + 多签 + 社会恢复。
- **对行业**：MPC ≠ 去中心化。需要"可证明门限"方案（ZK-based proof of key distribution）或改用完全链上的 light-client 桥（如 Succinct / Polyhedra ZK bridge）。
- **对监管**：CEX/DeFi 共同暴露"单人密钥被外部实体接管"的系统性风险，"人身风险即协议风险"。

## 6. 参考资料

- SlowMist 分析: <https://slowmist.medium.com/multichain-exploit-analysis-c6c5b6b9b4a0>
- rekt.news: <https://rekt.news/multichain-rekt/>
- CertiK: <https://www.certik.com/resources/blog/3KJQUJuFQK3VApQjMY6eIb-multichain-exit-scam-incident-analysis>
- Elliptic: <https://www.elliptic.co/blog/multichain-suspected-125-million-hack>
- Multichain 官方 Twitter 声明: <https://twitter.com/MultichainOrg/status/1679768452084981760>
- Fantom Foundation 回应: <https://fantom.foundation/blog/multichain-update/>

---

*Last verified: 2026-04-23*
