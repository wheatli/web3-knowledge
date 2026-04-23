---
title: Kelp DAO / rsETH Bridge（2026-04-18，~$292M，Multi-chain via LayerZero）
module: 08-security-incidents
category: Bridge | Supply-Chain | Key-Mgmt
date: 2026-04-18
loss_usd: 292000000
chain: Ethereum + LayerZero (multi-chain)
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://hacked.slowmist.io
  - https://rekt.news/leaderboard/
  - https://layerzero.network (官方声明，待归档)
  - https://kelpdao.xyz (官方 post-mortem，待归档)
tx_hashes:
  - 待 Kelp DAO 官方 post-mortem 披露
---

# Kelp DAO / rsETH Bridge（2026-04-18，~$292M）

> **TL;DR**：2026-04-18，以太坊头部 LRT（Liquid Restaking Token）协议 **Kelp DAO** 的 rsETH 跨链桥遭 **~$2.92 亿美元** 洗劫。根因**不是 LayerZero 协议本身**，而是 Kelp DAO 配置的 **单一 DVN（Decentralized Verifier Network）** 下游 **RPC 基础设施** 被攻破——攻击者先入侵两个 RPC 节点替换二进制，再对剩余未失陷节点发起 **DDoS** 迫使 failover 到"被投毒"节点；随后通过 forged payload 铸造巨量 rsETH。LayerZero 首次公开声明此事件揭示**"单 DVN"设置的系统性风险**。归因为**高置信度 Lazarus Group / TraderTraitor 子团**。

## 1. 事件背景

### 1.1 Kelp DAO 与 rsETH

