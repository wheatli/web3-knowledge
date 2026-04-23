---
title: Euler Finance 闪电贷攻击（2023-03-13, ~$197M）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2023-03-13
loss_usd: 197000000
chain: Ethereum
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://blog.euler.finance/upgradability-and-the-donateToReserves-function-a-post-mortem-of-the-euler-attack-14d9c7a1a9e
  - https://slowmist.medium.com/slowmist-analysis-of-the-euler-finance-attack-b29e5d4b47ed
  - https://rekt.news/euler-rekt/
  - https://www.certik.com/resources/blog/2NgvoP47xUxs8l6C4bgpK4-euler-finance-incident-analysis
  - https://blog.chainalysis.com/reports/euler-finance-attack-2023/
tx_hashes:
  - 0xc310a0affe2169d1f6feec1c63dbc7f7c62a887fa48795d327d4d2da2d6b111d (Ethereum, 主攻击交易之一)
  - 0x71a908be0bef6174bccc3d493becdfd28395d8898aa0a53d65ae32ed7ef76d42 (Ethereum, DAI 池攻击)
  - 0x465a6780145f1efe3ab52f94c006065575712d2003d83d85481f3d110ed131d9 (Ethereum, WBTC 池)
  - 0x62bd3d31a7b75c098ccf28bc4d4af8c4a191b4b9e451fab4232258079e8b18c4 (Ethereum, stETH 池)
---

# Euler Finance 闪电贷攻击

> **TL;DR**：2023-03-13，借贷协议 Euler Finance 被闪电贷 + self-liquidation 组合攻击，损失约 1.97 亿美元（DAI/WBTC/USDC/stETH）。根因是 `eToken.donateToReserves()` 函数在提交捐赠时未调用 `checkLiquidity()`，允许用户把自己的账户推入"违约但未被清算"的人为不健康状态，再触发自清算以极低折价购回抵押。事件戏剧性转折：攻击者在三周内分批归还全部资金，成为 DeFi 历史上最大白帽化案例。

## 1. 事件背景

- **协议**：Euler Finance，基于 Compound v2 / Aave 思路迭代的无需许可借贷市场，支持长尾资产。攻击前 TVL 约 2.6 亿美元。2021 年由 Michael Bentley（牛津大学博士）创立，Paradigm、Coinbase Ventures 领投。
- **审计记录**：曾经历 Halborn、Solidified、ZK Labs、Certora、Omniscia、Sherlock 等至少 6 轮审计与形式化验证。2022-07-13 引入的 `donateToReserves` 函数在一次小版本升级中加入，彼时审计已基本完成。
- **时间轴**：
  - 2022-07-13：合约升级 EIP-17，新增 `donateToReserves`。
  - 2023-03-13 08:50 UTC：首笔攻击交易落地。
  - 2023-03-13 09:30 UTC：6 个连续池子被抽空。
  - 2023-03-13 10:24 UTC：Euler 官方 Twitter 确认事件，暂停协议。
  - 2023-03-17：攻击者开始零星归还部分资金。
  - 2023-04-04：攻击者归还全部主钱包资金，并公开"Jon, I am sorry"道歉信。
- **发现**：Meta Sleuth / BlockSec 监控系统率先告警；Meir Bank 在 Twitter 首发 tx。

## 2. 事件影响

- **直接损失**（tx 全部可 Etherscan 验证）：
  - DAI 池：约 8,878,502 DAI
  - WBTC 池：约 849 WBTC（约 $19.3M）
  - stETH 池：约 85,818 stETH（约 $140M）
  - USDC 池：约 34M USDC
  - 合计按当日价约 $197M
- **受害方**：Balancer（Boosted USD 池暴露 $11.9M）、Angle Protocol（$17.5M agEUR 储备）、Yearn yvUSDT / yvUSDC 策略、Idle Finance、Yield Protocol、SwissBorg、Gearbox、Inverse Finance 等至少 11 个集成方。
- **连带影响**：DeFi TVL 当日跌 3%；eToken/dToken 价格脱钩导致 Curve EUR 相关池套利；长尾借贷市场设计被行业重新审视。
- **资金去向**：攻击者先通过 Tornado Cash 混入 1000 ETH，后主动归还。最终 Euler Labs 与攻击者达成链下谈判，所有资金悉数返还用户（含部分 USDC 超过原始损失）。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**协议逻辑缺陷（Protocol Bug）**——借贷健康因子校验路径不一致。既非重入也非溢出，而是"两条可抵达相同状态的路径，健康性检查只装在其中一条上"。

### 3.2 受损合约

- 主合约：Euler `EToken.sol`，部署地址 `0xEb91861f8A4e1C12333F42DCE8fB0Ecdc28dA716`
- 关键函数：`donateToReserves(uint subAccountId, uint amount)`（2022-07-13 升级引入）
- 源码仓库：`https://github.com/euler-xyz/euler-contracts`（commit 8d3a89f 前后对比）

### 3.3 关键代码片段（简化伪代码）

```solidity
// ===== 错误版本（攻击发生时）=====
function donateToReserves(uint subAccountId, uint amount) external {
    address account = getSubAccount(msg.sender, subAccountId);
    AssetStorage storage assetStorage = eTokenLookup[address(this)];
    AssetCache memory assetCache = loadAssetCache(...);

    uint balance = assetStorage.users[account].balance;
    require(amount <= balance, "insufficient");

    // 扣减用户 eToken 余额
    assetStorage.users[account].balance = encodeAmount(balance - amount);
    // 累加到 reserves
    assetStorage.reserveBalance = encodeSmallAmount(
        assetStorage.reserveBalance + amount
    );

    emit RequestDonate(account, amount);
    // ❌ 缺失：checkLiquidity(account)
}

// ===== 对照：正常的 transfer 路径 =====
function transfer(address to, uint amount) external {
    ... // 余额变动
    checkLiquidity(msg.sender);  // ✅ 必须健康因子 >= 1
}

// ===== 以及 liquidate 路径 =====
function liquidate(address violator, ...) external {
    ... // 清算人以折价获得抵押
    require(getHealthScore(violator) < 1e18, "not violating");
    ... // 允许极大折价（最多 ~20%）转移抵押给清算人
}
```

