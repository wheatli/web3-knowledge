---
title: Nomad Bridge "复制粘贴"被盗事件（2022-08-01, ~$190M）
module: 08-security-incidents
category: Bridge
date: 2022-08-01
loss_usd: 190000000
chain: [Ethereum, Moonbeam, Avalanche, Evmos, Milkomeda]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://medium.com/nomad-xyz-blog/nomad-bridge-hack-root-cause-analysis-875ad2e5aacd
  - https://rekt.news/nomad-rekt/
  - https://www.certik.com/resources/blog/nomad-bridge-exploit-incident-analysis
  - https://twitter.com/samczsun/status/1554252024723546112
  - https://github.com/nomad-xyz/monorepo/commit/61ed91ed16fb3bd9e6b26376b0ebb64f21020377
tx_hashes:
  - 0xa5fe9d044e4f3e5aa5bc4c0709333cd2190cba0f4e7f16bcf73f49f83e4a5460（第一笔利用 tx）
  - 0xb09f30b0e45c5a672f1a3f86d5f456b9b78e1ffd4e0dbe02e1dc7a5b1c3b0f20（示例跟随 tx；Nomad 事件数千笔跟随 tx 散见 Etherscan 对 Replica 合约）
  - Nomad Replica 合约: 0x5D94309E5a0090b165FA4181519701637B6DAEBA
  - Nomad Bridge: 0x88A69B4E698A4B090DF6CF5Bd7B2D47325Ad30A3
---

# Nomad Bridge "复制粘贴"被盗事件

> **TL;DR**：2022-08-01，通用消息跨链桥 Nomad 在一次看似常规的合约升级里，把 `acceptableRoot[0x00...00]` 由 `0x00` 改为 `0x01`（非零），导致 `process()` 的"`messageRoot` 必须已被接受"检查形同虚设：任何人只要提交一条以未知 `messageHash` 开头、抄袭一份真实 `process()` 调用、把收款人地址改成自己的 calldata，就能直接提走桥里的 token。由于利用方法只需要**复制前人的成功 tx 改一个地址**，结果演变成**数千个地址公开抢钱**的链上"公共洗劫"，最终 $190M 被 300+ 地址分食。链上历史少见的群体性事件。

## 1. 事件背景

### 1.1 Nomad

Nomad 是 2022 年春季上线的乐观式跨链消息协议，由 James Prestwich 主导，在 Illusory Systems 团队（后更名 Nomad XYZ）研发。它的"optimistic bridge"设计以 updater 提交消息 root + 30 分钟挑战窗口代替多签，声称只需 1 个诚实参与者即可保证安全。2022-04 拿到 $22.4M 种子轮，估值 $225M，投资方包括 Polychain、Coinbase Ventures。

到 2022-08 初，Nomad 桥 TVL 约 **$190M**，承载 Moonbeam ↔ Ethereum 的主要流动性（GLMR、WETH、USDC、WBTC、FRAX、CQT 等）。

### 1.2 时间轴

- **2022-04-21**：Nomad 部署 `Home` / `Replica` 合约于 Ethereum 与 Moonbeam 等。
- **2022-04-27**：Quantstamp 完成审计并发布报告，包含若干 medium 级发现。
- **2022-06-21**：Nomad 合并 `Replica` 合约升级，commit `61ed91e...` 修改 `initialize` 函数，把 `_committedRoot` 作为 `acceptableRoot` 的 key 初始化。升级部署后 `acceptableRoot[0x00]` 因为代码把 `_committedRoot = 0x00` 初始化 **而错误地被赋值为 1**（含 fallback 逻辑）。
- **2022-08-01 UTC 21:32**：第一笔利用交易（被白帽 Etherscan 标注）出现。
- **2022-08-01 UTC 21:40–23:00**：Twitter 上 @samczsun、@officer_cia 警告"Nomad 正在被耗空"，任何懂复制粘贴的人都可以加入。数百地址在 2 小时内抢走几乎全部 TVL。
- **2022-08-02 UTC 02:00**：Nomad 官方 Twitter 确认事件、冻结前端。
- **2022-08-22**：Nomad 发起公开白帽返还通道；约 $37M 在数月内返还。

### 1.3 发现过程

Etherscan 上不明 tx 从 Replica 合约 `0x5D94309E5a0090b165FA4181519701637B6DAEBA` 放出一大笔 WBTC，几分钟内出现完全雷同、只改收款人的 tx。@samczsun 在 UTC 21:53 发推 "Nomad is being drained"。由于利用方式极简，上 Etherscan 抄 calldata 换地址即可，普通 MEV / 脚本小子也在事件 30 分钟内参与，其中还夹杂白帽（事后返还）。

