---
title: Beanstalk Farms 治理闪电贷攻击（2022-04-17, ~$182M）
module: 08-security-incidents
category: Governance
date: 2022-04-17
loss_usd: 182000000
chain: [Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://bean.money/blog/beanstalk-governance-exploit
  - https://rekt.news/beanstalk-rekt/
  - https://blog.openzeppelin.com/13-4m-governance-exploit-beanstalk/
  - https://www.certik.com/resources/blog/1NHvrWGCOa4Z80ttNYfVFE-beanstalk-incident-analysis
  - https://twitter.com/peckshield/status/1515784513034686465
tx_hashes:
  - 0xcd314668aaa9bbfebaf1a0bd2b6553d01dd58899c508d4729fa7311dc5d33ad7（Beanstalk 攻击主 tx）
  - 0x887ae5c8af3f26ae17e384a0c5d768e363274541ff7d8c01108de3dd4a47de31（提案注册 tx，commit）
  - Beanstalk Diamond: 0xC1E088fC1323b20BCBee9bd1B9fC9546db5624C5
  - Attacker: 0x1c5dCdd006EA78a7E4783f9e6021C32935a10fb4
---

# Beanstalk Farms 治理闪电贷攻击

> **TL;DR**：2022-04-17，稳定币协议 Beanstalk Farms 被一笔 10 亿美元级闪电贷（Aave + Uniswap V2）注入获得 **> 67% Stalk 治理投票权**，立刻触发 `emergencyCommit` 批准攻击者前一天提交的 BIP-18 / BIP-19 提案，把 Beanstalk 金库中所有资产转给攻击者地址。共损失约 **$182M**（协议净流失 ~$77M，其余为闪电贷返还部分）。根因为治理模块**无 timelock + emergencyCommit 可在同一笔 tx 内完成投票 → 执行**。

## 1. 事件背景

### 1.1 Beanstalk

Beanstalk（BEAN）是一个 credit-based 算法稳定币协议，2021-08 上线，以"Silo（存款池）→ Stalk（治理权重）→ Pod（债券）"三层机制维持 BEAN 锚定 $1。存款人获得 Stalk 代币代表治理权重与收益分成，任何提案需要 Stalk quorum。

2022-04 事件前 TVL 约 $150M，Stalk 总量集中在少数长期 LP。官方审计方为 Omniscia（2021-10 发布），该报告**未涉及** `emergencyCommit` 路径——该函数于上线后 fork 期间加入。

### 1.2 时间轴

- **2022-04-16**：攻击者钱包 `0x1c5dCdd006EA78a7E4783f9e6021C32935a10fb4` 在链上向 Beanstalk DAO 提交两份恶意提案：
  - **BIP-18**："向 Ukraine Relief 地址捐赠 $250k"（烟雾幕）；
  - **BIP-19**：表面用相同措辞，实则调用 Diamond `init()` 将 Silo 全部 3pool LP 与 BEAN:3CRV LP 转至攻击者地址。
- **2022-04-17 UTC 12:24**：攻击者在 Flashbots 批量打包如下流程的交易 `0xcd314668...`：闪电贷 → 存入 Silo → 获得 Stalk → `emergencyCommit(BIP-18)` → `emergencyCommit(BIP-19)` → 执行捐赠 + 抽空 → 还闪电贷 → 留下 ~$77M 利润。
- **2022-04-17 UTC 12:40**：PeckShield 与 @peckshield、Igor Igamberdiev 最先发出警报。
- **2022-04-17 UTC 14:00**：Beanstalk Farms 官方确认事件、暂停前端，BEAN 脱锚至 $0.10。
- **2022-08-06**：Beanstalk 通过社区重启 Replant，使用新合约与 timelock 模型复活协议。

### 1.3 发现过程

链上 mempool + Flashbots bundle 一曝光 @peckshield 立刻推文"$80M+ drained from Beanstalk via flashloan + governance exploit"。

## 2. 事件影响

### 2.1 直接损失

- 协议 Silo 被抽出 24,830 ETH + 36M BEAN + 0.54M BEAN:3CRV LP + 0.84M BEAN:LUSD LP；
- 按当日价近 **$182M 名义资产移动**（含闪电贷部分），攻击者**净利润 ~$77M**（约 24,830 ETH），其余用于还闪电贷；
- 攻击者另向"Ukraine 捐赠"地址 `0x16e02...` 转出 $250k 作为掩护。

### 2.2 受害方

- Beanstalk 所有 Silo 存款人本金归零，BEAN 失去抵押；
- 协议治理代币 Stalk 价值归零；
- 关联 LP（BEAN:3CRV、BEAN:LUSD）清盘。

### 2.3 连带影响

- 第一个被广泛引用的"闪电贷 → 治理代币 → timelock-less 治理执行"样板，促发后续所有 governance 合约把 timelock 写入硬性要求；
- 业界开始区分"治理提案注册 → 投票 → 执行"必须的最短时间间隔（通常 ≥ 24h 甚至 7 天）；
- 对 Curve / Compound / Aave 的 `GovernorBravo` 设计成为事实上的行业模板的推动事件之一。

### 2.4 资金去向

净利润 24,830 ETH 通过 Tornado Cash 洗净。**归因未公开**，Beanstalk post-mortem、Chainalysis 均未指明 Lazarus；攻击特征（闪电贷 + 经济学型漏洞）更接近独立黑客。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**治理设计缺陷**（governance flash-loan attack）：缺 timelock + emergencyCommit 门槛可被闪电贷满足。

### 3.2 受损模块

- Beanstalk Diamond (EIP-2535) 地址：`0xC1E088fC1323b20BCBee9bd1B9fC9546db5624C5`
- Facet：`GovernanceFacet.sol`、`SiloFacet.sol`
- 关键函数：`emergencyCommit(uint32 bip)`、`commit()`, `vote()`

### 3.3 关键代码片段（基于事件时版本，伪代码精简）

```solidity
// GovernanceFacet.sol（事件时版本）
// Stalk 为治理权重（由存入 Silo 时 mint，本笔 tx 可立即获得）

function vote(uint32 bip) external {
    require(isActive(bip), "!active");
    stalkFor[bip][msg.sender] = balanceOfStalk(msg.sender); // 按当前快照
    totalStalkFor[bip] += stalkFor[bip][msg.sender];
}

function emergencyCommit(uint32 bip) external {
    require(isActive(bip), "!active");
    // 支持率 >= 2/3 Stalk 即可立即执行，跳过正常 7 天投票
    require(totalStalkFor[bip] * 3 >= totalStalk() * 2, "!super-majority");
    // 没有 timelock —— 问题所在
    _execute(bip);   // bip 由攻击者提交，init() 会把 Silo 所有资金转出
}

// 严重点：
// 1. Stalk 由当前区块 Silo 余额映射得到，闪电贷存入可瞬间放大；
// 2. emergencyCommit 无 24h/7d timelock；
// 3. proposals 提交只需等待 0-ish 区块就可进入 `isActive`（没有最小 age）。
```

**错在哪**：
- `emergencyCommit` 把"投票 + 执行"合并到**单笔交易可完成**的路径；
- Stalk 权重取自当前区块——没有 snapshot / checkpoint，也没有"提案提交前 X 区块持仓"的 `getPriorVotes`；
- 提案内容是任意 `init()` delegatecall，可以执行**任意**合约逻辑（Diamond 特性），包括把 Silo 资金转出。

### 3.4 攻击步骤分解（单笔 tx `0xcd314668...`）

1. **准备**（4/16）：攻击者以新地址 submit BIP-18 和 BIP-19。BIP-19 的 `init()` 指向攻击者部署的 `InitBip18` 合约，该合约 delegatecall 后从 Silo 取出全部 3pool LP、BEAN:3CRV LP、BEAN:LUSD LP 发送到攻击者。
2. **闪电贷**（单 tx）：
   - Aave V2 借 350M DAI、500M USDC、150M USDT；
   - Uniswap V2 借 32M BEAN。
3. **存入 Silo**：用借来的稳定币在 Curve 打入 BEAN:3CRV LP，再把 LP deposit 到 Silo；1 秒内铸造出巨量 Stalk，总量远超存量 Stalk 的 2/3。
4. **投票 + 执行**：
   - `vote(BIP-18)`：烟雾幕；
   - `emergencyCommit(BIP-18)`：执行捐赠 $250k 到乌克兰；
   - `vote(BIP-19)`、`emergencyCommit(BIP-19)`：delegatecall 恶意 init()，把 Silo 中 24,830 ETH 等值 LP 提取到攻击者。
5. **还闪电贷**：把稳定币 + fee 还给 Aave / Uniswap，剩余 ~24,830 ETH 留给攻击者。
6. **离场**：24,830 ETH 全部通过 Tornado Cash 洗出。

整个过程在 **1 笔 tx 内完成**，发生在同一区块，这是该攻击典范性之所在。

### 3.5 为何审计未发现

- Omniscia 审计覆盖 Silo 和发行逻辑，`emergencyCommit` 路径是 2021-10 审计之后加入，**未再审**；
- 审计方主观上假设"stalkholder 是长期 LP"——未考虑到 Stalk 可在同一笔 tx 内瞬时铸造；
- 当时 DeFi 治理模型常识尚未形成"snapshot checkpoint + timelock"标配（Compound Bravo 有但 Beanstalk 自写）。

## 4. 事后响应

### 4.1 项目方动作

- 紧急暂停前端、链上合约 `pause()`；
- 团队 KYC 本身为匿名（主理人 Publius 三人组后被迫半公开）；
- 众筹 Barn Raise 募集 $77M 用于重启；
- 2022-08-06 **Replant**：启用新 Diamond 部署 `0xC1E088fC1323b20BCBee9bd1B9fC9546db5624C5` 中加入 `Season` / `Sunrise` 治理时间 lock，禁止 same-block 投票执行。

### 4.2 资产追回

链上追回 0。社区通过 Barn Raise 募集 Fertilizer 代币补偿历史 Silo 存款人（按债券模型分期偿还）。

### 4.3 法律 / 执法

Beanstalk 在美国司法部立案。攻击者身份至今未被确认；"Ukraine 捐赠"一度引发媒体戏剧性报道，并非减轻情节。

### 4.4 复查与审计

- Halborn、Trail of Bits 对 Replant 版做审计；
- OpenZeppelin 发文《13.4M Governance Exploit Beanstalk》（注：OZ 的数字口径按攻击者净利润不同计量）。

## 5. 启发与教训

### 5.1 对开发者

- **治理权重必须基于 past-block snapshot**（如 `ERC20Votes.getPastVotes(block-1)`），禁止同一笔 tx 存款即投票；
- **所有执行路径必须 timelock**（24h/48h/7d 根据风险），`emergencyCommit` 路径必须极其严格（例如需要多签白名单 + 冷却期）；
- 禁止治理 delegatecall 到任意 init() 合约，或加地址白名单。

### 5.2 对审计方

- 对 Diamond / Proxy 架构必须审计**所有可升级 Facet**，包括治理；
- Governance 审计 checklist：snapshot、quorum、proposal minimum age、execution delay、闪电贷抵抗；
- Certora 规则：`voteWeight(proposalId, voter) <= stalkBalanceAt(proposalId.snapshotBlock)`。

### 5.3 对用户

- 匿名团队 + 无 timelock 治理 = 典型高危；
- 关注协议治理参数：提案冷却期、emergency 路径门槛、投票权快照；

### 5.4 对协议

- 治理事件必须有最短投票窗口与执行延迟；
- 引入 Tally/Snapshot/OZ Governor 等经过验证组件，减少自研风险；
- emergency 路径收敛到可多签确认的白名单 action（pause/unpause），不可扩展到"任意调用"。

## 6. 参考资料

- Beanstalk 官方 post-mortem：https://bean.money/blog/beanstalk-governance-exploit
- rekt.news：https://rekt.news/beanstalk-rekt/
- OpenZeppelin 分析：https://blog.openzeppelin.com/13-4m-governance-exploit-beanstalk/
- CertiK 分析：https://www.certik.com/resources/blog/1NHvrWGCOa4Z80ttNYfVFE-beanstalk-incident-analysis
- PeckShield 实时 thread：https://twitter.com/peckshield/status/1515784513034686465
- 攻击主交易：`0xcd314668aaa9bbfebaf1a0bd2b6553d01dd58899c508d4729fa7311dc5d33ad7`
- Beanstalk Diamond：`0xC1E088fC1323b20BCBee9bd1B9fC9546db5624C5`
- 攻击者地址：`0x1c5dCdd006EA78a7E4783f9e6021C32935a10fb4`

---

*Last verified: 2026-04-23*
