---
title: Hyperbridge（2026-04-13，~$2.5M，Multi-chain）
module: 08-security-incidents
category: Bridge | Protocol-Bug
date: 2026-04-13
loss_usd: 2500000
chain: Ethereum, Base, BNB Chain, Arbitrum
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/hyperbridge-rekt
  - https://hacked.slowmist.io
  - https://rekt.news/leaderboard/
tx_hashes:
  - 0x240aeb9a8b2aabf64ed8e1e480d3e7be140cf530dc1e5606cb16671029401109 (Ethereum — 10 亿 DOT mint)
  - 0xb28ab9526e1538bdb7a26ec8485d055f9e417620c72a2f4de0f42234b5f8ac09 (Ethereum — 9990 亿 ARGN mint)
  - 0xeff151ef58d57d6523874a7b97344fcd1ce3c7c6880cfc26a93da17f82062d59 (Ethereum — 前置 245 ETH 攻击)
---

# Hyperbridge（2026-04-13，~$2.5M）

> **TL;DR**：2026-04-13 03:55 UTC，Polkadot ↔ EVM 跨链桥 **Hyperbridge** 被利用**自研 Merkle Mountain Range (MMR) 证明验证库缺失边界检查** 漏洞——攻击者把 `leaf_index` 参数设为 **1 而非 0**，绕过了早期退出逻辑，使 `CalculateRoot()` 函数直接返回"数组元素本身"作为 root → 伪造 proof 通过验证 → 无中生有铸造 **10 亿 DOT** + **~9990 亿 ARGN** + MANTA/CERE 等代币，夺取桥接 DOT 合约的 admin。损失 **$2.5M**（远低于潜在规模，因大部分铸造代币没有足够二级流动性换成实值）。资金经 Tornado Cash 混合 ~$269k。本事件是 2026 年首起**"自写密码学库 + 缺测试边界"** 的典型案例。

## 1. 事件背景

### 1.1 Hyperbridge 是什么