## 2. 事件影响

### 2.1 直接损失

- 总流失 **$190M**（Nomad 全部 TVL 的 99%）；
- 损失 token：WBTC、WETH、USDC、FRAX、CQT、DAI、HBOT 等；
- 参与地址：> 300，其中 40+ 被归类为白帽/机会主义；
- 所有损失来自 `Replica` 合约 on-chain free-for-all。

### 2.2 受害方

- Moonbeam ↔ Ethereum 跨链 LP / 用户；
- Connext Bridge（彼时共用 Nomad messaging）短期功能停摆；
- Saddle Finance、Curve 上的 nomadUSDC / nomadWETH 池脱锚归零；
- Moonbeam 生态总 TVL 一周内跌 40%。

### 2.3 连带影响

- "Optimistic bridge" 叙事遭重大打击，行业重新评估"1 诚实 participant"假设；
- Quantstamp 审计声誉受挫，后续公开道歉并赞助漏洞赏金；
- 加深 "upgrade + initializer bug" 主题研究，OpenZeppelin 发布增强版 `Initializable` 模式。

### 2.4 资金去向

- 约 **$37M+** 在 Nomad "白帽返还地址" `0x94A84433101A10aEda762968f6995c574D1bF154` 内收齐；
- 其余进入 Tornado Cash、各类 CEX；
- 归因：Nomad 官方 post-mortem **未归因**；参与者分散至多个国家，不构成单一 APT 行为。**APT 归因未公开**。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**初始化 / 升级错误**（initialization bug），叠加 **Merkle root 接受逻辑失效**。

### 3.2 受损模块

- 合约：`contracts/core/Replica.sol`
- 仓库：`github.com/nomad-xyz/monorepo`
- 决定性 commit：`61ed91ed16fb3bd9e6b26376b0ebb64f21020377`（2022-06-21 合并）
- 主网 Replica 地址：`0x5D94309E5a0090b165FA4181519701637B6DAEBA`

### 3.3 关键代码片段

```solidity
// contracts/core/Replica.sol （漏洞版本）
// 正常流程：updater 签名的 root 通过 update() 进入 confirmAt；过 challenge 后写入 acceptableRoot
// 然后 process(bytes _message) 只允许处理 acceptableRoot[computedRoot] != 0 的消息

function initialize(
    uint32 _remoteDomain,
    address _updater,
    bytes32 _committedRoot,   // <-- 新版 initialize 新增参数
    uint256 _optimisticSeconds
) public initializer {
    __NomadBase_initialize(_updater);
    committedRoot = _committedRoot;
    // 将初始 root 标记为可接受（开发者本意：如果有初始 root 则置 1，否则跳过）
    acceptableRoot[_committedRoot] = 1;   // <-- 漏洞行：未判空
    // 在主网实际部署时，_committedRoot 传入了 0x00...00
    // 结果：acceptableRoot[bytes32(0)] = 1
    ...
}

function process(bytes memory _message) public returns (bool) {
    // 1. 对消息做 keccak，拼 Merkle path
    bytes32 _messageHash = keccak256(_message);

    // 2. 计算或读取该消息所属 root
    bytes32 _root = messages[_messageHash];   // 对未处理过的消息而言，默认 = bytes32(0)

    // 3. 要求该 root 在可接受集合中
    require(acceptableRoot[_root] == 1, "!proven"); // <-- 漏洞：_root = 0 时通过！

    // 4. 调用 handler 根据 body 决定收款人、token、金额
    _handle(_origin, _sender, _body);  // 这里直接按 _body 解出 recipient 把桥里 token 转走
    return true;
}
```

**错在哪**：
- `acceptableRoot[bytes32(0)] = 1` 是 **初始化无判空分支的后果**；
- `process()` 第 3 行要求 `acceptableRoot[_root] == 1`，而任何**首次被提交的消息** `messages[h]` 默认值是 `bytes32(0)`，于是 `acceptableRoot[0]` 通过；
- 换言之，**任何 `bytes _message` 只要格式合法就能直接 process()**，无需 updater 签名、无需过挑战窗口、无需 Merkle 证明。

### 3.4 攻击步骤分解

1. **第一起利用**（某位研究者或攻击者）：构造一条 `_body`，声明"把桥里 100 WBTC 发到我的地址"，调用 `process(bytes)`，合约通过，WBTC 落袋。tx: `0xa5fe9d04...`。
2. **复制粘贴扩散**：攻击 tx 在 Etherscan 公开，任意地址把同一 calldata 复制、只换 `recipient` 字段中 20 字节，继续调用 `process`。数千笔 tx 在 2 小时内涌入。
3. **MEV bot 跟进**：bot 监听 `Replica.process` event，自动 bundling 以更高 gas 抢跑大额 token 提取。
4. **后期白帽**：部分 security researcher（如 Dragonfly、@samczsun 等）也按同样方法"抢在坏人前"拿到资金，再等 Nomad 官方公布返还地址后归还（37M+）。
5. **耗尽**：Nomad TVL 几乎归零。

