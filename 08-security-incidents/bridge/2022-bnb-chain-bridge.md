---
title: BNB Chain Cross-Chain Bridge (Token Hub) 伪造 IAVL Proof 事件（2022-10-06, ~$570M 名义 / ~$100M 逃出）
module: 08-security-incidents
category: Bridge
date: 2022-10-06
loss_usd: 570000000
chain: [BNB Beacon Chain, BNB Smart Chain]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.bnbchain.org/en/blog/bnb-chain-ecosystem-update/
  - https://halborn.com/explained-the-bnb-bridge-hack-october-2022/
  - https://rekt.news/bnb-bridge-rekt/
  - https://www.certik.com/resources/blog/bnb-cross-chain-bridge-exploit
  - https://twitter.com/samczsun/status/1578392113282158592
tx_hashes:
  - BSC Block 21955968, 21955970（含 2 次 2,000,000 BNB 铸造的 handlePackage 调用）
  - 攻击者 BSC 地址: 0x489A8756C18C0b8B24EC2a2b9FF3D4d447F79BEc
  - Token Hub precompile (BSC): 0x0000000000000000000000000000000000001004
---

# BNB Chain Cross-Chain Bridge（Token Hub）伪造 IAVL Proof 事件

> **TL;DR**：2022-10-06 UTC 18:26 左右，BNB Beacon Chain ↔ BNB Smart Chain 的官方桥 **BSC Token Hub** 被利用：攻击者向 BSC 上的跨链 precompile（合约地址 `0x0000...1004`）提交**两笔伪造的 IAVL Merkle proof**，使合约相信 Beacon Chain 已锁定 2,000,000 BNB 两次，BSC 端铸造 **总计 2,000,000 BNB**（约 **$570M** 名义价值）至攻击者账户。其中攻击者作为跨链 DeFi 用户出逃约 **$100M–$137M**（数字因追踪口径而异），其余被社区集体 freeze；BNB Chain 罕见地协调 44 个 validator **紧急硬停链** 约 8 小时修复。根因是 BSC 使用的 Cosmos IAVL 库 `cosmos/iavl` 在 RangeProof 校验中可以通过构造空 `leaf / inner` 序列通过验证，允许伪造。

## 1. 事件背景

### 1.1 BNB Chain 双链架构

BNB Chain 由 BNB Beacon Chain（原 Binance Chain，Cosmos 技术栈，21 validator）与 BNB Smart Chain（BSC，EVM 兼容，21 validator PoSA）构成。两条链间通过**内置跨链通信**：在 BSC 侧部署系统 precompile `0x0000...1004`（CrossChain）、`0x0000...2002`（TokenHub）等；Beacon Chain 向 BSC 发包时，BSC 收到 IAVL RangeProof（Cosmos 证明格式）并在 precompile 内验证，通过即铸造对应 token。

这是**链内置功能**，非独立桥项目，受 validator 集合与内置 precompile 实现保护。

### 1.2 时间轴

- **2022-10-06 UTC 18:26**（BSC block **21955968**）：攻击者第一次调用 CrossChain `handlePackage`，提交伪造 RangeProof，铸造 **1,000,000 BNB** 至地址 `0x489A8756...79BEc`。
- **UTC 18:38**（block **21955970**）：第二次同样方式铸造 **1,000,000 BNB**，累计 2,000,000 BNB。
- **UTC 19:00 左右**：攻击者把 BNB 拆分到多条链（Avalanche、Ethereum、Polygon、Fantom、Arbitrum），通过 Stargate、Multichain、Celer 等跨出 BSC。
- **UTC 19:30**：@samczsun 发推披露。
- **UTC 19:58**：BNB Chain 官方 Twitter 确认，协调 validator 集合 **硬停链**（halt block production）。
- **2022-10-07 UTC 03:40 左右**：BSC 打补丁后重启，validator 批量升级节点，精确回滚/冻结攻击者剩余资金。
- **2022-10-08**：BNB Chain 官方 blog 发布 post-mortem，Halborn、CertiK 对应发技术分析。

### 1.3 发现过程

异常的 2,000,000 BNB 铸造在 Etherscan-style 区块浏览器上肉眼可见（BSC TVL 突然跃增）。@samczsun、@PeckShieldAlert、@ZachXBT 并行告警。Binance CEO CZ 发推请求 validator 集体停链响应。

