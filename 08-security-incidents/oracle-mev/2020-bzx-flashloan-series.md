---
title: bZx 闪电贷三连击（2020-02 & 2020-09, 合计 ~$9M）
module: 08-security-incidents
category: DeFi
date: 2020-02-15
loss_usd: 9000000
chain: [Ethereum]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://peckshield.medium.com/bzx-hack-full-disclosure-with-detailed-profit-analysis-e6b1fa9b18fc
  - https://rekt.news/bzx-rekt/
  - https://bzx.network/blog/postmortem-ethdenver
  - https://bzx.network/blog/prelminary-post-mortem
  - https://slowmist.medium.com/slowmist-analysis-of-the-bzx-hack-c0062a9e41d
tx_hashes:
  - 0xb5c8bd9430b6cc87a0e2fe110ece6bf527fa4f170a4bc8cd032f768fc5219838 (Ethereum, 2020-02-15 第一次)
  - 0x762881b07feb63c436dee38edd4ff1f7a74c33091e534af56c9f7d49b5ecac15 (Ethereum, 2020-02-18 第二次)
  - 0x0a29628bf38a9b4fff56b6a94fbe4114326d3aa82b19fba2ea3a6caf75f83cc5 (Ethereum, 2020-09-13 iToken)
---

# bZx 闪电贷三连击

> **TL;DR**：2020 年 2 月与 9 月，衍生品借贷协议 bZx（Fulcrum）先后被三次洗劫，总计约 $9M。前两次（2020-02-15 / 02-18）是 DeFi 史上首批**闪电贷 + 现货预言机操纵**组合攻击，奠定"flashloan → 操纵 Uniswap/Kyber spot → 在目标协议内套利/清算"的攻击范式；9 月一次为 iToken `transferFrom` 自转逻辑漏洞导致 iETH 账面凭空膨胀。

## 1. 事件背景

- **项目**：bZx 是早期杠杆交易 / 借贷协议，产品线 Fulcrum（利息代币 iToken）、Torque（固定利率借贷）。事件前 TVL 约 $15M。
- **时间轴**：
  - 2020-02-15：第一次闪贷攻击，利润 ~1,193 ETH（彼时约 $350K）。
  - 2020-02-18：第二次，利润 ~2,378 ETH（约 $630K）。
  - 2020-09-13：iToken `transferFrom` 漏洞，损失约 $8M。
- **发现过程**：02-15 事件由社区在 ETHDenver 现场被 bZx 团队注意到链上异常；02-18 事件由 PeckShield 链上监控先行发现；09-13 事件由 bZx 团队自检发现。

## 2. 事件影响

- **直接损失**：
  - 2020-02-15：1,193 ETH，约 **$350K**（按当日 ETH $290）。
  - 2020-02-18：2,378 ETH，约 **$630K**（当日 ETH $265）。
  - 2020-09-13：219,200 LINK + 4,502 ETH + 1,756,351 USDT + 1,412,048 USDC + 668,248 DAI，约 **$8M**。
- **受害方**：bZx 协议保险基金（iToken 持有人实际上承担了本金缩水）。
- **连带影响**：闪贷攻击正式进入公众视野，`flash loan` 一词成为 DeFi 风险话术的核心；此后半年内 Harvest、Balancer、Eminence、Value DeFi 等多起同类事件发生；DeFi 项目普遍被要求切换到 Chainlink TWAP 或加入时间加权预言机。
- **资金去向**：攻击者兑换为 ETH，部分流入 Tornado Cash，未被追回。

## 3. 技术根因（代码级分析）

### 3.1 2020-02-15：sUSD × Kyber 价格操纵

- **漏洞分类**：预言机（现货价格）+ 业务逻辑未检查 `shouldLiquidate`。
- **核心路径**：
  1. 闪贷 10,000 ETH（dYdX）。
  2. 用 5,637 ETH 在 Compound 借出 112 WBTC。
  3. 在 bZx Fulcrum 开 5× 杠杆 WBTC/ETH 头寸（`mintWithETH` → `marginTradeFromDeposit`），触发 bZx 自动去 Kyber / Uniswap 吃单。
  4. 由于当时 Kyber WBTC/ETH 深度极浅，bZx 一笔市价单把 WBTC 价格推高 ~3×。
  5. 把前面从 Compound 借出的 WBTC 按"被推高的价格"卖回 Uniswap / Kyber，套取 ETH 差价。
  6. 偿还闪贷，净利润 1,193 ETH。
- **关键代码问题**：bZx `marginTradeFromDeposit` 中调用 `KyberNetworkProxy.trade()` 时未检查滑点，也未与 TWAP 比对：

```solidity
// bZx Fulcrum，简化
function marginTradeFromDeposit(...) external {
    // 1) 按用户指定杠杆借出 loanToken
    uint borrowed = _borrow(...);
    // 2) 直接在 Kyber 市价成交 —— 无 maxSlippage，无 oracle 校验
    uint bought = KyberNetwork.trade(loanToken, collateralToken, borrowed);
    // 3) 立刻以"成交价"记账为头寸成本
    positions[user].collateralAmount = bought;
    // ⚠️ 如果上一笔市价单把池子打歪，这里的 collateralAmount 记的是畸形价
}
```

同时 bZx 的清算逻辑也依赖这个被操纵的现货价，使攻击者可避开 `shouldLiquidate` 判定：