### 3.5 为何审计未发现

- Quantstamp 2022-04 审计版本不含该 initializer 分支——**漏洞由 2022-06-21 升级引入**，属于"审计后变更未再审"。
- Nomad 内部代码审查把 `acceptableRoot[_committedRoot] = 1` 视为"无害初始化"，未覆盖 `_committedRoot = 0` 的边界。
- 部署脚本传入 `0x0` 作 `_committedRoot` 视作"无历史 root 初始化"，没有 guard。

## 4. 事后响应

### 4.1 项目方动作

- **2022-08-02**：Nomad 关闭前端、联系 Chainalysis / TRM / 各大 CEX 配合追踪；
- **2022-08-06**：发布根因分析，坦承 `initialize` 漏洞；
- **2022-08-22**：公布白帽返还地址 `0x94A84433101A10aEda762968f6995c574D1bF154`，奖励 10%；
- 桥长期暂停；Nomad 后续与 Connext 合作转型为消息层项目。

### 4.2 资产追回

返还约 $37M+（其中较大笔包括 Dragonfly $5M、Worldcoin 团队约 $4M、多位匿名白帽）。正式黑客并未大规模返还。

### 4.3 法律 / 执法

- FBI 未发布针对 Nomad 的归因；
- 由于参与者众多、许多系"跟风小用户"，美国司法部对个别地址发出 subpoena；
- Chainalysis 2023 年报告把 Nomad 事件列为"chaotic free-for-all"代表案例。

### 4.4 复查与审计

- Halborn、OpenZeppelin 对 Nomad v2 做新审计；
- OpenZeppelin 发表博文《Nomad Bridge Hack Root Cause Analysis》，倡导 `Initializable` 增强版：初始化参数中任何 "重要 zero-val fallback" 必须 revert；
- 业界普遍在 CI 中加入"差异审计（diff audit）"——升级后必须审新增/变更函数全部分支。

## 5. 启发与教训

### 5.1 对开发者

- **升级即新代码**。任何合约升级必须重启审计窗口；
- `initialize` 函数的输入不可信，所有写入 `mapping` 的赋值必须做零值 / 默认值检查；
- "Known bad inputs" 的回归测试用例必须包含 `bytes32(0)`、`address(0)`、`uint256.max` 等边界；
- 关键断言 (`acceptableRoot[r] == 1`) 应加独立 sentinel：`_root != bytes32(0)` 显式 require，杜绝"默认零值 = 接受"。

### 5.2 对审计方

- 审计报告要标注"审计快照 commit hash"，并警示所有 **后续 diff 必须再审**；
- Slither / Semgrep 自定义规则：`mapping[T] = 1` + key 可能为 zero-sentinel 时告警；
- 形式化：lean4 / Certora 规则 "`acceptableRoot[x] == 1` 蕴含 x 为有效 root"。

### 5.3 对用户

- 跨链桥的"先进"叙事（optimistic / ZK / PoA / MPC）都不豁免 initializer bug；
- 大额用户应分仓、避免把跨链当长期托管。

### 5.4 对协议

- 升级流程必须有 "canary"：先用小额测试 replica；
- 部署脚本必须在上链前 dry-run 并产出状态快照；
- 监控告警：`process()` 调用量异常 / 无 `update()` 前驱事件的 process 应即时报警。

## 6. 参考资料

- Nomad 官方 Root Cause：https://medium.com/nomad-xyz-blog/nomad-bridge-hack-root-cause-analysis-875ad2e5aacd
- CertiK 分析：https://www.certik.com/resources/blog/nomad-bridge-exploit-incident-analysis
- rekt.news：https://rekt.news/nomad-rekt/
- @samczsun 实时 thread：https://twitter.com/samczsun/status/1554252024723546112
- 漏洞提交 commit：https://github.com/nomad-xyz/monorepo/commit/61ed91ed16fb3bd9e6b26376b0ebb64f21020377
- OpenZeppelin：https://blog.openzeppelin.com/nomad-hack-analysis-and-bug-bounty-update
- Nomad 返还地址：`0x94A84433101A10aEda762968f6995c574D1bF154`
- 利用 tx：`0xa5fe9d044e4f3e5aa5bc4c0709333cd2190cba0f4e7f16bcf73f49f83e4a5460`

---

*Last verified: 2026-04-23*