[Kelp DAO](https://kelpdao.xyz) 是 [EigenLayer](https://www.eigenlayer.xyz) 再质押生态的头部 LRT 协议之一，发行 **rsETH** 作为"流动再质押凭证"。用户存入 ETH/stETH → 协议代为 EigenLayer restake → 铸 rsETH。2026-Q1 Kelp DAO TVL ~$1.8B，rsETH 流通于 Ethereum、Arbitrum、Optimism、Base、Linea、Scroll 等 7+ 链——**跨链由 LayerZero OFT 标准承载**。

### 1.2 LayerZero DVN 模型简介

[LayerZero V2](https://docs.layerzero.network) 把消息验证责任交给 **DVN（Decentralized Verifier Network）**。项目方可以自选 DVN 组合：
- **Required DVN set**：必须全部签名
- **Optional DVN set + threshold**：至少 K/N 签名

每个 DVN 是一个独立运行的验证者服务，通过 RPC 节点读取源链状态并签名消息。**Kelp DAO 仅配置了 1 个 required DVN，且该 DVN 只连接少数几个 RPC 节点**——这是本次事件的结构性漏洞。

### 1.3 攻击时间轴

| 日期/时间 (UTC) | 事件 |
|------|------|
| 2026-04-18 之前数周 | 攻击者侦察 Kelp DVN 的 RPC 基础设施，选定 2 个节点为渗透目标 |
| 2026-04-18 之前数日 | 通过开发者投递恶意更新包 / 二进制替换成功入侵 2 个 RPC 节点；植入"对特定 payload 返回伪造状态"的逻辑 |
| 2026-04-18 ~早间 | 对 Kelp DVN 连接的其余 RPC 节点发起大规模 DDoS |
| 2026-04-18 | DVN 因其余节点不可达 → failover 到"被投毒"节点 → 接受伪造状态 → 对攻击者构造的 forged message 签名 |
| 2026-04-18 | 攻击者在目标链上提交 forged payload → 合约接受 → 铸造巨量 rsETH（及对应桥接资产）|
| 2026-04-18 ~中段 | 攻击者立即 swap rsETH → ETH / USDT / USDC，分散多链，部分过 THORChain/Chainflip 换 BTC |
| 2026-04-18 下午 | Kelp DAO 团队发现异常 → 暂停跨链合约 → LayerZero 发公告 |
| 2026-04-19 | LayerZero 正式声明："此为客户侧 DVN/RPC 配置问题，非协议层漏洞" |
| 2026-04-20 | FBI / TRM Labs 初步归因 Lazarus（TraderTraitor） |

### 1.4 发现过程

多方同时告警：**Cyvers** 链上 ML 风险信号 + **PeckShield** 异常大额铸造检测 + Kelp DAO 内部 Treasury 监控。LayerZero 接到告警后 <30 分钟确认是 DVN 路径异常而非消息传递协议本身。

## 2. 事件影响

### 2.1 直接损失

- **~$292M**（按事发当日 USD 估值）
- 主要为 rsETH 无中生有铸造，然后 dump 进二级流动性池 → **拖累 rsETH 二级价格脱锚 ~18%**（最深跌至 0.82 ETH），殃及所有 rsETH 持有人
- Kelp DAO Treasury 部分 ETH 储备被动用于维持 redemption

### 2.2 连带影响

- **LRT 赛道全体 -8~12%**：EtherFi (weETH)、Renzo (ezETH)、Puffer (pufETH) 跟跌
- **LayerZero 代币 ZRO** 24h -14%，尽管官方声明"非协议层漏洞"
- **DVN 设置审查浪潮**：大量使用 LayerZero 的项目（Stargate、Radiant、Pendle 等）紧急公示 DVN 数量与 RPC 配置
- EigenLayer 生态整体 TVL 一周内蒸发 ~$1.2B（赎回潮）

### 2.3 资金流向

（截至 2026-04-23，链上追踪进行中，完整 hop 图待 Chainalysis 报告）

```
[Forged rsETH Mint on target chain]
   ↓ swap on DEX aggregators (1inch / OKX DEX)
[ETH / USDT / USDC]
   ↓ THORChain / Chainflip
[BTC]
   ↓ Mixer / OTC（Lazarus 典型链路）
```

**尚无资金追回**；部分 EOA 已被 CEX 端冻结约 $6M（Binance、OKX 协作）。

## 3. 技术根因

### 3.1 核心：DVN 单点 + RPC 供应链

Kelp 的配置错误**不在 LayerZero 协议本身**，而在**客户侧的 DVN 与 RPC 部署**：

```
  [LayerZero Endpoint (A链)]
          ↓ emits packet
  [Kelp DVN Verifier Service]   ← 只有 1 个
          ↓ read via
  [RPC Node pool: N1, N2, N3 (public), N4, N5 (premium)]
                         ↑
                    攻击面：N4, N5 被投毒；N1-N3 被 DDoS
          ↓ failover
  [Compromised RPC returns forged state]
          ↓ DVN 签名 forged message
  [LayerZero Endpoint (B链)] → 合约接受 → 铸 rsETH
```

**三层失效叠加**：
1. **DVN 单点**（应配置 Required 2-of-3 或以上）
2. **RPC 无白名单 / 无二源校验**（应至少 2 个独立 provider 状态一致才进入签名流程）
3. **二进制完整性未校验**（RPC 节点被二进制替换后，DVN 未通过 hash / SGX attestation 检测）

### 3.2 为何 DDoS + RPC 投毒构成"有效的验签绕过"

Single-DVN 架构下，DVN 的"真相"完全取决于其可见的 RPC 状态。**RPC = 单一源头**。攻击者组合：
- **主动入侵**（投毒）：构造特定消息时返回虚假源链状态
- **DDoS**（排除）：让 DVN 只能看到被投毒节点

这让被投毒节点**成为唯一源头** → DVN 对伪造源链状态签名 → LayerZero 消息层原封不动投递到目标链 → 合约无法区分"真来自源链"和"DVN 说来自源链"。

### 3.3 LayerZero 的定位与声明

LayerZero 在官方声明中强调：
- **协议本身未被攻破**，消息路由 / 验证逻辑工作正常。
- 漏洞在"**客户选择的 DVN + RPC 配置**"，类同"HTTPS 协议没问题但用户把 CA 设为伪造"。
- 已向所有使用 LayerZero 的项目推送**必选配置检查清单**：≥2 DVN、≥3 独立 RPC provider、启用 SGX attestation、每 24h 失效自动 reset。

社区对 LayerZero 的"免责声明式"回应存在争议——批评者认为协议设计应**默认拒绝单 DVN 配置**（"insecure by default 是协议的锅，不是用户的锅"）。

## 4. 事后响应

### 4.1 项目方（Kelp DAO）

- **2026-04-18**：暂停跨链合约、rsETH mint / redeem 窗口冻结。
- **2026-04-19**：发布初步 post-mortem，承认 DVN 与 RPC 配置失误；承诺治理代币 + 协议未来收入回购补偿。
- **2026-04-22**：升级至 **LayerZero 3 DVN + 5 RPC provider** 架构；引入 [Chaos Labs](https://chaoslabs.xyz) 实时监控。

### 4.2 LayerZero

- 公告强调"协议未被攻破"；同时发布 **DVN 安全基线文档**并启动"低于基线的项目名单"预警机制。
- 推动 **SGX-based DVN** 生产化，计划 2026-Q3 强制上线。

### 4.3 执法 / 归因

- **2026-04-20** FBI IC3 PSA + **TRM Labs** 报告：**高置信度 Lazarus Group / TraderTraitor 子团**；与 **2025-02 Bybit $1.46B** / **2024-10 Radiant Capital** 钱包簇与 TTP（DDoS + 供应链投毒）重叠。
- OFAC 发布新一批 SDN 地址。

### 4.4 行业连锁

- 所有 LayerZero 项目在 72h 内披露 DVN 数量（超过 50% 使用 ≥2 DVN；余下限期整改）。
- 跨链消息协议行业（CCIP / Wormhole / Axelar / Across）集体发文解释各自的 RPC 路径与供应链安全模型。
- **SGX / TEE 在 DVN 与 Oracle 的使用率**短期内大幅提升。

## 5. 启发与教训

### 5.1 对开发者 / 协议方

- **跨链验证永远不能依赖单一源头**：单 DVN = 单点 = 必然失败。
- **RPC ≠ 可信源**：必须引入双/多 provider 状态比对、SGX attestation、binary integrity check。
- **Insecure by default 是设计责任**：协议应默认禁止"1 DVN + 无校验 RPC"这类不安全配置。
- LRT 协议尤其高风险：**铸造权**是无限资产授权，必须配合 rate-limit / timelock / circuit breaker。

### 5.2 对 LayerZero / 同类跨链协议

- 需承担更多"安全默认值"责任，不能仅以"协议完备"脱责。
- DVN 安全基线应**强制执行**而非推荐。
- 引入 **on-chain DVN health attestation**：消息到达目标链时，合约层能验证 DVN 来源的多样性与时效性。

### 5.3 对用户 / LP

- LRT 存款前检查：**协议用几个 DVN？几个 RPC？是否有 circuit breaker？**
- 大额铸造事件的实时监控工具（DeFiLlama mint alerts）应纳入日常巡检。

### 5.4 对审计方

- Kelp DAO 合约审计覆盖了 on-chain 逻辑，**但 DVN/RPC 配置属于运维层**，目前多数审计未覆盖——需行业扩展审计范围至"跨链配置 + 运维基础设施"。

## 6. 参考资料

- [rekt.news leaderboard](https://rekt.news/leaderboard/) — 金额 & 时间列表（Kelp DAO 深度复盘待补）
- [SlowMist Hacked Database](https://hacked.slowmist.io) — 链上证据与 DVN 投毒机制分析
- [LayerZero 官方博客 / Twitter](https://layerzero.network) — 2026-04-19 声明
- [Kelp DAO 官方 post-mortem](https://kelpdao.xyz) — 待 2026-04-22 完整版归档
- [FBI IC3 PSA](https://www.ic3.gov/Media/News)（2026-04-20）— Lazarus / TraderTraitor 归因
- [TRM Labs 报告](https://www.trmlabs.com/resources) — 资金流与归因分析
- [Chainalysis Blog](https://www.chainalysis.com/blog/) — 2026 LRT 攻击趋势

---

**本条目为 DRAFT**：tx hash 与攻击者地址待 Kelp DAO 完整 post-mortem + Chainalysis 报告披露后更新；LayerZero 声明与 Kelp 之间的责任划分仍在演化，后续需重审。

*Last verified: 2026-04-23*
