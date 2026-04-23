---
title: Cetus Protocol (Sui) $223M 流动性数学溢出攻击（2025-05-22, ~$223M）
module: 08-security-incidents
category: DeFi | Protocol-Bug
date: 2025-05-22
loss_usd: 223000000
chain: Sui
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://x.com/CetusProtocol/status/1925605739471044799
  - https://slowmist.medium.com/slowmist-analysis-of-the-cetus-protocol-exploit-8a9c8c1c1d2e
  - https://x.com/Dedaub/status/1925702155870683254
  - https://rekt.news/cetus-rekt/
  - https://x.com/SuiNetwork/status/1925661077454643492
  - https://blog.sui.io/cetus-incident-response/
tx_hashes:
  - Sui 链上攻击交易序列：官方 Cetus 与 Sui Foundation 联合 post-mortem 已披露（Sui 浏览器 suivision.xyz / suiscan.xyz 可查）；首笔攻击 digest 未公开完整字符串前以"未公开"处置
---

# Cetus Protocol (Sui) $223M 数学溢出攻击

> **TL;DR**：2025-05-22，Sui 生态最大 DEX Cetus Protocol 旗下 CLMM（集中流动性做市）合约被攻击者利用"liquidity 计算中的整数溢出 / 检查缺陷"抽取约 $223M（含 SUI、USDC、WAL、BUCK、HAEDAL 等）。攻击者通过构造极大 tick-range 并用极小单位 token 添加 "伪天量流动性"，绕过溢出校验，再以几乎零成本换出池内全部资产。Sui Foundation 与验证者紧急协调，冻结约 $160M 于攻击者地址（未出链），Cetus 与 Sui 社区投票表决救援方案。

## 1. 事件背景

- **Cetus Protocol**：基于 Move 语言的 CLMM（Uniswap v3 式集中流动性）DEX，Sui 与 Aptos 双部署，事发时 Sui 链 TVL 约 $240M，占 Sui DEX 市场 >60%，日交易量 $50–100M。
- **被攻击模块**：`cetus-clmm` 中的 `add_liquidity` / `get_delta_a/b` 等流动性与 tick 计算函数（Move 源码，开源仓库 github.com/CetusProtocol/cetus-clmm-sui）。
- **时间轴**：
  - 2025-05-22 ~10:30 UTC（区块 Sui checkpoint，具体以官方披露为准）：首笔攻击交易打包。
  - 攻击持续约 1 小时，分多笔耗尽多个 CLMM 池。
  - 11:40 UTC 附近：Cetus Twitter 宣布"检测到可疑活动，暂停合约"。
  - 12:xx UTC：Sui Foundation 与验证者协调，通过节点拒绝列表（deny list）冻结攻击者 Sui 链上地址约 $160M。
  - 部分资金（~$60M）已在冻结前桥出 Sui 至以太坊。
  - 2025-05-23 起：Cetus 提交补丁、发布 post-mortem；Sui 社区就"救援资金处置"进行投票。

### 发现过程

- Dedaub、BlockSec、慢雾分别在 15–30 分钟内发告警；Sui 官方节点拒绝列表机制（原设计用于基础安全响应）首次大规模启用。

## 2. 事件影响

### 直接损失

- 累计约 $223M（官方公告口径）；组成包括 SUI、USDC、WAL（Walrus 代币）、HAEDAL、BUCK、LOFI、SCA、CETUS 等多种 Sui 生态代币。部分新上线小市值代币在攻击后价格下跌 50–90%。

### 受害方

- Cetus LP（含个人与协议金库）。
- Sui 生态多个 launch 代币因 Cetus 主池被抽空而瞬间失去定价基准。

### 连带影响

- SUI 代币 24h 跌约 15%，CETUS 跌 40%+；整个 Sui DeFi TVL 当日缩水约 40%。
- 启动"L1 层介入 DeFi 事件"的大讨论：Sui Foundation 与验证者的"冻结协调"被部分批评为中心化介入，另一部分则视为有效拯救。
- Aptos 版 Cetus 未受影响（部署相对独立）。

### 资金去向

- 约 $160M 被冻结在 Sui 链上攻击者地址（验证者 deny-list，不进入区块打包）。
- 约 $60M 已桥出到 Ethereum，主要通过 Wormhole，其中部分进入 Tornado Cash / 跨链桥后续追踪。
- 攻击者向 Cetus 白帽谈判，最终 Sui + Cetus 通过治理提案 "升级协议、重分配被冻结资金用于赔付 LP"。详细比例官方披露。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类

**Protocol-Bug（数学校验缺陷）**——CLMM 在计算 "给定流动性 Δ 与 tick 范围，需存入多少 token" 时，中间乘法溢出检查遗漏了一条分支。

### 3.2 受损模块

- 仓库：`github.com/CetusProtocol/cetus-clmm-sui`
- 文件（Move 模块）：涉及 `pool.move` 中的 `add_liquidity` 与 `math_u256` / `clmm_math` 相关 `get_delta_a` / `get_delta_b` 函数。具体 commit 与函数名以 Dedaub/SlowMist 最终报告为准。

### 3.3 漏洞机理（基于 Dedaub / SlowMist 公开分析整合）

核心公式（Uniswap v3 CLMM 几何）：

