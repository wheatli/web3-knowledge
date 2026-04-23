---
title: Harmony Horizon Bridge 被盗事件（2022-06-24, ~$100M）
module: 08-security-incidents
category: Bridge
date: 2022-06-24
loss_usd: 100000000
chain: [Harmony, Ethereum, BSC]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://medium.com/harmony-one/harmonys-horizon-bridge-hack-1e8d283b6d66
  - https://www.elliptic.co/blog/harmony-horizon-bridge-heist-attributed-to-lazarus-group
  - https://rekt.news/harmony-rekt/
  - https://twitter.com/harmonyprotocol/status/1540110924400861186
  - https://www.fbi.gov/news/press-releases/fbi-identifies-lazarus-group-cyber-actors-as-responsible-for-theft-of-100-million
tx_hashes:
  - 0x4a9a79cac8c76d8d04ea8c0db6f0c9b8c7d79c5b（Ethereum 攻击者一次出账 tx，示例；实际包含多笔，见 Etherscan 攻击地址）
  - 攻击者 Ethereum 地址: 0x0d043128146654c7683fbf30ac98d7b2285ded00
  - Horizon MultiSigWallet（Ethereum）: 0xf9Fb1c508Ff49F78b60d3A96dea99Fa5D7F3A8A6
---

# Harmony Horizon Bridge 被盗事件

> **TL;DR**：2022-06-24，Harmony 团队运营的跨链桥 Horizon Bridge 被盗约 **$100M**（ETH、USDC、BUSD、WBTC、BNB 等）。桥的 Ethereum 侧 MultiSig 阈值仅为 **2/5**，且 2 把关键私钥都由 Harmony 核心团队成员保管在热钱包 / 云端。攻击者通过针对开发者的**钓鱼（phishing）**与社工手段获取两把签名密钥后，直接发起两笔 MultiSig transaction 把桥清空。FBI 2023-01-23 正式归因 **Lazarus Group**。

## 1. 事件背景

### 1.1 Harmony 与 Horizon

Harmony（ONE）是一条 EVM 兼容 L1，2019 年主网上线。Horizon Bridge 是 Harmony 官方桥，连接 Ethereum、BSC、Harmony，支持 ETH / USDC / DAI / BUSD / WBTC / LINK / FXS / AAG 等主流资产。桥采用 **N-of-M 多签模型**，但 Ethereum 侧阈值为 2/5、BSC 侧为 2/4——属于行业公认过低配置。

截至 2022-06，Horizon Bridge 托管资产超 $100M，是 Harmony 生态 DeFi 的主干流动性路径（如 Tranquil Finance、DeFi Kingdoms）。

### 1.2 时间轴

- **2022-06-23 UTC 晚**：攻击者以钓鱼方式获取 Harmony 团队 2 名工程师的多签私钥（官方后续披露：密钥存储于加密 keyvault，攻击者在本地解密）。
- **2022-06-24 UTC 07:00–08:30**：攻击者在 Ethereum 上发起 11 笔 `MultiSigWallet.submitTransaction → confirmTransaction → executeTransaction`，耗尽桥内所有支持 token。
- **2022-06-24 UTC 12:48**：Harmony 官方 Twitter 确认 Horizon Bridge 被盗约 $100M，暂停桥。
- **2022-07-01**：Harmony 发起白帽协商，10% 赏金未被接受。
- **2022-06-27 ~ 2022-06-29**：攻击者开始通过 Tornado Cash 洗币，分批 100 ETH。
- **2023-01-23**：FBI 正式归因 Lazarus Group。

### 1.3 发现过程

Ethereum 上 `0xf9Fb1c5...` MultiSigWallet 的异常 execute 被 Etherscan 追踪者、@zachxbt、Chainalysis 几乎同时标注；Harmony 团队也在内部监控触发告警。

## 2. 事件影响

### 2.1 直接损失

- **13,100 ETH** (~$26M)、
- **$41M BUSD**、**$36.7M USDC**、**$3.3M USDT**、
- **$4M AAG**、**$3M FXS**、**$2.2M SUSHI**、**$1.9M WBTC**、**$1.9M DAI**、**$1.8M FRAX** 等；
- 合计约 **$100M**（按 2022-06-24 价格）。

### 2.2 受害方

- Harmony 生态所有跨链用户；
- Tranquil Finance（在 Horizon 侧）、DeFi Kingdoms、OpenSwap 等协议内的 bridged 资产持有人；
- Harmony 官方承担赔付，发行债券类方案，但因市场低迷执行受阻。

### 2.3 连带影响

- ONE 币价 24h 内下跌 ~10%；
- Harmony 2022-07 提出 `HIP-29` 通胀 50 亿 ONE 补偿用户，引起社区分裂最终被否；
- 行业对 "2/N 小多签 + 热钱包密钥" 跨链桥模型彻底抛弃。

### 2.4 资金去向

- 攻击者 Ethereum 主地址：`0x0d043128146654c7683fbf30ac98d7b2285ded00`；
- 2022-06-27 起分批 100 ETH 进入 Tornado Cash，6 天内洗完约 65,000 ETH 等值资产；
- 2023-01 Elliptic 追踪发现与 Ronin Bridge 攻击地址聚类重合，强化 Lazarus 归因；
- FBI 2023-01-23 披露攻击者把部分资金经 Railgun、Sinbad.io 进入 OTC；执法部门冻结部分稳定币。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**密钥管理 + 阈值过低**（不是合约漏洞）。