## 2. 事件影响

### 2.1 直接损失

- 名义 **2,000,000 BNB**，按当日价 $285 计约 **$570M**；
- 攻击者实际逃出约 **$100M–$137M**（主要流向 Ethereum、Avalanche、Fantom、Polygon、Arbitrum 上的 Stargate/Multichain 等跨链桥），其余 ~$430M 名义价值被 BSC 在暂停期间**冻结 / 回滚**，以及跨链桥在被通知后拒绝接受。

### 2.2 受害方

- 技术上 BSC 全网用户（增发摊薄），但通过硬停 + 精准冻结将实际损失限缩；
- 接收攻击者跨链的 Stargate、Multichain 等协议 LP 池短期受压；
- Venus Protocol：攻击者把部分 BNB 作抵押借出稳定币，Venus 短期坏账风险（后由 Binance 注资化解）。

### 2.3 连带影响

- BNB 价格 24h 跌 ~5%；
- **"硬停公链"**这一反去中心化动作引起重大争议，但社区多数认可这是快速止血手段；
- Cosmos IAVL 库漏洞影响辐射：Celo、Kava、Secret 等依赖 iavl 的链紧急审计、打补丁；
- 跨链桥信任模型进一步被质疑；后续 LayerZero / CCIP 强调 DVN 独立性。

### 2.4 资金去向

- 攻击者把资产分散到多条链；Tether、Circle、Binance 配合冻结部分 USDT/USDC；
- Chainalysis、Elliptic 跟踪，但**归因未公开**——无 FBI / Chainalysis 正式归因 Lazarus；攻击特征更像熟悉 Cosmos SDK 的高水平个人/小组。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**Merkle proof 伪造**（Cosmos IAVL RangeProof 校验逻辑存在空证明可通过）。

### 3.2 受损模块

- `github.com/cosmos/iavl`（RangeProof 验证）
- BSC 系统合约 `tool-config/crosschain_precompile.go` 调用的 IAVL verify 函数
- Precompile 地址 BSC：`0x0000000000000000000000000000000000001004`
- BSC 运行 go-ethereum fork（BEP-20 Token Hub 部分 Go 代码受影响）

### 3.3 关键逻辑（伪代码）

```go
// Cosmos iavl RangeProof.Verify(root []byte, keys, values [][]byte) error
// 用于验证 "从 Beacon Chain 某 block 提交的一组 key/value 确实在 AppHash = root 的状态里"

func (rp *RangeProof) Verify(root []byte) error {
    // step 1: 校验 leaves 构成的哈希链与 root 一致
    //         (实现细节：computeHash 遍历 Leaves + InnerNodes)
    got := rp.computeRootHash()
    if !bytes.Equal(got, root) { return errors.New("mismatch") }
    return nil
}

// 漏洞：当 rp.Leaves 和 rp.InnerNodes 为空且 rp.LeftPath 构造得当时，
// computeRootHash 返回的 got 可以被构造等于任意 root，因为实现里对空结构
// 的 hash 存在退化路径（类似 "ethereum patricia empty subtree" 绕过）。
```

**错在哪**：
- IAVL RangeProof 在 **空证明 / 特殊子路径** 情形下的完整性校验不严谨；攻击者可构造一个语法合法但**不证明任何 key/value 存在**的 proof，使 `Verify(root)` 返回 nil。
- BSC 的 crosschain precompile 信任 `iavl.Verify` 的结果，即把 Beacon Chain 上"BNB 已锁定"视为事实，调用 TokenHub mint。
- 具体漏洞由 Cosmos 团队在 iavl v0.19.x 分支修复（BNB Chain 使用的 fork 在事件时未跟进该补丁）。

### 3.4 攻击步骤分解