```
Δa = L · (√P_upper - √P_lower) / (√P_upper · √P_lower)
Δb = L · (√P_upper - √P_lower)
```

实现在 Move 里使用 `u128` / `u256` 定点数。漏洞点在于 `checked_shlw`（"checked shift-left wide"）在某个分支对最高位的判定写成了错误的比较，允许 `L` 极大时仍返回成功，导致 `Δa` 或 `Δb` 被向下舍入为 "1" 甚至 "0"。

**攻击者的操作**：

1. 选取一个流动性稀薄的 tick range，使 `√P_upper / √P_lower` 比值极端。
2. 调用 `add_liquidity` 传入一个超大 `liquidity_delta`，因 checked 失效，协议计算出"只需要 1 单位 token_a + 1 单位 token_b"就能铸造天量 LP。
3. 以这笔"天价 LP 头寸"调用 `remove_liquidity` 或直接 swap 换出池内全部真实资产。

伪代码（概念还原，不是实际字节）：

```move
// 错误版本（示意）
fun checked_shlw(n: u256): (u256, bool) {
    // 意图：若 n 左移 64 位会溢出，返回 (0, true)
    // 实际：比较位被写成 `>` 而非 `>=`，导致
    // 某个边界值 n 溢出但被判为 "未溢出"
    if (n > MASK) { (0, true) } else { (n << 64, false) }
}
```

当 `liquidity_delta` 取特定巨值时，`checked_shlw` 返回"未溢出"但实际高位已丢失，后续 `get_delta_a` / `get_delta_b` 给出近乎 0 的所需投入。

> **注**：上述代码仅为示意，真实函数名与签名以 Cetus 官方修复 commit 为准，参见 CetusProtocol/cetus-clmm-sui 修复 PR。

### 3.4 攻击步骤分解

1. 攻击者部署攻击合约（Move package）。
2. 借用少量 SUI 作为 gas + 种子。
3. 对多个池（SUI/USDC、WAL/SUI、HAEDAL/SUI 等）逐一：
   - 创建极端 tick 范围头寸；
   - 使用漏洞铸造天量 LP；
   - 在单笔 PTB（Sui Programmable Tx Block）内 swap / remove 全部池内真实资产。
4. 将产出代币归集到若干 Sui 主地址 → 经 Wormhole 桥 → Ethereum。

### 3.5 为何审计 / 测试未发现

- Cetus 经过 OtterSec、MoveBit 等审计；审计在 CLMM 上线时覆盖了主要路径，但 `checked_shlw` 的边界条件属于数值分析盲区，单元测试多覆盖"正常量级"，未覆盖"接近 u256 最大值"的极端输入。
- Move 的 `u256` 运算没有像 Solidity 0.8+ 那样"默认 checked"，需程序员显式写 checked 包装，易漏检。

## 4. 事后响应

- **Cetus**：紧急暂停；发布初步 post-mortem；提出补偿方案（以新代币 + 国库 + Sui 基金补贴）。
- **Sui Foundation**：协调验证者运行 deny list，冻结攻击者地址（约 $160M），召开社区投票表决是否将冻结资金返还 Cetus 用于赔付 LP。
- **审计复查**：OtterSec、MoveBit、Verichains 陆续发布独立分析；Cetus 重上线前通过多重重新审计 + 形式化验证（工具链披露于官方博客）。
- **跨链协作**：Wormhole、相关桥协助追踪跨链出金；部分资金被 Tether / Circle 冻结。
- **归因**：链上行为未指向已知组织，归因未公开。

## 5. 启发与教训

- **定点数 / 位运算一定要 fuzz**：不变量测试应枚举 `liquidity_delta ∈ [1, u256::MAX]`、`tick ∈ [MIN_TICK, MAX_TICK]` 的边界组合。
- **Move 语言的"checked"不自动**：开发者需显式引入并审计每个算术路径；学习 Uniswap v3 的 `FullMath.mulDiv` 精确乘除模式。
- **L1 主权救援是把双刃剑**：Sui 以主权干预能力冻结资金给生态用户带来现实上的保护，但也引出"抗审查性 / 去中心化"取舍的长期辩论。开发者与用户需明确所选 L1 的社会契约。
- **DEX 合约 Guard 化**：对 `add_liquidity` 铸造出的 LP 金额设置合理上限（例如相对池现有流动性 ≤ X 倍），可在数学层被绕过后仍具有"业务层"兜底。
- **监控告警**：对 LP mint 单笔 > 池 TVL 一定比例、或 swap 输出 > 池 TVL x% 的事件实时告警。

## 6. 参考资料

- Cetus Protocol 官方 Twitter、Medium post-mortem
- Sui Foundation 响应博客与社区投票页
- Dedaub、SlowMist、BlockSec 技术分析
- rekt.news — Cetus Rekt
- OtterSec、MoveBit 复查报告
- Wormhole 跨链追踪公告

---

**本条目源于 SlowMist / Dedaub / Cetus 2025-05 公开分析；Cetus 与 Sui 官方完整技术报告后续可能更新，请重审。**

未公开 / 待核实字段：
- 首笔攻击交易 Sui digest（完整字符串）
- 精确损失细分（各代币数量与 USD 快照）
- 被冻结资金最终治理处置比例（救援 vs. 永久冻结）
- 攻击者身份 / 归因

*Last verified: 2026-04-23*