**错在哪一行**：`donateToReserves` 在修改了 `users[account].balance` 之后**没有调用 `checkLiquidity`**。这让攻击者可以把自己的抵押"捐掉"，使账户人为地从健康滑入违约状态——而 `liquidate` 只校验"当前违约"，并不问"为何违约"。

### 3.4 攻击步骤分解

以 DAI 池为例（tx `0xc310...111d`）：

1. **闪电贷**：从 Aave V2 借 30,000,000 DAI。
2. **两段式存入**：向 Euler 存入 20,000,000 DAI → 获得 eDAI。
3. **递归借贷**（Euler 允许自借）：借出 195,000,000 eDAI（mint），对应 dDAI 负债 200,000,000。反复再存再借，放大杠杆约 10 倍。
4. **偿还一部分**：用剩余 10,000,000 DAI 偿还部分债务，释放部分 dDAI，但保留巨量 eDAI。
5. **核心一击**：调用 `donateToReserves(0, 100,000,000 eDAI)`——把 1 亿 eDAI"捐"给协议 reserves。账户因此从健康（抵押 >> 负债）瞬间变为严重违约（抵押远小于负债）。**此处本应 revert，但缺少 `checkLiquidity`，交易成功。**
6. **自清算**：用另一个 subAccount（Euler 允许同一 EOA 下多个子账户互相清算）调用 `liquidate`，以接近 20% 的折价从违约账户获取抵押 eDAI 并承接极少负债。
7. **提现**：将清算所得的 eDAI 通过 `withdraw` 换回 38,900,000 DAI。
8. **归还闪电贷** 30,000,000 DAI，净利约 8,878,502 DAI。

同一套组合被复制到 WBTC / USDC / stETH 池，6 条 tx 内完成 $197M 抽取。

### 3.5 为何审计未发现

- `donateToReserves` 是"后引入"的单一函数，多数主审计完成于 2022-06 前，未覆盖该 PR。
- Certora 形式化验证规范里未针对"所有改变用户余额的写路径都必须尾随 `checkLiquidity`"这类不变量建模——这是一个**类不变量（class-level invariant）**，而不是函数局部不变量。
- Slither 的默认规则不会发现"函数缺一个调用"类型的 bug（需要自定义规则"若函数写入 `users[x].balance`，则必须调用 `checkLiquidity(x)`"）。
- Euler 事后请 Spearbit 牵头的安全联盟（含 OpenZeppelin、Trail of Bits、Hexens 等）进行全面复审。

## 4. 事后响应

- **紧急停摆**：攻击后 1.5 小时 Euler 关闭 `batchDispatch` 与所有借贷入口；合约 governor 调用 `setMarketFrozen`。
- **谈判与归还**：Euler 设立 $1M 赏金，后通过链上公告 + Mail3 + 直接 tx message 持续对话。攻击者在 3-17 至 4-04 期间分六批归还全部资金至治理多签。
- **赔付机制**：Euler 部署"Recovery Distributor"合约，按快照 pro-rata 向 eToken / dToken 持有者及集成协议退还；Angle、Balancer 等集成方分别与 Euler DAO 签署独立返还协议。
- **执法**：FBI 与英国 NCA 介入；Chainalysis 追踪 Tornado Cash 出入口。归因最终未公开身份，但攻击者自称"Jon"并道歉。
- **后续审计**：Spearbit、Hexens、OpenZeppelin、ChainSecurity 联合复查后 Euler v2 于 2024-03 重启。

## 5. 启发与教训

- **对开发者**：所有修改用户抵押/债务的函数必须**强制**尾随一次健康检查；可用修饰器 `modifier withLiquidityCheck(address account)` 强制装配。**新增函数必须重跑完整审计**，尤其是借贷/保证金类协议。
- **对审计方**：把"写路径-检查点匹配"上升为类不变量，在 Certora / Halmos 中写出如 `invariant all_writes_to_balance_followed_by_checkLiquidity`；对多子账户模型（Euler 的 subAccount）专门建模"自清算"攻击者。
- **对用户**：集成第三方借贷时必须评估对方的 upgrade 流程与 immutable 范围；不能假设"6 次审计 = 安全"。
- **对协议**：治理 timelock 应覆盖所有可添加新 external function 的升级；在函数选择器层面做 allowlist 可减轻新函数盲区。

## 6. 参考资料

- Euler 官方 post-mortem: <https://blog.euler.finance/upgradability-and-the-donateToReserves-function-a-post-mortem-of-the-euler-attack-14d9c7a1a9e>
- SlowMist 分析: <https://slowmist.medium.com/slowmist-analysis-of-the-euler-finance-attack-b29e5d4b47ed>
- rekt.news: <https://rekt.news/euler-rekt/>
- CertiK: <https://www.certik.com/resources/blog/2NgvoP47xUxs8l6C4bgpK4-euler-finance-incident-analysis>
- Chainalysis: <https://blog.chainalysis.com/reports/euler-finance-attack-2023/>
- BlockSec 交易回放: <https://phalcon.blocksec.com/tx/eth/0xc310a0affe2169d1f6feec1c63dbc7f7c62a887fa48795d327d4d2da2d6b111d>
- DeFiHackLabs PoC: <https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/src/test/2023-03/Euler_exp.sol>

---

*Last verified: 2026-04-23*
