---
title: Drift Protocol（2026-04-01，~$285M，Solana）
module: 08-security-incidents
category: Supply-Chain | Social-Eng | Key-Mgmt
date: 2026-04-01
loss_usd: 285260000
chain: Solana
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/drift-protocol-rekt
  - https://hacked.slowmist.io
  - https://rekt.news/leaderboard/
tx_hashes:
  - 2HvMSgDEfKhNryYZKhjowrBY55rUx5MWtcWkG9hqxZCFBaTiahPwfynP1dxBSRk9s5UTVc8LFeS4Btvkm9pc2C4H (Solana — admin transfer 创建)
  - 5V72ZK1WejP5Mh3uryEy6BZCV6ukSAnZBFSvHTqfD4NS38xKJUuh4RV5F8D4tDbgMsB2dcTJyZf7hLxH34nCRHRE (Solana — CVT deposit)
---

# Drift Protocol（2026-04-01，~$285M，Solana）

> **TL;DR**：2026-04-01 下午，Solana 头部 Perp DEX **Drift Protocol** 遭受 **~$2.85 亿美元** 洗劫。根因**不是合约漏洞**，而是长达 6 个月的 DPRK 社工渗透：攻击者以「合法做市商」身份建立关系 → 通过代码仓库与 TestFlight 应用投递恶意载荷 → 入侵贡献者的签名设备 → 抢夺 admin key。最终通过 Solana **durable nonce** 预签事务在半小时内清空 JLP Delta Neutral / SOL Super Staking / BTC Super Staking 等 10+ vault；资金经 Jupiter → USDC/WSOL/WETH → Circle CCTP 跨到 Ethereum，在 100+ 笔交易中分散到 4 个 EOA。归因中高置信度指向 **UNC4736 / AppleJeus（Lazarus Group 子团）**，与 2024-10 Radiant Capital 事件存在钱包簇与 TTP 重叠。

## 1. 事件背景

### 1.1 Drift Protocol 是什么

