---
title: Curve Pools Vyper 编译器重入锁失效（2023-07-30, ~$73M）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2023-07-30
loss_usd: 73000000
chain: Ethereum
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://hackmd.io/@vyperlang/HJ4o7MMn2
  - https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/
  - https://slowmist.medium.com/slowmist-analysis-of-the-curve-finance-attack-c2c0c6ae7b3f
  - https://rekt.news/curve-vyper-rekt/
  - https://twitter.com/CurveFinance/status/1685693202722848768
  - https://blog.openzeppelin.com/curve-finance-vulnerability-analysis
tx_hashes:
  - 0x2e7dc8b2fb7e25fd00ed9565dcda565c12d32a268ff4c10f51bb44c1ce1b9d93 (Ethereum, pETH/ETH 池)
  - 0xa84aa065ce61dbb1eb50ab6ae67fc31a9da50dd2c74eefd561661bfce2f1620c (Ethereum, msETH/ETH 池)
  - 0xb676d789bb8b66a08105c844a49c2bcffb400e5c1cfabd4bc30cca4bff3c9801 (Ethereum, alETH/ETH 池)
  - 0x5557e415e878bfe6bfcfe74ddad0ea35e802f95ec4c5aa2dc89b9f5a4c1d4f39 (Ethereum, CRV/ETH 池)
---

# Curve Finance Vyper 编译器重入锁失效

> **TL;DR**：2023-07-30，Curve Finance 多个使用原生 ETH 的 StableSwap/CryptoSwap 池被重入攻击，损失约 $73M。根因不在 Curve 合约，而在 Vyper 0.2.15 / 0.2.16 / 0.3.0 三个版本编译器：`@nonreentrant("lock")` 装饰器生成的字节码存在 bug，reentrancy lock 的 slot 会在重复使用同名 key 时互相覆盖 / 被初始化为非持久值，导致本应互斥的函数实际上不互斥。攻击者借 `remove_liquidity` 与 `remove_liquidity_imbalance` 之间的重入窗口掏空池子。

## 1. 事件背景

- **项目**：Curve Finance，稳定币与锚定资产 AMM 龙头，峰值 TVL ~$4.5B。受影响的均为"CryptoSwap-NG"或"Crypto v2"类含 ETH 资产的双币池（而非主力 3pool/stETH 池）。
- **受影响池**：
  - `pETH/ETH`（JPEG'd）：~$11.4M
  - `msETH/ETH`（Metronome）：~$3.4M
  - `alETH/ETH`（Alchemix）：~$22.6M
  - `CRV/ETH`（Curve 原生）：~$24.7M（加上后续连带）
  - 总计按当日价约 $73M。
- **Vyper 版本**：根据 Vyper 官方披露，以下版本受影响：`0.2.15`、`0.2.16`、`0.3.0`；`0.3.1+` 已修复。
- **时间轴**：
  - 2023-07-30 18:14 UTC：pETH/ETH 池被 MEV bot 首先攻击。
  - 2023-07-30 19:01 UTC：msETH/ETH 池。
  - 2023-07-30 19:12 UTC：alETH/ETH 池。
  - 2023-07-30 20:28 UTC：CRV/ETH 池。
  - 2023-07-30 23:00 UTC：Curve、Vyper、ChainSecurity、OpenZeppelin 联合披露事件；Vyper 团队公开 root cause。
- **发现过程**：MEV searcher（c0ffeebabe.eth）率先复现攻击并**front-run 白帽化** $5.4M alETH 返还 Curve。

## 2. 事件影响

- **直接损失**：$73M（涉及 4 个主池）。另有 Metronome、JPEG'd、Alchemix 协议 TVL 间接损失。
- **二次效应**：CRV 价格当日跌 ~10%。Curve 创始人 Michael Egorov 巨额 CRV 质押借贷头寸面临清算风险，Aave/Fraxlend/Inverse 上其头寸被市场盯盘；Egorov 后紧急 OTC 卖出 CRV 给 Tron、Mechanism Capital 等以避免级联清算。
- **资金去向**：约 $52.3M 通过白帽（c0ffeebabe）与攻击者谈判归还；其余由 Curve 追讨团队与链上谈判回收一部分，最终净损约 $24M。
- **连带披露**：所有使用同版本 Vyper 部署的其他协议（含 EraLend、Conic Finance）被紧急排查。

## 3. 技术根因（代码级分析，本章为核心）

### 3.1 漏洞分类
**编译器缺陷（Compiler Bug）**——高级语言中的互斥原语在特定版本下未被正确编译，产生看似正常但不起作用的字节码。

### 3.2 受损模块

- Vyper 编译器源码：<https://github.com/vyperlang/vyper>
- 出问题的语义：装饰器 `@nonreentrant("<key>")` 的实现（`vyper/semantics/analysis/base.py` 与代码生成 `vyper/codegen/function_definitions/common.py`，版本 0.2.15/0.2.16/0.3.0）。
- 受影响合约：Curve StableSwap / CryptoSwap 池，例 alETH/ETH 池 `0xC4C319E2D4d66CcA4464C0c2B32c9Bd23ebe784e`。

### 3.3 Vyper 重入锁工作原理 & 错在哪里