```solidity
function shouldLiquidate(bytes32 loanOrderHash, address trader) public view returns (bool) {
    // 现货价来自 Kyber/Uniswap，单块内可被闪贷操纵
    uint currentMargin = getCurrentMarginAmount(...);
    return currentMargin < maintenanceMargin;
}
```

- **tx**：`0xb5c8bd9430b6cc87a0e2fe110ece6bf527fa4f170a4bc8cd032f768fc5219838`。

### 3.2 2020-02-18：sUSD / Synthetix + Kyber 反向操纵

- **路径**：
  1. 闪贷 7,500 ETH（bZx 自家）。
  2. 用 3,517 ETH 在 Synthetix 铸 sUSD（Synthetix 彼时允许任意用户按 ETH 抵押铸 sUSD）。
  3. 在 Kyber 四次买入 sUSD，把 Kyber 上 sUSD/ETH 价推高 ~2×。
  4. 用"被推高的 sUSD"作抵押在 bZx 借 6,796 ETH（bZx 价格源正是 Kyber）。
  5. 偿还 7,500 ETH 闪贷，净赚 2,378 ETH。
- **根因与 02-15 同构**：bZx 价格源单一依赖 Kyber 即时成交价，未经聚合或时间加权。

### 3.3 2020-09-13：iToken `transferFrom` 自转漏洞

- **漏洞分类**：状态更新顺序 / 自转账未校验。
- **受损函数**：`iToken.transferFrom(from, to, amount)`，当 `from == to` 时。
- **关键代码**：

```solidity
// bZx iToken._internalTransferFrom
function _internalTransferFrom(address _from, address _to, uint256 _value, ...) internal {
    // 先读取两端余额
    uint256 _balancesFrom = balances[_from];
    uint256 _balancesTo = balances[_to];

    // 计算新值
    uint256 _balancesFromNew = _balancesFrom.sub(_value);
    uint256 _balancesToNew   = _balancesTo.add(_value);

    // 分别写回 —— 若 _from == _to，后一次写入覆盖前一次，余额凭空增加 _value
    balances[_from] = _balancesFromNew;
    balances[_to]   = _balancesToNew;    // ← 此处以 add 为准，把之前 sub 的量抵消
}
```

攻击者反复调用 `transferFrom(self, self, amount)`，每次 iETH 余额翻倍上升，再通过 `burn()` 赎回底层 ETH。关键 tx `0x0a29628bf38a...`。

- **审计盲区**：bZx 此前接受 Peckshield、ZK Labs、Certik 等多次审计，但 iToken `transferFrom` 自转支路在单元测试中未覆盖；DeFi Saver 式快速迭代导致新旧代码不一致。

## 4. 事后响应

- **项目方**：02 月两次后 bZx 上线"最大价格偏离" checker，并接入 Chainlink。09 月事件后暂停合约，由 PeckShield 与 Certik 联合事后审计，修复 `_internalTransferFrom`（加入 `require(_from != _to)`）。
- **赔付**：bZx 保险基金 + iToken 利率补贴分摊，用户最终拿回约 100% 本金。
- **行业连锁**：Chainlink 采用率急升，MakerDAO OSM、Uniswap V2 TWAP 被广泛引用为替代方案；DeFi 审计 checklist 加入"单块价格可被闪贷操纵吗"。

## 5. 启发与教训

- **开发者**：
  - 价格源绝不可直接用单一 AMM 现货 `getReserves` / Kyber 瞬时成交价；必须 TWAP / 聚合 / Chainlink。
  - Checks-Effects-Interactions、自转安全（`from == to`）、滑点上限三件套必须进入 `transferFrom`、`swap`、`liquidate` 等关键路径。
- **审计方**：
  - Slither 新增 `price-oracle-manipulation` 启发；Trail of Bits `necessist` 生成单块闪贷测试。
  - 不变量 fuzz：`Σ balances == totalSupply` 自转场景必测。
- **协议**：治理 timelock、紧急暂停开关在此后成为 DeFi 标配。
- **用户**：任何强调"价格从 Uniswap/Kyber 实时获取"的协议都应警惕，尤其在深度浅的池子上。

## 6. 参考资料

- PeckShield：<https://peckshield.medium.com/bzx-hack-full-disclosure-with-detailed-profit-analysis-e6b1fa9b18fc>
- rekt.news：<https://rekt.news/bzx-rekt/>
- bZx 官方 post-mortem (Feb)：<https://bzx.network/blog/postmortem-ethdenver>
- bZx 官方 (Sep)：<https://bzx.network/blog/prelminary-post-mortem>
- SlowMist：<https://slowmist.medium.com/slowmist-analysis-of-the-bzx-hack-c0062a9e41d>
- Samczsun 技术分析：<https://samczsun.com/taking-undercollateralized-loans-for-fun-and-for-profit/>
- DeFiHackLabs：<https://github.com/SunWeb3Sec/DeFiHackLabs>（bZx 子目录 `forge test` 复现）
- 关键 tx：<https://etherscan.io/tx/0xb5c8bd9430b6cc87a0e2fe110ece6bf527fa4f170a4bc8cd032f768fc5219838>、<https://etherscan.io/tx/0x762881b07feb63c436dee38edd4ff1f7a74c33091e534af56c9f7d49b5ecac15>、<https://etherscan.io/tx/0x0a29628bf38a9b4fff56b6a94fbe4114326d3aa82b19fba2ea3a6caf75f83cc5>

---

*Last verified: 2026-04-23*
