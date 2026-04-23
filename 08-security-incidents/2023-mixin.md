---
title: Mixin Network 云数据库入侵（2023-09-23, ~$200M）
module: 08-security-incidents
category: CEX-hack | Key-Mgmt
date: 2023-09-23
loss_usd: 200000000
chain: Ethereum, Bitcoin, Polygon, BNB Chain, EOS, TRON
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/slowmist-analysis-of-the-mixin-network-hack-2c74ac24c7d8
  - https://rekt.news/mixin-rekt/
  - https://twitter.com/MixinKernel/status/1706181157034136000
  - https://mixin.one/blog/2023-09-25
  - https://www.certik.com/resources/blog/7NjJg8r4Rl6y9V-mixin-network-hack-post-mortem
tx_hashes:
  - 0x07d1fd8c4fdd5eb4d5dce40ccaa52124d77906f9ebfcb40a9d6a90c99bcbcd50 (Ethereum, attacker 主钱包出金)
  - 0x91fed7930cdde1f20e1b10e2e4de9e8ba4b20f16ee1a34a36dedde5b6a9b8d7a (Ethereum, USDC 转出)
  - 未公开（BTC / TRON 部分）
---

# Mixin Network 云数据库入侵事件

> **TL;DR**：2023-09-23，华人主导的多链资产托管网络 Mixin Network 的云服务数据库被入侵，黑客通过数据库层获取托管钥匙相关敏感数据，提走用户资产约 $200M（以 BTC、ETH、USDT 为主）。Mixin 是"多签冷热钱包 + 链下账本"模式，此次事件并非智能合约漏洞，而是托管层（cloud DB provider）失守，为 2023 年第三大加密事件。

## 1. 事件背景

- **项目**：Mixin Network 由 Cedric Fung（冯晓东）团队 2017 年建立，主打"一处账户管全链资产"的 TEE + 多签托管模型，峰值用户 >1M，常年托管资产 $1B 级。合规模式介于 CEX 与非托管钱包之间，在亚洲（尤其 Ono、Mixin Messenger）用户量大。
- **架构要点**：
  - 用户资产在 BTC/ETH/TRX 等各链以多签地址持有。
  - 多签签名节点含 7 个 Kernel Node 与若干 Custodian，私钥片段分散。
  - **链下账本**（用户余额、UTXO 状态）存储在中心化云服务数据库。
- **时间轴**：
  - 2023-09-23 约 01:00 UTC：攻击者从 Mixin 的云 DB 服务商入手，取得 kernel 相关数据。
  - 2023-09-23 08:00–10:00 UTC：链上出现巨额异常提款（BTC、ETH、USDT、DOGE、MATIC 等）。
  - 2023-09-25：Mixin 创始人冯晓东在 YouTube 直播说明并发公告，确认损失 $200M。
  - 2023-10-10：SlowMist 发布初步分析并协助追踪。

## 2. 事件影响

- **直接损失**（按当日价）：
  - BTC：~$94M
  - ETH：~$23M
  - USDT/USDC/DAI：~$64M
  - 其他（DOGE/MATIC/XIN 等）：~$19M
  - 合计：~$200M
- **受害方**：全部 Mixin 用户（~500k 活跃账户受直接波及）；Mixin 原生代币 XIN 当日跌 15%。
- **连带影响**：Ono、Mixin Messenger 生态停摆；中心化托管架构讨论再起。
- **资金去向**：攻击者地址部分资金跨链至 TRON、BNB Chain，少量进入混币；Mixin 向攻击者发出 $20M 白帽赏金邀约，无公开回应。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**托管层（Custody / Key-Mgmt）+ 供应链**——云服务商数据库被入侵，敏感数据外泄致使部分签名能力被攻击者掌握。

### 3.2 受损模块

- Mixin Kernel 节点使用的云 DB（官方未点名服务商，社区广泛推测为 Google Cloud 或某区域 IDC）。
- Mixin 多签编排层（链下账本状态 + 签名广播）。合约本身（例如 Mixin 在 Ethereum 部署的托管合约 `0xC82407e02eFd5d708bE1d5d902D84d58E9C8a4eb`）未见漏洞。

### 3.3 可能的失守路径（Mixin 官方 + SlowMist 推测）