预期语义：

```python
# 期望：所有带同样 key 的函数共享一个 storage slot，
# 进入任一函数即置 lock，退出清除；若二次进入则 revert。

@external
@nonreentrant('lock')
def remove_liquidity(_amount: uint256, _min: uint256[2]) -> uint256[2]:
    ...
    raw_call(receiver, b"", value=eth_amount)   # ← 回调点
    ...

@external
@nonreentrant('lock')
def remove_liquidity_imbalance(_amounts: uint256[2], _max: uint256) -> uint256:
    ...
```

**实际 0.2.15/0.2.16/0.3.0 编译结果（简化）**：

```text
# 不同函数虽然 key 同为 'lock'，但编译器为每个装饰实例分配了
# 彼此独立的 reentrancy slot（另一类触发：slot 计算使用了不稳定
# 的符号索引，在跨升级 / 跨函数签名时会错位）。
# 结果：从 remove_liquidity 回调进入 remove_liquidity_imbalance
# 时，后者看到的 lock slot 仍为 0 → 允许进入 → 重入成立。
```

修复后（0.3.1+）：所有同 key 的装饰器共享同一个运行期 slot，且 slot 索引采用稳定的 deterministic allocator（见 Vyper issue #3390 与 commit）。

### 3.4 攻击步骤分解（以 alETH/ETH 池为例 `tx 0xb676...9801`）

1. **闪电贷**：攻击者从 Balancer 借 ETH 若干（具体约 3,000 ETH）。
2. **添加流动性**：向 alETH/ETH 池 `add_liquidity`，获取大量 LP 代币。
3. **触发 `remove_liquidity`**：请求移除全部 LP。合约开始向 receiver（攻击合约）用 `raw_call` 发送原生 ETH，并在此之前**已经更新了 LP balance，但 AMM 虚拟价格尚未最终结算**。
4. **重入 `remove_liquidity_imbalance`**：受益于 Vyper 编译 bug，`@nonreentrant('lock')` 未生效，攻击合约在 `receive()` 回调中再次调用池子的 `remove_liquidity_imbalance`。此时池子的内部储备与 LP 计算出现不一致（虚拟价格被低估），合约按旧比率支付出更多 alETH。
5. **交易回到第 3 步**，继续完成 `remove_liquidity` 的剩余逻辑，再派发一次 ETH。
6. **卖出 alETH** 获 ETH，偿还闪电贷，净赚 4,820 ETH（~$11.5M，其中 c0ffeebabe 白帽抢先回收 $5.4M）。

### 3.5 为何审计未发现

- 所有审计方（Trail of Bits、ChainSecurity、MixBytes）对 Curve 池本身做过多轮审计，审计范围**假定编译器可信**。
- 编译器自身的 CI/fuzz 测试里没有"同一合约多个 `@nonreentrant(key)` + 原生 ETH raw_call" 这一具体场景。
- 类似模式 OpenZeppelin 的 `nonReentrant` modifier（Solidity）由开发者可见的 storage 变量支持，排查直观；Vyper 装饰器属于语言级"黑盒"，开发者难自测。

## 4. 事后响应

- Vyper 在事件当日发布修复版本 `0.3.7` 并强制要求升级。
- Curve 冻结受影响池；白帽 c0ffeebabe 主动归还 2,879 ETH（alETH 池 ~$5.4M）。
- 通过链上谈判与赏金，总归还约 73% 资产。
- Egorov 通过 OTC 卖 CRV、降低杠杆化解个人清算风险，但此后引发 CRV/stETH/crvUSD 整体"创始人风险"讨论。
- ChainSecurity 与 OpenZeppelin 分别发布编译器漏洞分析文章。
- Vyper 团队补充 fuzz：Hevm + foundry-vyper 对装饰器语义专项测试。

## 5. 启发与教训

- **对开发者**：永远不要把互斥原语当"免费中间件"使用；在高价值合约中可在 Solidity/Vyper 层手动实现 `lock` storage 并显式检查，降低对编译器的信任面。
- **对审计方**：扩展 scope 到编译器 + 工具链；为同类"language-primitive correctness"引入 differential fuzzing。
- **对协议**：上线前对"新版本编译器 + 大额资金"路径增加红队阶段；关键业务逻辑对照不同编译器版本跑 invariant tests。
- **对生态**：鼓励 Vyper/Solidity 编译器发布 SBOM（软件物料清单）和确定性 bytecode diff，协议方可持续自验证。

## 6. 参考资料

- Vyper 官方披露: <https://hackmd.io/@vyperlang/HJ4o7MMn2>
- ChainSecurity 分析: <https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/>
- OpenZeppelin 分析: <https://blog.openzeppelin.com/curve-finance-vulnerability-analysis>
- SlowMist 分析: <https://slowmist.medium.com/slowmist-analysis-of-the-curve-finance-attack-c2c0c6ae7b3f>
- rekt.news: <https://rekt.news/curve-vyper-rekt/>
- Curve 官方 Twitter: <https://twitter.com/CurveFinance/status/1685693202722848768>
- DeFiHackLabs PoC: <https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/src/test/2023-07/Curve_exp.sol>

---

*Last verified: 2026-04-23*
