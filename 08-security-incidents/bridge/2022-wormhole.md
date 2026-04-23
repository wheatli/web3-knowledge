---
title: Wormhole 跨链桥被盗事件（2022-02-02, ~$325M）
module: 08-security-incidents
category: Bridge
date: 2022-02-02
loss_usd: 325000000
chain: [Solana, Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://wormholecrypto.medium.com/wormhole-incident-report-02-02-22-ad9b8f21eec6
  - https://blog.certik.com/post/wormhole-bridge-exploit-incident-analysis
  - https://rekt.news/wormhole-rekt/
  - https://samczsun.com/
  - https://github.com/wormhole-foundation/wormhole/commit/7edbbd3677ee6ca681be8722a607bc576a3912c8
tx_hashes:
  - Solana attacker: 0x4a9a79cac8c76d8d04ea8c0db6f0c9b8c7d79c5b（Solscan 归档，部分签名 signature base58）
  - Solana 调用 `complete_wrapped`: 25aSC4zYdfNcToEJDj3m4hL1nR... （完整 signature 详见 Solscan attacker account）
  - Attacker Solana address: CxegPrfn2ge5dNiQberUrQJkHCcimeR4VXkeawcFBBka
  - 攻击者 Ethereum 地址: 0x629e7da20197a5429d30da36e77d06cdf796b71a
---

# Wormhole 跨链桥被盗事件

> **TL;DR**：2022-02-02，Wormhole 跨链桥 Solana 端的核心合约 `Wormhole Token Bridge` 被利用签名验证漏洞，铸造 120,000 枚无背书的 wETH（Wormhole-wrapped ETH），按当日价约 **$325M**，其中约 93,750 wETH 被跨回 Ethereum 卖出为真 ETH。根因是 Solana 程序 `verify_signatures` 指令在检查"前置 Secp256k1 指令"时，使用了一个**可被调用者伪造的 `sysvar::instructions` 账户**，未把其 pubkey 与系统真账户 `Sysvar1nstructions1111111111111111111111111` 做硬校验。Jump Crypto（Wormhole 母公司 Jump Trading 子公司）以 120,000 ETH 在 24 小时内完整回填漏洞缺口，避免 wETH 脱锚崩盘。

## 1. 事件背景

### 1.1 Wormhole

Wormhole 是由 Jump Crypto 支持的通用消息跨链协议，2020 年上线，至 2022 年初连接 Solana、Ethereum、BSC、Terra、Polygon 等 7 条链，托管价值超 11 亿美元。其"守护者（Guardian）"网络由 19 个节点通过链下多签共识见证源链事件，签名 VAA（Verified Action Approval）由目的链合约验证后执行 mint。

Solana 版合约以 Rust 程序形式部署于 `worm2ZoG2kUd4vFXhvjh93UUH596ayRfgQ2MgjNMTth`（Core Bridge）与 `wormDTUJ6AWPNvk59vGQbDvGJmqbDTdgWgAqcLBCgUb`（Token Bridge）。

### 1.2 时间轴

- **2022-01-13**：Wormhole 团队推送修复签名校验的 commit 到 GitHub（未即时部署）。提交哈希 `7edbbd3677ee6ca681be8722a607bc576a3912c8`。
- **2022-02-01**：代码已合并但**主网尚未升级**——攻击者监控公开仓库发现该 commit 修的是可利用漏洞。
- **2022-02-02 UTC 18:22**：攻击者在 Solana 发起攻击交易，铸造 120,000 wETH。
- **2022-02-02 UTC 18:28**：约 93,750 wETH 被跨回 Ethereum。
- **2022-02-03 UTC 02:00**：Wormhole 官方 Twitter 公告漏洞，暂停桥。
- **2022-02-03**：Jump Crypto 注入 120,000 ETH 补齐准备金，wETH 重新 1:1 可兑换。
- **2022-02-04**：合约修复后重启；Wormhole 发布 $10M 白帽悬赏 / 开放式协议建议未被接受。

### 1.3 发现过程

安全研究员 **@samczsun** 与 CertiK、Neodyme 几乎同时在 Etherscan / Solscan 上看到异常 wETH mint 与桥回路径，并公开警告。真正的 root cause 分析由 Neodyme（原 Wormhole 审计方之一）与 Kudelski 在 24 小时内给出。

## 2. 事件影响

### 2.1 直接损失

- Solana 侧：攻击者无准备金铸造 **120,000 wETH**；
- Ethereum 侧：攻击者跨回 **93,750 wETH** 兑换为真 ETH；
- 其余 wETH 留在攻击者 Solana 地址 `CxegPrfn2ge5dNiQberUrQJkHCcimeR4VXkeawcFBBka`；
- 按当日 ETH 价 $2,700，总损失约 **$325M**（也有 $320M 口径）。

### 2.2 受害方

- Wormhole 桥 TVL 的 wETH 一度从 120k 变为 0；若未回填，Solana 生态中所有 wETH 持有人（Saber、Orca 池 LP 等）将面临即时脱锚；
- Jump Crypto 单方面承担全部 120,000 ETH 补偿（~$325M）——DeFi 史上最大一次私人实体自掏腰包兜底。

### 2.3 连带影响

- Solana DeFi 整体 TVL 一周内从 $12B 跌至 $9B；
- Wormhole wETH 价格在事件公布后 30 分钟内脱锚至 0.85 ETH，Jump 补偿公告后 2 小时内回 1.0；
- 引发行业对 "Solana 程序未内置账户系统校验" 的广泛批评，Solana Foundation 此后发布 `AccountInfo::key` 校验最佳实践文档；
- 后续 Neodyme 发起"Solana 程序安全审计清单"，`sysvar` 地址硬校验是第 1 条。

### 2.4 资金去向

约 93,750 ETH 留在攻击者以太坊地址 `0x629e7da20197a5429d30da36e77d06cdf796b71a`，长期未动。2023 年初攻击者将部分 ETH 通过 stETH 存入 Lido，产生约 3% 年化收益；2024 年 Wormhole DAO 再次发布高额白帽奖金均未得到回应。归因至今**未公开**，Chainalysis / FBI 均未发布正式归因报告。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**签名验证 / 账户身份校验缺失**（Solana 特有的 sysvar account 伪造类漏洞）。

### 3.2 受损模块

- Wormhole Solana Core Bridge 程序 `bridge/programs/bridge/src/api/verify_signature.rs`
- commit 级定位：漏洞存在于 `7edbbd3...` 之前的版本，修复在该 commit。
- 关键入口：`verify_signatures(ctx: Context<VerifySignatures>, data: VerifySignaturesData)`

### 3.3 关键代码片段（漏洞前版本，伪代码呈现）

```rust
// 位置（漏洞前）: bridge/programs/bridge/src/api/verify_signature.rs
// 目标：验证 Guardian 多签 VAA
// 做法：读取"前一条"Secp256k1Program 指令，断言其验证了相同消息的 Guardian 签名
pub fn verify_signatures(
    ctx: Context<VerifySignatures>,
    data: VerifySignaturesData,
) -> ProgramResult {
    // 取出当前交易的 instructions sysvar（理应是真实 sysvar 账户）
    let ix_acc = &ctx.accounts.instructions;       // <-- 漏洞：仅类型注解
                                                   //     未检查 ix_acc.key() == sysvar::instructions::ID

    // 读 index=0 指令，期望它是 ed25519/secp256k1 预编译调用
    let secp_ix = load_instruction_at(0, ix_acc)?;  // 旧 solana_program 函数
    require!(secp_ix.program_id == secp256k1_program::ID, BadSigIx);

    // 验证：由预编译产出的 messages 与 VAA body 一致
    verify_secp_payload(&secp_ix.data, &data.hash)?;
    // 通过后累加 Guardian 权重
    ...
    Ok(())
}
```

**错在哪**：`ctx.accounts.instructions` 只是 Anchor 里的字段命名，它**并未强制** `account.key == sysvar::instructions::ID`（Solana 系统唯一的 `Sysvar1nstructions1111111111111111111111111`）。攻击者可以传入自己构造的任意账户数据，使 `load_instruction_at(0, ix_acc)` 读到攻击者预先写好的"看起来像是 secp256k1 预编译验证通过"的假指令；由于合约信任了这个假 sysvar 的内容，便认为 Guardian 签名已被链上预编译验证通过，从而批准 VAA。

修复 commit 的关键 diff 是在 Cargo 依赖升级 `solana_program` 并把 `load_instruction_at` 换为 `load_instruction_at_checked`，该新函数会校验 account key 是真 sysvar。

### 3.4 攻击步骤分解

1. **观察 commit**：2022-01-13 commit `7edbbd3` 公开，研究者可从 diff 推断漏洞位置。
2. **构造假 sysvar 账户**：攻击者部署一条 Solana 交易，在参数里传一个自己拥有的普通账户，其 data 模拟合法 Instructions sysvar 布局，包含一条"secp256k1 程序调用，验证结果为 true"的伪指令。
3. **调用 `verify_signatures`**：传入伪造 sysvar 与一个自编 VAA（声称有 13 个 Guardian 签名通过、消息是"在 Solana 端 mint 120,000 wETH 给自己"）。合约读伪 sysvar 得出"预编译已验证"，接受 VAA。
4. **调用 `post_vaa` + `complete_wrapped`**：Wormhole Token Bridge 按已验证 VAA 铸造 120,000 wETH 至攻击者 Solana 地址 `CxegPrf...FBBka`。
5. **跨回 Ethereum**：调用 `transfer_wrapped` 发送 93,750 wETH 回 Ethereum Token Bridge `0x3ee18B2214AFF97000D974cf647E7C347E8fa585`，由 Ethereum 合约兑换为准备金中的真 ETH（桥侧是真正的 ETH 抵押）。
6. 桥被暂停前，攻击者已完成 Ethereum 侧兑换。

攻击主交易 Solana signature 为攻击者账户 `CxegPrfn2ge5dNiQberUrQJkHCcimeR4VXkeawcFBBka` 下 2022-02-02 UTC 18:22:xx 的 `complete_wrapped` 调用，具体 signature 可在 Solscan 该地址历史查询。

### 3.5 为何审计未发现

- Wormhole 曾由 Neodyme、Kudelski 审计，但 Solana 程序安全社区 2021 年尚未普及"所有 sysvar 账户必须硬校验 key"的清单；Anchor 框架直到 2022 中旬才默认添加 `#[account(address = sysvar::instructions::ID)]` 注解约束。
- 漏洞只在特定组合下暴露：使用 `load_instruction_at` 旧 API + 不做 key 校验 + 允许签名 context 里混入"攻击者可传递"的账户。
- commit 修复与主网部署之间有时间差，对手抢在部署前利用。这揭示了"公开修复 → 未立即部署" 是活体攻击信号（与 Poly Network、Nomad 同类模式）。

## 4. 事后响应

### 4.1 项目方动作

- **2022-02-02 事发后 ~3h**：Wormhole 广播漏洞并暂停桥；
- **2022-02-03**：合约升级至含 `load_instruction_at_checked` 的版本；
- Jump Crypto 链上注入 120,000 ETH 补齐准备金（以太坊侧 `0x3ee18B2214AFF97000D974cf647E7C347E8fa585`）；
- 对攻击者发起公开白帽协商：10% 赏金（$10M）换回资金，被拒。

### 4.2 资产追回

截至 2026-04，~93,750 ETH 仍在 `0x629e7da20197a5429d30da36e77d06cdf796b71a`，期间产出 stETH 奖励数千 ETH。追回率 0。

### 4.3 法律 / 执法

- Jump Crypto 自掏腰包后，以**民事诉讼方式**起诉 Oasis Protocol，借助 Oasis 金库升级权在 2023-02-24 反攻取回约 $140M 的资金（攻击者后来将资金存入 Oasis vault，在 Oasis 配合下由智能合约升级授权抽回）。此为 DeFi 史上最具争议的"链上诉讼式追回"。
- Chainalysis、TRM Labs 追踪但未正式归因国家行为体；FBI 未发布报告。**归因未公开**。

### 4.4 复查与审计

Trail of Bits 与 Neodyme 对修复版本做二次审计；Wormhole 发布 Immunefi **$10M** 白帽悬赏（当时业界最高）。Solana Foundation 发起"secure.solana.com"清单，sysvar 身份校验被列为首项。

## 5. 启发与教训

### 5.1 对开发者

- **Solana 程序的任何 sysvar / 预编译账户**都必须显式检查 `account.key == <known ID>`；Anchor 现代写法用 `#[account(address = ...)]` 或 `Sysvar<'_, Instructions>` 类型。
- 跨链桥的"签名验证"必须端到端：签名是谁验证的、消息是否是本合约期望的格式、调用 context 的账户是否可伪造。
- **修复 commit 部署前不公开**：安全补丁提交流程必须走 embargo，否则公共仓库即攻击信号。

### 5.2 对审计方

- Solana 程序审计清单（Neodyme / OtterSec）应作为最低要求；
- 审计时间差：commit 合并到主网上线之间的窗口必须做威胁评估；
- `load_instruction_at` 这类历史 API 应默认标红。

### 5.3 对用户

- 桥上 wrapped 资产本质依赖桥方偿付能力，Wormhole 的救援是"运气"（Jump Crypto 有钱愿意兜底），不是通用情况。

### 5.4 对协议

- 单一金主兜底不是可复制方案——之后 LayerZero、Axelar 强调"去中心化 validator + 经济激励保险"。
- 严重 bug 修复应走 SRC 流程：内部部署 → 公开 → 主网升级 → 发 commit。Wormhole 的教训直接塑造了 Immunefi "Safe Harbor" 框架。

## 6. 参考资料

- Wormhole 官方 Incident Report：https://wormholecrypto.medium.com/wormhole-incident-report-02-02-22-ad9b8f21eec6
- CertiK 分析：https://blog.certik.com/post/wormhole-bridge-exploit-incident-analysis
- rekt.news：https://rekt.news/wormhole-rekt/
- Neodyme deep-dive：https://medium.com/@neodyme/wormhole-hack-analysis-ab03d2d1c05b
- 修复 commit：https://github.com/wormhole-foundation/wormhole/commit/7edbbd3677ee6ca681be8722a607bc576a3912c8
- @samczsun 事件时间线 Twitter thread（2022-02-03）
- Jump Crypto 注资公告：https://medium.com/@jump_crypto
- Oasis / Jump 追回事件：https://oasis.app/blog/oasis-multisig

---

*Last verified: 2026-04-23*