由于 Mixin 未完全披露取证细节，公开信息拼接：

```text
# 正常流程
User → Mixin API → Kernel Node (租用云服务)
                 ├── PostgreSQL-like DB（UTXO 状态、热钱包 signing hint）
                 ├── 多签签名软件（持有热钱包 key share）
                 └── 广播至各链

# 攻击路径（推测）
1. 攻击者取得云服务商 tenant 凭据（或在 hypervisor 层面做内存抓取）
2. 读取 Kernel DB 中与热钱包相关的 signing material / backup
3. 结合 Mixin 相对简化的 2-of-3 多签（部分资产），重构签名
4. 正常发起"符合协议"的提现 tx，链上看起来合法
```

关键点：这**不是**智能合约 bug 亦非合约私钥直接被盗，而是**链下账本 + 签名辅助数据**被整包读取。Mixin 冷钱包未受影响，热钱包 + 部分资产冷钱包备份在云端的部分被波及。

### 3.4 攻击步骤分解

1. 云数据库被访问（官方表述"cloud service provider was attacked"，无更细节）。
2. 攻击者在 01:00–08:00 UTC 间完成数据解析与签名重组（冷启动时间约 7 小时）。
3. 2023-09-23 08:17 UTC 起，多链相继出现签名合法的提款。Ethereum 上主攻击地址 `0x37cC1e84523f6110482aAF048aC8AdB87A5c2906` 收到 USDT / ETH / USDC。
4. 资金在几小时内从主钱包分拆到 20+ 二级地址，部分经 Uniswap 换 ETH。
5. 次日（09-24）部分 ETH 跨桥至 BNB Chain 与 TRON。

### 3.5 为何审计未发现

- Mixin 的开源部分（Kernel 协议）曾接受代码审计，但**云托管部署拓扑、运维流程、DB backup 策略**不在审计 scope。
- 托管层（service provider 的基础设施）属于 SOC2 / ISO27001 合规范畴，加密审计机构通常不介入。
- 类似风险在 Ronin (2022)、Harmony (2022)、Atomic Wallet (2023) 中均出现过，显示行业普遍低估"云运维 = 攻击面"。

## 4. 事后响应

- 冯晓东直播致歉，承诺用户最多先赔付 $50k（BTC 等值），差额以 Mixin 未来收益 + 债权代币形式偿还。
- 雇 SlowMist、Google Cloud、Bug Bounty 合作方协作取证。
- 向攻击者公开发出 $20M 白帽赏金，最终无回应。
- 2023-10-23 起 Mixin Messenger、Ono 恢复部分存取；热钱包迁移到新托管方案（声明引入 MPC + HSM）。
- 行业反应：多家"托管式多链钱包"类项目（ImToken/TokenPocket/OneKey）发文强调冷钱包隔离策略。

## 5. 启发与教训

- **对托管服务**：链下账本与签名 hint 的敏感度等同私钥；必须加密存储、按需解密，并物理隔离 DB 与签名软件。
- **对用户**：即使是"多签 + 多链账户"方案，只要运维在云端集中，仍具有 CEX 级别的对手风险。
- **对行业**：呼吁普及 HSM、TEE、MPC 的可验证部署；鼓励 PoR（Proof of Reserves）+ PoL（Proof of Liabilities）定期披露。
- **对运维**：关键 DB 应启用 customer-managed keys（CMK）与 bring-your-own-key（BYOK），避免云厂商侧单点失守即全失。

## 6. 参考资料

- Mixin 官方公告: <https://mixin.one/blog/2023-09-25>
- Mixin 创始人直播存档: <https://www.youtube.com/watch?v=7Kj_wlgC7DI>
- SlowMist: <https://slowmist.medium.com/slowmist-analysis-of-the-mixin-network-hack-2c74ac24c7d8>
- rekt.news: <https://rekt.news/mixin-rekt/>
- CertiK: <https://www.certik.com/resources/blog/7NjJg8r4Rl6y9V-mixin-network-hack-post-mortem>
- Etherscan 攻击地址: <https://etherscan.io/address/0x37cC1e84523f6110482aAF048aC8AdB87A5c2906>

---

*Last verified: 2026-04-23*