[Hyperbridge](https://hyperbridge.network) 是 Polkadot / Substrate 生态与 EVM 生态间的跨链消息与资产桥，2024 年由 [Polytope Labs](https://polytope.technology) 推出。通过 MMR（Merkle Mountain Range，Substrate 生态常用 append-only 树结构）在 Substrate 链上累积状态根，并在 EVM 链上通过**自研 Solidity 验证库**验证 MMR proof → 支持 DOT、ARGN（Argon Network）、MANTA、CERE 等 Polkadot 系代币的跨链转移。

### 1.2 攻击时间轴

| 时间 (UTC) | 事件 |
|------|------|
| 2024-06 | SR Labs 审计报告已明确标注"自研 MMR 库需专业审查"，未采纳 |
| 2025-01 | 社区安全顾问在 grandpa crate 中发现类似 proof verification bug，报告未引起重视 |
| 2026-04-13 ~03:55 UTC | 攻击首次执行，通过前置 245 ETH 铸造资金准备 |
| 2026-04-13 ~03:55 UTC 后 | 铸造 10 亿 DOT（以太坊上的 0x8d01...0b8 桥接代币） |
| 同批次 | 铸造 ~9990 亿 ARGN，MANTA / CERE 也被波及 |
| 2026-04-13 | 攻击者将 DOT 合约 admin 转移到自己控制地址 |
| 2026-04-13 上午 | Hyperbridge 团队发现并暂停合约；各链 DEX 对新铸造代币启用风控 |
| 2026-04-13 ~中段 | 攻击者通过 DEX 能套现的部分 swap 到 ETH，**估值 $2.5M** 洗出；剩余代币因流动性不足无法有效变现 |
| 2026-04-14 | 攻击者通过 Tornado Cash 混合约 $269k |

### 1.3 事件规模说明

**账面铸造量极大（10 亿 DOT + 近万亿 ARGN），但实际获利仅 ~$2.5M**：
- Hyperbridge 桥接的 DOT 是 wrapped DOT，与真实 DOT 不互换；二级流动性仅数百万美元规模 → 短时抛售快速打穿流动性。
- ARGN 的 EVM 流动性更薄弱，9990 亿几乎无法变现。
- 初始估值曾报告为 $237k，经完整盘点修正至 $2.5M。

虽损失额落在 Tier-3 范围，但因**攻击模式新颖（自研密码学库边界漏洞）+ 对 Polkadot/EVM 跨链桥模型有系统性警示**，归类为 **Tier-2**。

## 2. 事件影响

### 2.1 直接损失

- 实际变现 **~$2.5M**（最初报告 $237k → 修正后 $2.5M）
- 桥接 DOT 合约 admin 被夺取
- Ethereum / Base / BNB Chain / Arbitrum 四链多个资产池被清空（小额合计）

### 2.2 连带影响

- Hyperbridge 代币（如有）及 Polytope Labs 代币生态受信任打击
- **Substrate/EVM 跨链桥项目**（Snowfork、XCM 桥等）集体审查自己的 MMR 实现
- 强化"**自研密码学库审计优先级**"行业共识

### 2.3 资金流向

```
[Forged proof → mint 10亿 DOT / 9990亿 ARGN on 4 chains]
   ↓ DEX swap（绝大部分因流动性不足无法套现）
[small portion → ETH / USDT]
   ↓ Tornado Cash mix
[~$269k consolidated]
```

**攻击者地址**：
- Primary: `0xc513e4f5d7a93a1dd5b7c4d9f6cc2f52d2f1f8e7`
- Secondary: `0xc0564bba9ba5a9be95ae866429f936012e1bf143`

**受损合约**：
- HandlerV1: `0x6C84eDd2A018b1fe2Fc93a56066B5C60dA4E6D64`
- TokenGateway: `0xFd413e3AFe560182C4471F4d143A96d3e259B6dE`
- Hijacked DOT: `0x8d010bf9c26881788b4e6bf5fd1bdc358c8f90b8`

## 3. 技术根因

### 3.1 漏洞本质：`leaf_index` 缺边界检查

自研 MMR 证明验证库中的 `CalculateRoot()` 函数用于：
1. 接收 `leaf`（要验证的叶子）
2. 接收 `leaf_index`（叶子在 MMR 中的索引）
3. 接收 `proof[]`（兄弟节点路径）
4. 计算并返回 MMR root

**关键漏洞**：当 `leaf_index == 0` 时有一个"早期退出"逻辑（即树只有一个叶子时直接返回 `leaf`）。当 `leaf_index == 1` 时，按理应进入正常路径：哈希当前叶子与 `proof[0]` 得到父节点，并继续迭代。

**但实现里错误地把 `leaf_index == 1` 当作"proof[0] 即为 root"的短路情况**——函数直接返回 `proof[0]`，**完全忽略了 leaf 本身**。

伪代码（重建自 rekt.news 描述；具体路径见 GitHub）：

```solidity
// 重建：简化示意
function CalculateRoot(
    bytes32 leaf,
    uint256 leaf_index,
    bytes32[] memory proof
) public pure returns (bytes32) {
    if (leaf_index == 0) {
        // 正常的单叶子早期退出
        return leaf;
    }

    // ❌ BUG：leaf_index == 1 时不应直接返回 proof[0]
    //      应进入 for 循环用 leaf + proof[0] 计算上层哈希
    //      当前实现等价于无条件返回 proof[0]，忽略 leaf
    return proof[0];

    // ……迭代哈希 leaf 与 proof[i] 的正常代码未被执行
}
```

**攻击向量**：
1. 攻击者构造 `leaf = bytes32(fake_message_hash)`，`leaf_index = 1`，`proof = [real_mmr_root]`
2. 调用验证 → 函数直接返回 `real_mmr_root`（因为 `proof[0]` 就是真实 root）
3. 验证通过 → **任意伪造消息都被视为"已被 Polkadot 链确认"**
4. 桥合约执行 forged payload → 铸造巨量代币 + 替换 admin

### 3.2 为何测试未发现

- 单元测试覆盖了 `leaf_index == 0` 和"多叶子"典型情况，**唯独没覆盖 `leaf_index == 1` 这个边界**。
- 缺少 **property-based / fuzz 测试**（如果用了 `forge fuzz` 对 CalculateRoot 做不变量"相同叶子不同 index 应产生不同 root"等断言，会立即触发）。
- **形式化验证未应用于 MMR 验证库**，而这本是 Substrate 社区推崇的做法。

### 3.3 为何审计未发现

- **2024-06 SR Labs 审计报告已明确标注："自研 MMR 库需专业审查，建议用 Substrate 官方已经形式化验证过的实现"**——未被采纳。
- **2025-01 社区安全顾问**在 grandpa crate 发现类似 proof verification bug（重放攻击面），通告未在 Hyperbridge 团队得到重视。
- 最终审计版本未做专项 MMR 边界 fuzz。

## 4. 事后响应

### 4.1 项目方

- **2026-04-13**：暂停 Hyperbridge 桥接，停止 mint / redeem
- **2026-04-13 当晚**：发布初步 post-mortem，承认 MMR 库边界漏洞；感谢 SR Labs 先前警告并公开致歉
- **2026-04-14+**：将 MMR 验证库替换为 Substrate 官方 `pallet-mmr` 对应的 Solidity port（经形式化验证）
- 准备赔付方案：协议 Treasury + Polytope Labs 自有资金覆盖真实变现损失

### 4.2 归因

- 无正式 APT 归因；链上 TTP 更像**独立黑客（white-gray hat 型）**，与 Lazarus 模式不一致。
- 攻击者保留 1 个地址未完全洗出，观察者猜测可能是"攻击示范 + 赏金博弈"——事件发酵后项目方公开邀请攻击者进入白帽通道（事发当日）。

### 4.3 行业连锁

- 所有使用自研密码学验证库的跨链桥（尤其 Substrate ↔ EVM 路径）开始自查。
- **forge fuzz / echidna 不变量测试** 成为审计标准项。
- **"自研密码学库应用"在审计报告中被强制标红**。

## 5. 启发与教训

### 5.1 对开发者

- **永远不要自研密码学验证库**。Substrate / OpenZeppelin / Arkworks 等社区经过形式化验证的库是首选。
- 所有"早期退出/短路"分支都必须有对应的**负向测试**（证明"这个短路确实只在预期条件下触发"）。
- 边界索引（0 / 1 / n-1 / n）是 fuzz 测试**必测的 4 个值**。

### 5.2 对审计方

- 审计报告的"建议不采纳"条目应**强制跟踪**，不能一次警告后就归档。
- 对自研密码学代码，普通人工审计不够，必须要求**形式化验证（K framework / Coq / Isabelle）**。
- **fuzz 不变量测试**应作为审计 checklist 的必选项。

### 5.3 对项目方

- **"审计过"≠"安全"**：SR Labs 警告在前却未执行是本事件真正的失误。
- 跨链桥的 admin 夺取向量应增加 **timelock + 白名单地址集合**，防止 admin 被任意地址接管。

## 6. 参考资料

- [rekt.news Hyperbridge Rekt](https://rekt.news/hyperbridge-rekt) — 完整技术复盘
- [SlowMist Hacked Database](https://hacked.slowmist.io) — 事件归档与链上证据
- [SR Labs 2024-06 Hyperbridge audit report](https://www.srlabs.de) — 预警出处（待归档）
- [Polytope Labs / Hyperbridge 官方 post-mortem](https://hyperbridge.network) — 待完整归档
- [Substrate pallet-mmr 参考实现](https://github.com/paritytech/polkadot-sdk)

---

**本条目为 DRAFT**：损失金额经 rekt.news 修正至 $2.5M；伪代码为基于 rekt.news 描述的重建示意，非原始源码——官方完整 post-mortem 发布后需对照真实代码与 commit hash 更新。

*Last verified: 2026-04-23*