[Drift](https://drift.trade) 是部署在 Solana 上的去中心化永续合约交易所，2021 年上线（V1），2022-11 重大版本 **Drift V2** 发布后成为 Solana Perp DEX TVL/交易量双第一。核心产品：

- **JLP Delta Neutral Vault**：对冲 Jupiter LP 敞口的 delta-neutral 策略金库。
- **SOL Super Staking / BTC Super Staking Vault**：以 LST 抵押放大收益的策略库。
- 其他 Strategy Vault（USDC/USDT/cbBTC 等 10+ 代币池）。

2026-Q1 Drift TVL 常年稳定在 $500M+，事件发生时正值 Solana 生态 RWA / Perp 叙事回暖。

### 1.2 攻击时间轴

| 日期 | 阶段 | 事件 |
|------|------|------|
| 2025-10 前后 | 侦察 / 社工 | 攻击者以"某合法做市/量化交易公司"身份，开始与 Drift 核心贡献者建立长期联系（LinkedIn + Telegram + email） |
| 2025-11 至 2026-03 | 载荷投递 | 通过共享"合作项目代码仓库" / "TestFlight 版专属工具"投递 macOS 恶意载荷；逐步入侵贡献者的开发机与签名环境 |
| 2026-03 | 权限收集 | 取得 Drift admin 多签其中一个 signer 的硬件签名通道与会话 |
| 2026-04-01 下午 (UTC) | 执行 | 利用 Solana **durable transaction nonce**（事务 TTL 可绕过 150 slot 限制）预签一组"管理员转账"交易 |
| 2026-04-01 下午 ~30 分钟内 | 资产清空 | 一组交易从 10+ vault 转出 JLP/USDC/cbBTC/USDT/USDS/WETH/dSOL/WBTC/Fartcoin/JitoSOL 等 |
| 2026-04-01 傍晚 (UTC) | 洗钱 | Jupiter swap → USDC/WSOL/WETH → Circle CCTP 跨到 Ethereum（6+ 小时，100+ 笔），合并到 4 个 EOA |
| 2026-04-02 | 首次披露 | Drift 团队推特公告暂停提款；PeckShield / SlowMist 发首批链上分析 |
| 2026-04-03 至 04-10 | 归因 | ZachXBT + Chainalysis + Mandiant 交叉比对：钱包簇 & TTP 与 2024-10 Radiant Capital 重叠 → UNC4736 / AppleJeus |

### 1.3 发现过程

监控告警由 **Cyvers + PeckShield** 双方在 2026-04-01 下午几乎同时触发——异常大额 `transfer_authority` 调用 + 单一地址在 <10 分钟内从 10+ vault 提取 → 标红 → 推特公告。Drift 团队在 <1 小时内通过治理暂停了余下 vault 的提款路径，避免损失进一步扩大。

## 2. 事件影响

### 2.1 直接损失（按事发当日 USD）

| 资产 | 数量 | USD 估值 |
|------|------|---------|
| JLP | 42.72M | ~$159.35M |
| USDC | 71.42M | $71.42M |
| cbBTC | — | ~$11.29M |
| USDT | 5.65M | $5.65M |
| USDS | 5.25M | $5.25M |
| WETH | — | ~$4.69M |
| dSOL | — | ~$4.47M |
| WBTC | — | ~$4.36M |
| Fartcoin | — | ~$4.11M |
| JitoSOL | — | ~$3.60M |
| 其他 8 个代币 | — | 余量 |
| **合计** | | **~$285.26M** |

### 2.2 受害方

- **JLP Delta Neutral Vault 存款人**（承担最大份额损失，约 $159M）
- **Super Staking Vault 存款人**（SOL / BTC）
- **平台自营金库**（部分 USDC/USDT 池）
- **Drift 治理代币 DRIFT** 24h 内 -62%

### 2.3 资金流向

```
[Solana: Vaults] 
   ↓ transfer_authority (admin sig + durable nonce)
[Attacker Solana: 55udxh...SM7ZCW]
   ↓ Jupiter swap
[USDC / WSOL / WETH on Solana]
   ↓ Circle CCTP (100+ txs, 6+ hours)
[Ethereum: 0xd91a...b7dd (Chainflip dst)]
   ↓ 合并
[4 × Ethereum EOA]
   - 0xD3FEEd...1cF6C7
   - 0xAa843e...5e57C1
   - 0xbDdAE9...40561B
   - 0x0FE3b6...F6B674
   ↓ 部分资金回溯至 Tornado Cash 起点（10 ETH via LiFi/NEAR）
```

截至 2026-04-23，**未有追回**；部分资金仍在 Ethereum EOA 中静默持有，符合 Lazarus 一贯的 6–24 个月"冷藏 → OTC 出金"模式。

## 3. 技术根因

### 3.1 关键不是合约，是**密钥与签名链路**

Drift V2 合约本身未被发现新漏洞；代码已通过多轮审计（OtterSec、Neodyme）。攻击面在**链下签名环境**：

1. **社工入口**：长达 6 个月的"合法合作伙伴"伪装 → 建立信任 → 共享代码仓库 → 贡献者运行含恶意载荷的项目（含 pre-install script）。
2. **TestFlight 绕 Gatekeeper**：iOS TestFlight 签名应用在 macOS 端通过 Catalyst 运行时获得部分系统权限；被用作二级载荷下发。
3. **签名设备劫持**：贡献者的 Ledger / 硬件钱包在"confirmed prompt"环节被中间人会话劫持，攻击者获得有效签名输出。
4. **durable nonce 利用**：Solana 标准事务 blockhash 150 slot (~60s) 过期；但 **[durable transaction nonce](https://solana.com/docs/core/transactions#durable-nonce)** 允许事务**长时间待执行**。攻击者预签一组 admin 转账事务，在所有签名收集完后一次性广播，受害方无法在链上通过"撤销会话"阻止。

### 3.2 Admin 权限模型问题

Drift 的 `transfer_authority` 权限设计本意是让 DAO 升级金库 manager，但：

- 该权限**没有 timelock**，签名有效即刻执行。
- 多签阈值（推测 2/3 或 3/5）相对较低，一个 signer 被劫持 + 社工诱导另一人签名 → 快速凑齐。
- `transfer_authority` 可将金库所有资产"打包转移"，粒度过粗（没有按资产/额度细分）。

（**注**：具体多签阈值与 program 地址待 Drift 官方 post-mortem 披露后更新；本条目在那之前保持 DRAFT 状态。）

### 3.3 为何审计未发现

审计覆盖的是 **on-chain program 代码**；链下 key 管理 / 签名流程 / 开发环境安全 **不在传统智能合约审计范围内**。这与 Radiant Capital 2024-10（$58M）、Bybit 2025-02（$1.46B）同构——**"合约没错，但签名给错了"** 是 2024 以来头部事故的共同模板。

## 4. 事后响应

### 4.1 项目方

- 2026-04-01 傍晚：推特公告暂停 vault 提款，启动 incident response。
- 2026-04-02 凌晨：发布初步 post-mortem，承认 admin 签名被外部获取；承诺全额赔付用户。
- 2026-04-05：与 SEAL 911 / Chainalysis / zeroShadow 合作；同步向 CEX 发布冻结通知。
- 2026-04-10：披露补偿方案（DRIFT 新 mint + 协议未来费用收入分成，分阶段兑付）。

### 4.2 执法 / 归因

- **2026-04-08** FBI IC3 PSA：中高置信度归因 **UNC4736 / AppleJeus（Lazarus 子团）**。
- **ZachXBT** 调查线：攻击者入金 10 ETH via Tornado → LiFi → NEAR 桥，与 Radiant Capital 2024-10 攻击钱包簇上游交汇。
- **OFAC**：将 4 个 Ethereum EOA 加入 SDN 临时观察名单。

### 4.3 行业连锁

- Solana 上其他使用 `transfer_authority` 的金库协议（Kamino、MarginFi、Save）立即披露自家权限模型并加 timelock。
- Ledger、Safe{Wallet} 重发 Blind Signing 警示。
- Drift 启动 **全量签名改造**：引入独立 HSM + Fireblocks MPC + 72h admin action timelock。

## 5. 启发与教训

### 5.1 对开发者

- **链下签名环境 = 链上安全的一部分**。核心贡献者的开发机、签名设备、Yubikey / Ledger 接入流程必须纳入威胁模型。
- Admin 路径**必须强制 timelock**（即使是 12–24 小时，也能挽回 80% 社工事件）。
- `transfer_authority` 类"粗粒度"权限应拆分为细粒度、每资产/每额度授权。

### 5.2 对审计方

- 扩展审计范围：不仅审 program，还要审**签名流程图**、multisig 阈值、timelock 策略、开发机隔离策略。
- 引入 **红队演练**（模拟 social engineering），而不只是代码静态扫描。

### 5.3 对用户

- 选择 vault 时检查：是否有 timelock？admin 权限是否可批量挪资？key management 是否用 MPC？
- 在大额 DeFi 存款前关注 **协议的"管理员可做什么"清单**，而不只是合约审计标签。

### 5.4 对协议

- **Solana durable nonce** 是威胁模型中经常被忽视的"定时炸弹"——任何 admin 事务都应检查 `recent_blockhash` 路径或使用 expiration guard。
- 引入 **on-chain anomaly monitor**（如 Gauntlet / Chaos Labs）自动暂停异常大额转出。

## 6. 参考资料

- [rekt.news Drift Protocol Rekt](https://rekt.news/drift-protocol-rekt) — 深度复盘
- [SlowMist Hacked Database](https://hacked.slowmist.io) — 链上 tx 归档
- [Drift Protocol 官方](https://drift.trade) — 事后 post-mortem（2026-04-02/05/10 多版）
- [Chainalysis Blog](https://www.chainalysis.com/blog/) — 归因与资金流分析
- [FBI IC3 PSA](https://www.ic3.gov/Media/News)（2026-04-08）— Lazarus 子团归因
- [Mandiant / Google TAG UNC4736 追踪](https://cloud.google.com/blog/topics/threat-intelligence) — TTP 与 Radiant 重叠

---

**本条目为 DRAFT**：金额与 tx hash 经 rekt.news + SlowMist 双源核验；归因表述遵循 FBI / Mandiant 正式用词（"中高置信度"）。Drift 官方完整 post-mortem 发布后需重审多签阈值、program 地址与补偿方案最终版本。

*Last verified: 2026-04-23*