### 3.2 受损模块

- `MultiSigWallet.sol`（基于 Gnosis MultiSig v1.1.1 改造）
- Ethereum 合约：`0xf9Fb1c508Ff49F78b60d3A96dea99Fa5D7F3A8A6`
- 阈值：`required = 2`，`owners = 5`

### 3.3 关键代码片段（示意）

```solidity
// 基于 Gnosis MultiSigWallet（Horizon 版本 required=2, owners=5）
function submitTransaction(address destination, uint value, bytes data) public returns (uint transactionId);
function confirmTransaction(uint transactionId) public;
function executeTransaction(uint transactionId) public {
    if (isConfirmed(transactionId)) {           // 检查 confirmation 数 >= required (=2)
        // ...
        destination.call.value(value)(data);    // 直接 call 到桥内 token 出口函数
    }
}
```

**错在哪**：
- 阈值 2/5 过低，攻击者只需 2 个有效签名即可解锁全部桥资产；
- 5 个签名者中 2 个密钥由同一小团队成员保管且使用常见云端 keyvault，**单点破解可影响 2 签**；
- 没有 time-lock、没有单笔提款上限、没有地址白名单、没有冷静期；
- 紧急情况下无法一键 freeze（桥管理员并非多签，恶意管理员也无法快速暂停）。

### 3.4 攻击步骤分解

1. **钓鱼 / 社工**（至少数周到数月）：攻击者用 Lazarus 惯用的 LinkedIn / Telegram 伪装成求职者或合作方，向 Harmony 工程师投递恶意文件；或通过 npm 供应链污染获取开发者机器权限。
2. **密钥提取**：从被攻陷机器的加密密钥库中解密出 2 个 validator 签名密钥。
3. **准备提款交易**：在 MultiSigWallet `submitTransaction` 提交 11 笔大额跨链提款交易（每笔对应不同 token）。
4. **两签执行**：以 2 个密钥分别 `confirmTransaction`，触发 `executeTransaction`，把桥内 token 送到攻击者地址 `0x0d04...`。
5. **桥空**：1.5 小时内清空 ~$100M。
6. **洗币**：通过 Tornado Cash 分批 100 ETH。

### 3.5 为何审计未发现

- Horizon Bridge 合约层面使用业界成熟 Gnosis MultiSig，审计集中在代码；
- 阈值参数（2/5）是**部署参数**，审计通常不质疑部署方治理选择；
- **密钥管理**不属于合约审计范围。

## 4. 事后响应

### 4.1 项目方动作

- 暂停 Horizon Bridge；
- 2022-06-26：悬赏 $1M，后加到 $10M；
- 2022-07：提出 HIP-29（增发 ONE 补偿），社区激烈反对后放弃；
- 桥重启采用新架构（LayerZero + Hashi 验证），放弃自运营多签。

### 4.2 资产追回

链上追回约 **$0.3M**（Tether / Circle 冻结部分 USDC / USDT）。主体 ~$100M 追回失败。

### 4.3 法律 / 执法

- **2023-01-23**：FBI 发布 public statement，把 Horizon Bridge 盗案归因于 Lazarus Group 的子组"APT38 / TraderTraitor"；
- Elliptic、Chainalysis 发布地址聚类报告，与 Ronin、Atomic Wallet 等案件相关联。

### 4.4 复查与审计

- 合约本身无需重审，团队转向使用 LayerZero 的 messaging 层 + 第三方 DVN；
- Harmony 此后将密钥托管迁至 HSM + MPC（Fireblocks / Safe modular）。

## 5. 启发与教训

### 5.1 对开发者

- 跨链桥多签阈值必须 **≥ 2/3** 且 M ≥ 7；
- 关键私钥必须使用 **HSM 或 MPC**，严禁热钱包 / 云 keyvault 存明文加密副本；
- 关键动作必须有 timelock 与额度限制。

### 5.2 对审计方

- 审计报告应明确标注 "部署参数（threshold、owners、timelock）为风险范畴但不在审计"，并强烈建议值；
- 引入 operational audit 范畴，检查密钥生成、保管、轮换流程。

### 5.3 对用户

- 2/5 多签 = 信任 2 个人，这是风险识别信号；
- 跨链资产分散在多个桥；观察桥的 MultiSig 阈值可在 Etherscan 上直接读出。

### 5.4 对协议

- 全员安全培训（针对 LinkedIn / Telegram 钓鱼）必须季度化；
- 引入 dedicated incident response playbook（冻结、熔断、监控）。

## 6. 参考资料

- Harmony 官方 post-mortem：https://medium.com/harmony-one/harmonys-horizon-bridge-hack-1e8d283b6d66
- Elliptic Lazarus 归因：https://www.elliptic.co/blog/harmony-horizon-bridge-heist-attributed-to-lazarus-group
- FBI 归因声明：https://www.fbi.gov/news/press-releases/fbi-identifies-lazarus-group-cyber-actors-as-responsible-for-theft-of-100-million
- rekt.news：https://rekt.news/harmony-rekt/
- 多签合约 Ethereum：`0xf9Fb1c508Ff49F78b60d3A96dea99Fa5D7F3A8A6`
- 攻击者地址：`0x0d043128146654c7683fbf30ac98d7b2285ded00`
- Harmony 官方推文：https://twitter.com/harmonyprotocol/status/1540110924400861186

---

*Last verified: 2026-04-23*
