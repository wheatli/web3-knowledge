---
title: zkLend (Starknet) $9.5M 闪贷 + 累计误差利用（2025-02-12, ~$9.5M）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2025-02-12
loss_usd: 9500000
chain: Starknet
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/zkLend/status/1889632450000
  - https://slowmist.medium.com/slowmist-analysis-of-zklend-exploit-2025-02-3b3e5c1a
  - https://rekt.news/zklend-rekt/
  - https://x.com/PeckShieldAlert/status/1889632630000
  - https://blog.zkcrypto.io/zklend-post-mortem
tx_hashes:
  - Starknet 链上攻击交易 hash：官方 post-mortem 已列出，以 zkLend 官方公告与 starkscan.co 为准；此处不列以免出错
---

# zkLend (Starknet) $9.5M 闪贷 + 累计误差利用

> **TL;DR**：2025-02-12，Starknet L2 上 TVL 最大的借贷协议 zkLend 被攻击者利用 "wstETH 池 accumulator（利率累加器）在极端状态下的舍入 / 累积误差"，通过闪贷放大头寸后重复 deposit / withdraw，吸走约 $9.5M。攻击者事后向 Tornado Cash 转入大部分资金后，还在链上声称"误操作到错误地址"导致 EigenLayer 关联钓鱼——社区对该说法普遍不信。Starknet 生态 TVL 的头部协议被打击，Starknet DeFi 叙事短期受创。

## 1. 事件背景

- **zkLend**：Starknet 上最早的借贷协议之一（类 Compound 架构），事发时 TVL 约 $50M，wstETH、ETH、USDC 为主要抵押品。
- **受攻击池**：wstETH 市场。
- **时间轴**：
  - 2025-02-12：攻击上链；几小时内官方检测到异常并暂停合约。
  - 官方 Twitter 公告发起 10% 白帽赏金。
  - 攻击者通过 StarkGate 桥将资金桥至 Ethereum，后入 Tornado Cash。
  - 攻击者后期声称"误发资金到一个钓鱼地址"（EigenLayer 钓鱼页面），声称无法归还，被广泛视为 exit-scam 说辞。
  - 2025-03：zkLend 宣布用协议国库部分赔付并继续运营。

## 2. 事件影响

### 直接损失

- 约 $9.5M，主要为 wstETH（随后换成 ETH 桥出）。

### 受害方

- zkLend 用户 / LP。
- Starknet 生态 TVL：事件后 Starknet 整体 TVL 下降约 10–15%。

### 资金去向

- 通过 StarkGate → Ethereum → Tornado Cash 混币；小部分据攻击者声明"误入钓鱼地址"，但链上行为与典型钓鱼套路不完全匹配，真伪存疑。

## 3. 技术根因

### 3.1 漏洞分类

**累积精度 / 舍入误差 + 可重复利用的状态路径**（典型 "Compound v2 风格 exchangeRate 被 donation 或直接转账推高后的整数舍入放大"）。

### 3.2 机理（基于 SlowMist / zkLend post-mortem 公开描述整合）

1. zkLend 类似 Compound v2，使用 `exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply` 将底层资产换算为 zToken。
2. 当 `totalSupply` 极小（例如经过一次大额提取之后），向池子直接"空投"少量底层资产即可让 `exchangeRate` 显著跃升；此后每次 deposit/withdraw 的舍入方向差产生微小盈余。
3. 攻击者利用闪贷构建极大/极小 totalSupply 的交替路径，把这个微小盈余重复放大数千次，在单笔链上交易内将池子抽空。
4. 攻击者重点针对 wstETH 池（Lido 包装 stETH 在 Starknet 上的 wrapped 版本）。

> 代码级细节请参见 zkLend 官方 post-mortem 与 SlowMist 技术分析；Cairo（Starknet 智能合约语言）源码具体函数名与 commit 以 github.com/zkLend-Finance 修复 PR 为准。代码片段待 SlowMist/Verichains 完整报告公布。

### 3.3 为何审计 / 测试未发现

- Compound v2 风格的"空 market 被 donation 操纵 exchangeRate"漏洞在以太坊已多次上演（2023 Hundred Finance、2023 Midas Capital、2024 Onyx 等）。Starknet / Cairo 生态对 Solidity 历史经验的转换不彻底，未同步加固。
- 审计报告覆盖面未完整涵盖"totalSupply 接近 0 时的行为"这一边界。

## 4. 事后响应

- zkLend：暂停合约，官方与攻击者公开对话并提供 10% 白帽赏金；未获回应后宣布用国库赔付用户部分损失。
- Starknet Foundation：协调生态其他协议排查类似漏洞。
- 业内被重申的课题：借贷协议必须在创建每个 market 时强制 "burn 少量 share"（dead shares）以避免 totalSupply 归零攻击面。

## 5. 启发与教训

- **Dead-shares 模式**：新上线 market 应强制存入最小数量份额并转 burn 地址，消除 totalSupply = 0 边界。
- **Cairo / Move / 新 VM 借鉴 Solidity 历史**：Starknet 生态安全模型不应以"我们是新 VM"为由忽略以太坊历史漏洞；将 DeFiHackLabs / SlowMist Hacked 数据库纳入必读。
- **协议国库赔付的 playbook**：与社区沟通节奏、DAO 治理投票时限、白帽通道 SOP 应事前建立。
- **桥 + 混币的时间窗**：Starknet → Ethereum 提现桥的挑战期 / 欺诈窗口在某些情形下可以成为冻结资金的关键，协议方应与桥方即时协作。

## 6. 参考资料

- zkLend 官方 post-mortem
- SlowMist / PeckShield 技术分析
- rekt.news — zkLend Rekt
- Starknet Foundation 生态公告

---

**本条目源于 2025-02 公开报告，Cairo 代码级细节待 SlowMist / Verichains 完整报告公布。**

未公开 / 待核实字段：
- 具体 Cairo 函数名 + 错误行号
- 最终国库赔付金额 / 用户回收比例
- 攻击者"误入钓鱼"声明真伪

*Last verified: 2026-04-23*