1. **漏洞研究**：攻击者识别 cosmos/iavl RangeProof Verify 逻辑的空证明绕过（可能基于已公开 issue / 审计备注）。
2. **构造伪证**：生成 2 笔伪造 proof，每笔声称 Beacon Chain 已锁定 1,000,000 BNB 并打入跨链 channel。
3. **提交 CrossChain 包**：以 BSC EOA `0x489A8756...` 调用 crosschain precompile `handlePackage`，附 2 个伪证。
4. **铸造 2,000,000 BNB**：precompile 内部调用 `iavl.Verify` 返回 nil，TokenHub `mint(recipient, amount)` 执行，攻击者获得 1M BNB ×2。
5. **分散跨链**：通过 Stargate、Celer、Multichain 等把部分 BNB 换成 BUSD、USDT、ETH、DAI 跨到 Ethereum、Avalanche、Arbitrum、Polygon、Fantom。部分 BNB 抵押到 Venus Protocol 借稳定币。
6. **被冻结**：BSC 硬停 + 社区 / 桥 / 稳定币冻结切断跨链通道，攻击者剩余 ~$430M 无法出逃。

### 3.5 为何审计未发现

- Cosmos IAVL 是开源库，多年被多家审计使用；该漏洞位于**空证明边界**，属"难以 fuzz 命中"的深处情况；
- BNB Chain 的 TokenHub precompile 审计在 2019-2020 年，彼时 iavl 版本尚未引入精确触发路径；
- BNB Chain 使用的 fork 未自动合并上游 iavl 修复。

## 4. 事后响应

### 4.1 项目方动作

- **硬停链**：协调 44 validator 集合暂停出块，8 小时内修复 precompile 并升级节点；
- 在 BSC 引入 `freeze` 功能（对特定地址冻结），禁用跨链 module 一段时间；
- 发布 v1.1.16 节点版本，合并 iavl 修复。

### 4.2 资产追回

- 冻结 ~$430M 名义价值；
- 攻击者逃出约 $100–137M，部分在 CEX、稳定币发行方冻结；
- Binance / BNB Chain DAO 投票后补贴 Venus 等协议损失。

### 4.3 法律 / 执法

- 未有公开执法归因；Binance 全球事件响应小组（SAFU）与多家链上调查机构协同；
- 攻击者身份至今**未公开**。

### 4.4 复查与审计

- Halborn 发布"Explained: The BNB Bridge Hack" 技术白皮书；
- Cosmos SDK 与 iavl 主仓库在其下游 v0.19.4 修正 RangeProof 空证明；
- 行业普遍审视 "Cosmos chain ↔ EVM chain 的内置桥" 模式，强调 proof 验证的 formal verification。

## 5. 启发与教训

### 5.1 对开发者

- 使用上游库必须**持续跟进 security release**；跨链桥 precompile 这样的关键组件应有 dedicated monitoring of upstream issues；
- Merkle / IAVL / patricia 证明验证逻辑必须测试 **空子树、单元素、边界长度** 等退化情形；
- Proof verification **必须返回"证明了什么"而非仅返回"无错"**：把 key/value 集合作为返回值，让调用方可以二次校验。

### 5.2 对审计方

- 对密码学 proof 库建议增加 **fuzz test + formal verification**；
- 审计 cross-chain module 时，应把上游依赖版本与已知 CVE 纳入范围。

### 5.3 对用户

- 大额使用内置桥的链需关注 validator 集合和快速修复能力；
- BSC 硬停表明中心化程度较高，是风险信号也是响应能力。

### 5.4 对协议

- 对冲灾难性漏洞：保留**紧急冻结 / 硬停**能力是双刃剑，但 2022 年底业界公认 BNB Chain 的响应**止损有效**；
- DAO 决策应预留快速通道覆盖跨链事件。

## 6. 参考资料

- BNB Chain 官方 blog：https://www.bnbchain.org/en/blog/bnb-chain-ecosystem-update/
- Halborn 技术分析：https://halborn.com/explained-the-bnb-bridge-hack-october-2022/
- rekt.news：https://rekt.news/bnb-bridge-rekt/
- CertiK 分析：https://www.certik.com/resources/blog/bnb-cross-chain-bridge-exploit
- @samczsun 实时 thread：https://twitter.com/samczsun/status/1578392113282158592
- Cosmos IAVL 漏洞修复：https://github.com/cosmos/iavl/releases
- 攻击者 BSC 地址：`0x489A8756C18C0b8B24EC2a2b9FF3D4d447F79BEc`

---

*Last verified: 2026-04-23*
