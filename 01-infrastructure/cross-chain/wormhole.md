---
title: Wormhole 跨链消息协议（Guardian / VAA / NTT）
module: 01-infrastructure/cross-chain
priority: P0
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-27
primary_sources:
  - https://docs.wormhole.com/
  - https://github.com/wormhole-foundation/wormhole
  - https://github.com/wormhole-foundation/native-token-transfers
  - https://wormhole.com/docs/learn/infrastructure/vaas/
---

# Wormhole 跨链消息协议

> **TL;DR**：Wormhole 是目前支持链最广的通用消息协议之一（35+ 条链，包括 Solana、SUI、Aptos、Algorand、Near 等非 EVM 原生链）。核心安全模型是一个由 19 个 **Guardian** 组成的外部 PoA 验证者集，对源链事件以 13/19 阈值签名生成 **VAA（Verified Action Approval）**，任何人可在目标链上提交 VAA 触发执行。在 2022 年 $325M Solana 事件后，Wormhole 新增了 Governor（限额保险熔断）、Accountant（全局余额守恒）、以及 NTT（Native Token Transfers，无 Wrapped 设计）等纵深防御机制。

## 1. 背景与动机

2020 年 Solana 生态崛起时，EVM 上的资金难以跨入，社区需要一个"通用消息总线"而非只做资产桥。Certus One 团队 2020 Q4 推出 Wormhole v1，初期只支持 Ethereum↔Solana 的资产桥；2021 年改为通用消息协议，Jump Crypto 接手后开发 v2。Wormhole 的设计哲学与 LayerZero 有显著分歧：

- **LayerZero**：最小化协议本身，把信任决策交给应用。
- **Wormhole**：协议自带一组共享 Guardian，所有应用共享同一 TCB（可信计算基）。这带来**部署成本几乎为零**（应用调用 Core Bridge 即可），但把全生态安全绑在 Guardian 上。

这种"共享信任"范式的代价在 2022-02-03 暴露：Solana 端 `verify_signatures` 签名验证绕过漏洞（attacker 提交伪造的 sysvar 账户），Wormhole Portal 被盗 120,000 wETH（约 $325M），Jump 于 24 小时内自掏资金补齐。事件后 Wormhole 的防御体系发生根本性重构。

## 2. 核心原理

### 2.1 形式化定义

Wormhole 的核心对象是 **Observation**：Guardian 看到源链上一个"带白名单 emitter"合约 emit 的事件。Guardian 集合 $G = \{g_1, \dots, g_{19}\}$ 按 superminority 阈值 $t = \lfloor 2/3 \cdot |G| \rfloor + 1 = 13$ 对 Observation 签名。

设 Observation = $(\text{emitterChain}, \text{emitter}, \text{sequence}, \text{payload}, \text{consistencyLevel}, \text{timestamp}, \text{nonce})$，签名后聚合为：

$$
\text{VAA} = \text{Observation} \,\|\, \{ (i, \sigma_i) : g_i \in S \}, \quad |S| \ge 13
$$

目标链上的 Core Bridge 合约 `parseAndVerifyVM(vaa)` 验证：
1. 13/19 签名有效；
2. Guardian 集合编号 `guardianSetIndex` 与目标链记录一致；
3. payload 由接收合约自行解释。

**安全不变式**：若 $\le 6$ 个 Guardian 被攻破，攻击者无法生成合法 VAA。

### 2.2 关键数据结构：VAA

```
version          uint8    // 目前 1
guardianSetIndex uint32   // Guardian 集合编号（每次 rotation +1）
len(signatures)  uint8    // 签名条数
signatures       []{
    guardianIndex uint8
    r, s          bytes32
    v             uint8
}
// ─── hash-covered body 开始 ───
timestamp        uint32
nonce            uint32
emitterChain     uint16   // Wormhole chain ID
emitterAddress   bytes32  // 源链 emitter（字节对齐）
sequence         uint64   // emitter 单调递增
consistencyLevel uint8    // 源链 finality 等级
payload          bytes    // 应用 payload
```

关键不变式：
- **sequence per-emitter 单调**：目标链维护已消费 `(emitterChain, emitter, sequence)` 集合防止重放。
- **guardianSetIndex 支持轮换**：Core Bridge 保存 N 个历史 Guardian 集合，允许提交旧集合签名的 VAA（通常有过期期）。
- **consistencyLevel**：Solana 的 `confirmed`/`finalized`、Ethereum 的 `latest`/`safe`/`finalized`，决定 Guardian 等多少块才 attest。

### 2.3 子机制拆解

**(1) Core Bridge（每链一份）**
每条支持链上部署一份 Core Bridge，EVM 实现为 `Implementation.sol`（UUPS 可升级）。它只做两件事：
- 源侧：`publishMessage(nonce, payload, consistencyLevel)` emit event；
- 目标侧：`parseAndVerifyVM(encodedVM)` 返回 `(VM, valid, reason)`。

**(2) Guardian 节点（链下）**
19 个机构分别运行 `wormhole/node`（Go）。每个节点连接所有 35+ 条链的 RPC，订阅 Core Bridge 事件，按 `consistencyLevel` 等待 finality，对 Observation 签名并通过 **Gossip 网络**（libp2p）广播；收到 13+ 签名后组装 VAA 并通过公共 API 公开。

**(3) Governor（限额熔断）**
2022 事件后新增的链下模块（`node/pkg/governor`）。按 `(chain, token)` 设置每日美元限额（例如 Ethereum WETH 每日 $1M）。超出限额的 VAA 被 Guardian 延迟签名 24 小时，期间治理可介入。这是**速率限制型熔断**。详见 §2.7。

**(4) Accountant（账本守恒）**
基于 Cosmos SDK 的独立区块链（Wormchain），由 Guardian 共同运行。所有 Token Bridge VAA 必须先提交到 Wormchain，Wormchain 维护每个 token 在每条链的 mint/burn 账本，任何"凭空铸造"（burn 不足却要 mint）的 VAA 被 reject。Accountant 是**全局守恒型熔断**。详见 §2.7。

**(5) Token Bridge / NTT**
- **Token Bridge（Portal）**：lock-and-mint 模式，源链锁原始 token，目标链铸造 Wrapped。
- **NTT（Native Token Transfers, 2024）**：无 Wrapped，项目方在每条链部署原生 token 合约，Wormhole 只做消息传递，`burn` on src / `mint` on dst。适合已经多链部署的协议（如 Lido wstETH、Ondo USDY）。

**(6) Guardian 治理**
Guardian 集合通过 "Governance VAA" 轮换：超过 13 个现任 Guardian 签署一条特殊 payload 即切换 `guardianSetIndex`。2023 年引入 Wormhole DAO（$W token）后，Guardian 变更也须通过链上治理提案。

### 2.4 参数与常量

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| Guardian 总数 | 19 | 可通过治理 VAA 调整（历史上曾 n=19 固定） |
| 签名阈值 | 13（2/3 + 1）| 硬编码 |
| consistencyLevel | enum | Ethereum: `finalized` 建议；Solana: `finalized` |
| Governor 每日限额 | per-token | 链下配置，热更新 |
| 最大 payload | 目标链实现决定 | EVM 受 calldata limit |

### 2.5 边界条件与失败模式

- **≥7 Guardian 被攻破**：协议共识失效，可伪造任意 VAA；Governor 可延缓但非阻止。
- **Gossip 网络分区**：VAA 可能延迟或无法组装；不影响资金安全。
- **源链重组深度 > consistencyLevel**：可能 attest 了被回滚的事件；这是 Wormhole 2022 Solana 事件之外另一类潜在问题。
- **目标链合约漏洞**：2022 事件本质；现有 Core Bridge 已修复 `verify_signatures`，但应用层仍可出错。
- **Accountant 的 BFT 假设**：Wormchain 本身是 PoS 链，若 Guardian 超过 1/3 离线会停链，VAA 无法被 Account。

### 2.6 图示

```mermaid
sequenceDiagram
    participant App as dApp (src chain)
    participant CoreSrc as Core Bridge (src)
    participant G1 as Guardian #1
    participant G2 as Guardian #2
    participant Gn as Guardian #19
    participant Accountant as Wormchain Accountant
    participant CoreDst as Core Bridge (dst)
    participant AppDst as dApp (dst chain)

    App->>CoreSrc: publishMessage(payload)
    CoreSrc-->>G1: emit LogMessagePublished
    CoreSrc-->>G2: emit ...
    CoreSrc-->>Gn: emit ...
    par wait finality & sign
        G1->>G1: sign Observation
        G2->>G2: sign
        Gn->>Gn: sign
    end
    G1->>Accountant: submit Observation (Token Bridge)
    Accountant->>Accountant: check global balance
    Accountant-->>G1: ack
    Note over G1,Gn: gossip signatures (libp2p)
    G1->>AppDst: (relayer) submit VAA
    AppDst->>CoreDst: parseAndVerifyVM(vaa)
    CoreDst-->>AppDst: valid → execute payload
```

### 2.7 Governor 与 Accountant 深入

2022 年 Solana 事件的根因是单点智能合约漏洞直接绕过 Guardian 共识铸造 120k wETH。事后 Wormhole 不再只依赖 "19 Guardian 一层签名"，而是在**签名生成环节**前后各加一层独立风控：链下的 **Governor** 负责速率限流，链上的 **Accountant** 负责全局守恒。两者并行运行，VAA 必须同时过关才会被最终发布。

#### 2.7.1 Governor（链下速率熔断）

Governor 代码在每个 Guardian 节点内部运行（`node/pkg/governor/`），**不是链上合约**。它在 Guardian 准备对 Observation 签名时切入，决定"立即签"、"延迟签"还是"进入人工审核队列"。覆盖范围目前是 **Token Bridge 转账**（`flow_cancel_tokens.go` 中定义的白名单 token）——不覆盖任意 `publishMessage` 消息。

**两维度限额**：

| 维度 | 参数 | 行为 |
| --- | --- | --- |
| per-(chain, token) 24h 总量 | `dailyLimit`（USD，例 Ethereum 上某 token ≈$50M/day） | 按 `(sourceChain, token)` 维度在滑动 24h 窗口累计 outbound notional；超出后续 VAA 进入 `pending`，累计回落到阈值以下后自动放行 |
| 单笔大额 | `bigTransactionSize`（例 Ethereum ≈$7M） | 单笔超阈值直接 `enqueued`，默认等待 24 小时由 Guardian 自动释放；期间可被治理拦截 |

**定价通路**：Guardian 配置中为每个 `(chain, token)` 预置 `floorPrice`（硬编码低位保护价），并订阅 Coingecko 实时报价，签名时使用 `max(spotPrice, floorPrice)`——这是针对"价格预言机被砸到 0 就能把攻击总额折算成 $0，绕过日限额"这一威胁的显式防御（见 `token_list.go`）。

**Flow cancel（2024 引入）**：回流方向的 inbound 转账可**部分冲销**该 (chain, token) 的日累计，避免一条合法的大额双向调仓被误拦。只对少量稳定币白名单启用，防止攻击者用伪造 inbound 洗掉 outbound 额度。

**人工通道**：通过 **Governor Action VAA**（特殊 payload，13/19 Guardian 签名）可提前释放某条被 enqueued 的 VAA，或把某 token/chain 直接冻结。因此 Governor 不是硬熔断，而是给治理层留出 24h 响应窗口的**缓冲带**。

**局限**：
1. 只在 Guardian 签名前生效——若攻击者直接伪造 VAA 绕过 Guardian 共识，Governor 形同虚设（这种场景由 Accountant 接管）。
2. 只覆盖 Token Bridge；NTT 用合约内 `NttManager` 自带的 inbound/outbound rate limiter（`RateLimiter.sol`），通用消息无 Governor。
3. 链下配置的热更新依赖 Governance VAA，19 节点观测不一致时会出现短暂的 "5 签了 / 8 没签"分裂，需要通过共识自然收敛。

核心路径：`node/pkg/governor/governor.go:ProcessMsgForTime` → 计算 `notionalValue` → 查 `chainEntry` 日累计 / `bigTransactionSize` → enqueue 或放行 → `CheckPending` 定时重扫释放。

#### 2.7.2 Accountant（链上全局守恒）

Accountant 部署在独立的 Cosmos SDK 链 **Wormchain**（对外品牌 Gateway）。Wormchain 的 validator set 与 Guardian set **一一对应**——19 个 Guardian 各运行一个 Tendermint validator，stake 名义存在但不是安全来源，安全完全来自 Guardian 集合本身。换句话说，Wormchain 是 Guardian 共享的一个**带共识的链下状态机**，用来记账。

**两套并列 Accountant**：

| 模块 | 上线 | 覆盖 | 代码 |
| --- | --- | --- | --- |
| Global Accountant | 2023 Q1 | Token Bridge (Portal) lock-and-mint | `wormchain/x/accountant` |
| NTT Accountant | 2024 Q2 | NTT burn-and-mint | 同模块，独立 namespace |

**守恒不变式**：对每个 origin asset `(originChain, originToken)`，Accountant 维护从该 origin 流向所有其他链的转账账本。Token Bridge 的不变式（lock-and-mint 语义）：

$$
\text{locked}(\text{origin}, \text{token}) \;=\; \sum_{c \ne \text{origin}} \text{minted}(\text{origin}, \text{token}, c)
$$

NTT 的不变式（burn-and-mint，无 home chain）：

$$
\sum_{c} \text{burned}(c, \text{token}) \;=\; \sum_{c} \text{minted}(c, \text{token})
$$

每笔 VAA 都被建模为账本上的**原子记账项**：`A→B` 的转账等价于 "A 上 locked/burned +amount, B 上 minted +amount"；目标链上的 `completeTransfer` 再被 observer 上报为 "B 上 minted 被消费，用户余额 +amount"。任何让某个 `minted[c]` 变负、或让 $\sum \text{minted} > \text{locked}$ 的记账项都违反不变式。

**双共识流程**：
1. Guardian 的链 watcher 观测源链 `LogMessagePublished`，产出 Observation；
2. Guardian 向 Wormchain 提交承载这些 Observation 的 Cosmos 消息 `MsgSubmitObservations`（对应 `x/wormhole` 的 handler `SubmitObservations`）；
3. Wormchain 状态机在 `DeliverTx` 中处理 `SubmitObservations` 这笔交易，并调用 `x/accountant` 的 `ProcessObservation` 校验守恒不变式：若违反直接 reject（`ErrInvariantViolation`），该 Observation 永远拿不到 Accountant 的 approve；
4. Wormchain 自身按 Tendermint BFT 出块，获得 Wormchain 侧超过 2/3 voting power 的 validator commit 签名后把 Observation 标记为 `approved`；
5. Guardian 监听到 Wormchain approval 后，才把自己的 ECDSA 签名发到主 P2P Gossip，参与组装目标链 VAA。

换句话说，攻击者即便攻破了某条目标链的 Core Bridge（能让那条链 `parseAndVerifyVM` 接受伪造 VAA），只要他不能同时伪造源链的 lock 事件，Wormchain 上的 `locked` 不会增加，Guardian 就不会为 `minted` 增量签名，伪 VAA 根本组装不出 13/19 签名——这是对 2022 Solana 事件类攻击的**结构性封堵**。

**可用性代价**：Wormchain 是 BFT 链，超过 1/3 Guardian 离线即停机，Token Bridge / NTT 消息全线阻塞（通用 `publishMessage` 不走 Accountant，不受影响）。Wormhole 显式选择**安全优先于可用性**。

**仍然防不住的攻击面**：
1. **源链 token 合约被 admin 无限铸造**：Accountant 只守恒 bridge 内账本，若源链 USDC 发行方自己增发，bridge 看到的 `locked` 会等比例放大，wrapped 侧照常 mint——需要上游信任。
2. **Wormchain 自身漏洞**：Cosmos SDK + 自定义模块构成新的攻击面，由 OtterSec 等多轮审计覆盖。
3. **Guardian 共谋伪造 Wormchain 状态**：仍然是 13/19 信任假设的延伸，没有比 Guardian 本身更强的假设。
4. **通用消息**：不经 Accountant；若应用自行实现 mint/burn 语义需自己做守恒。

核心代码：`wormchain/x/accountant/keeper/msg_server.go:SubmitObservations` → `handleObservation` → `CommitPendingTransfer`（approve）或返回 `ErrInvariantViolation`（reject）。合约侧的 `GlobalAccountant` wrapper 见 `ethereum/contracts/`。

#### 2.7.3 两层叠加的威胁模型

| 攻击类型 | Governor 能否拦截 | Accountant 能否拦截 |
| --- | --- | --- |
| 某目标链 Core Bridge 漏洞伪造 VAA（2022 型） | 否（VAA 不经 Guardian 签名） | **能**（伪造 mint 无对应 locked） |
| Guardian 私钥批量泄露（≥13） | 否 | 否（Guardian 也控制 Wormchain） |
| 单个 token 价格预言机被操纵 | **能**（floor price 保护日限额） | 部分（不直接用价格，但限额绕过仍有价值） |
| 合法但异常的大额转账 | **能**（24h 延迟 + 人工审核） | 否（守恒成立即放行） |
| 源链原始 token 合约恶意增发 | 否 | 否（上游信任问题） |
| 通用 `publishMessage` 攻击 | 否（不覆盖） | 否（不覆盖） |

两层是**互补而非冗余**：Governor 处理"签名前看到的异常流量模式"，Accountant 处理"签名前看到的账本不一致"。2022 事件的攻击路径（伪造 VAA 绕过 Guardian 共识）只有 Accountant 能封堵；而 Accountant 无法判断"一笔合法但超常规的大额转账是否值得 24h 观察期"——这是 Governor 的职责。

## 3. 架构剖析

### 3.1 分层视图

1. **应用层**：Token Bridge、NTT、CCTP integration、xAsset（Worm, W token）、第三方（Mayan、deBridge）
2. **Core Bridge 层**：每链 Implementation 合约
3. **Guardian 共识层**：19 节点 + Gossip + Wormchain Accountant
4. **观察层**：每 Guardian 独立运行的 watcher（每条链一个，如 `solana_watcher.go`、`evm_watcher.go`）
5. **Relayer 层**：Specialized Relayer / Standard Relayer（Wormhole 提供官方通用 relayer）
6. **客户端 SDK 层**：`@wormhole-foundation/sdk`（TS）、`wormhole-sdk-rs`（Rust）

### 3.2 核心模块清单

| 模块 | 路径（`wormhole-foundation/wormhole`，tag v2.23+） | 职责 | 可替换性 |
| --- | --- | --- | --- |
| Core Bridge (EVM) | `ethereum/contracts/Implementation.sol` | publishMessage / parseAndVerifyVM | UUPS 可升级 |
| Core Bridge (Solana) | `solana/bridge/program/src/lib.rs` | 同上 Solana 版 | program upgrade authority |
| Token Bridge | `ethereum/contracts/bridge/Bridge.sol` | lock-mint 资产桥 | 应用层，可替换 |
| NTT Manager | `native-token-transfers/evm/src/NttManager` | burn-mint 原生代币 | 项目独立部署 |
| Guardian node | `node/cmd/guardiand` | Observation / 签名 / Gossip | 19 机构独立运行 |
| Accountant | `wormchain/x/accountant` | 全局账本守恒 | Wormchain 模块 |
| Governor | `node/pkg/governor` | 每日限额熔断 | Guardian 链下 |
| Relayer (generic) | `relayer/generic_relayer` | 官方通用 relayer | 应用可自建 |

### 3.3 数据流 / 生命周期

以 **Token Bridge：Ethereum USDC → Solana** 为例：

1. **用户**：调用 `TokenBridge.transferTokens(token, amount, recipientChain, recipient, arbiterFee, nonce)`。合约把 USDC 转入 bridge 持有，调用 `Core.publishMessage(payload)` emit `LogMessagePublished(sender, sequence, payload, ...)`。
2. **Guardian watchers**：每个 Guardian 的 EVM watcher 订阅 `LogMessagePublished`，等待 `consistencyLevel=finalized`（Ethereum ≈13 min）。
3. **Observation 签名**：Guardian 对 `keccak256(body)` 用自己的 ECDSA key 签名，广播到 Gossip。
4. **Accountant**：因为是 Token Bridge VAA，必须先提交到 Wormchain，校验 `solana.wUSDC.mintedSupply + amount ≤ ethereum.USDC.lockedSupply`，通过后 Guardian 才完成签名。
5. **VAA 组装**：13 签名齐后，任一 Guardian 公开 VAA；第三方 Relayer（如官方 Generic Relayer）拉取。
6. **目标链**：Relayer 调用 Solana `token_bridge::complete_transfer(vaa)`，Solana program 校验签名、nonce、emitter；mint wUSDC 到 recipient。
7. **可观测性**：https://wormholescan.io 按 sequence 展示每条 VAA；Guardian 状态页展示每节点健康度。

### 3.4 客户端多样性 / 参考实现

- Guardian 节点：Go 实现唯一（`node/`），19 Guardian 运行同一代码库，**这是 Wormhole 的单点**。Jump Crypto 表示正在推动多实现，但截至 2026-Q1 尚未落地。
- 智能合约：EVM (Solidity)、Solana (Rust/Anchor)、Sui (Move)、Aptos (Move)、Algorand (TEAL)、Near (Rust)、CosmWasm 等，各自独立实现。
- SDK：`@wormhole-foundation/sdk`（TS，官方主推）、`wormhole-sdk-rs`、`@wormhole-foundation/sdk-solana` 等。

### 3.5 扩展 / 互操作接口

- **Specialized Relayer**：应用自运营的 relayer，针对性 gas 策略
- **Generic Relayer**：官方 `WormholeRelayer.sol`，支持 `sendPayloadToEvm`、gas 代付
- **Queries**：链下"预言机式"查询（从 Guardian 拉取任意 EVM slot），2024 新增
- **CCTP 集成**：Wormhole 包 Circle CCTP 做 USDC，以 VAA 作 attestation 包装
- **MultiGov / Hubble**：跨链治理框架

## 4. 关键代码 / 实现细节

参考 tag `v2.23.0`。

**Core Bridge（EVM）发送侧**：

```solidity
// ethereum/contracts/Implementation.sol:L35
function publishMessage(
    uint32 nonce,
    bytes memory payload,
    uint8 consistencyLevel
) public payable returns (uint64 sequence) {
    // 1. 读取 emitter 的 sequence 并自增
    sequence = useSequence(msg.sender);
    // 2. 收取 messageFee（可配置，目前 0 ETH）
    require(msg.value == messageFee(), "invalid fee");
    // 3. emit 事件供 Guardian 捕获
    emit LogMessagePublished(
        msg.sender, sequence, nonce, payload, consistencyLevel
    );
}
```

**VAA 验证（关键路径）**：

```solidity
// ethereum/contracts/Messages.sol:L35
function parseAndVerifyVM(bytes calldata encodedVM)
    public view returns (Structs.VM memory vm, bool valid, string memory reason)
{
    vm = parseVM(encodedVM);
    Structs.GuardianSet memory gs = getGuardianSet(vm.guardianSetIndex);
    require(gs.expirationTime == 0 || gs.expirationTime > block.timestamp, "gs expired");
    if (vm.signatures.length < (gs.keys.length * 2) / 3 + 1) {
        return (vm, false, "quorum not met");
    }
    (bool signaturesValid, string memory r) = verifySignatures(vm.hash, vm.signatures, gs);
    if (!signaturesValid) return (vm, false, r);
    return (vm, true, "");
}
```

**Guardian watcher（Go，EVM）**：

```go
// node/pkg/watchers/evm/watcher.go:~L450
func (w *Watcher) fetchAndProcessBlock(ctx context.Context, block *ethTypes.Header) {
    logs, err := w.ethConn.FilterLogs(ctx, ethereum.FilterQuery{
        BlockHash: &block.Hash,
        Addresses: []common.Address{w.contract},
        Topics:    [][]common.Hash{{LogMessagePublishedTopic}},
    })
    for _, log := range logs {
        msg := parseLogMessagePublished(log)
        if !w.hasReachedFinality(msg.ConsistencyLevel, block.Number) {
            w.pending[msg.TxHash] = msg
            continue
        }
        w.msgC <- msg
    }
}
```

> 省略：Accountant gRPC 交互、Gossip 拓扑、reobservation 协议。完整见 `node/` 目录。

## 5. 演进与版本对比

| 版本 | 时间 | 关键变化 |
| --- | --- | --- |
| v1 | 2020-10 | Certus One 发布，仅 ETH↔Solana |
| v2 | 2021 | 通用消息协议，引入 Core Bridge |
| 2022-02 事件后 | 2022 Q2 | Solana 合约漏洞修复；引入 Governor 链下限额 |
| Accountant 上线 | 2023 Q1 | Wormchain 全局账本守恒 |
| Generic Relayer | 2023 Q3 | 官方 `WormholeRelayer.sol` |
| NTT | 2024 Q1 | 原生代币跨链，无 Wrapped |
| Queries | 2024 Q3 | Guardian 作为通用读预言机 |
| $W token + DAO | 2024 Q2 | Guardian 治理权逐步上链 |

## 6. 实战示例

**发送一条简单消息（EVM → Solana）**

```solidity
interface IWormhole { function publishMessage(uint32, bytes calldata, uint8) external payable returns (uint64); }
contract Sender {
    IWormhole public wh = IWormhole(0x98f3c9e6E3fAce36bAAd05FE09d375Ef1464288B);
    function send(bytes calldata payload) external payable returns (uint64) {
        return wh.publishMessage{value: msg.value}(uint32(block.timestamp), payload, 1);
    }
}
```

拉取并提交 VAA（TS）：

```typescript
import { wormhole } from "@wormhole-foundation/sdk";
const wh = await wormhole("Mainnet", ["evm", "solana"]);
const eth = wh.getChain("Ethereum");
const [whm] = await eth.parseTransaction(txHash);
const vaa = await wh.getVaa(whm, "Uint8Array", 60_000);
```

预期：约 15–30 分钟（Ethereum finalized）后 `getVaa` 返回，Solana 端合约可消费。

## 7. 安全与已知攻击

- **2022-02-03 Solana 事件（$325M）**：`solana_bridge::verify_signatures` 允许 attacker 传入伪造 sysvar 账户绕过签名校验；Jump Crypto 24h 内补仓。根因：Solana CPI sysvar 验证逻辑误用。修复 commit：`wormhole@fb8bf64`。见 Certik 事后分析。
- **$W 合约风波（2024）**：$W token 启动初期被质疑 Guardian 中心化；DAO 治理逐步将 Guardian 轮换权上链。
- **供应链攻击面**：19 Guardian 均运行同一 Go 代码 → 单一 bug 可影响全集。2023-Q4 起推动形式化验证与 differential testing。
- **Rug-pull 风险**：Token Bridge 的 Wrapped tokens 若源链原始合约被升级/铸造，Wrapped 资产无法感知（Accountant 只管 bridge 内守恒）。
- **审计**：OtterSec、Certik、Coinspect、Neodyme 多轮。

## 8. 与同类方案对比

| 维度 | Wormhole | LayerZero | Axelar | CCIP | IBC |
| --- | --- | --- | --- | --- | --- |
| 信任模型 | 19 Guardian 13/19 | 应用配置 DVN | Tendermint PoS (~75 validators) | DON + RMN | Light Client |
| 非 EVM 深度 | 最强（Solana/Sui/Aptos/Algorand/TON） | 强 | 中 | 弱 | Cosmos 限定 |
| 熔断机制 | Governor + Accountant | 无协议级 | 无协议级 | RMN 可暂停 | 无 |
| 单点风险 | Guardian 共享 TCB | 可配置 | 验证者集 | DON 集中 | 无 |
| 费用 | 链上 gas（messageFee≈0） | DVN+Executor | 目标链 gas 代付 | 固定费 | Relayer 市场 |
| 代币 | W | ZRO | AXL | LINK | ATOM 无必要 |

## 9. 延伸阅读

- Docs：https://docs.wormhole.com/
- Whitepaper v2：https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers
- 2022 事件事后：https://wormholecrypto.medium.com/wormhole-incident-report-02-02-22-ad9b8f21eec6
- a16z "Cross-chain bridges design space"
- NTT docs：https://wormhole.com/products/native-token-transfers
- 中文：登链"Wormhole 被盗 3 亿美元事件分析"

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 守护者 | Guardian | 19 个外部验证节点之一 |
| 可验证动作批准 | VAA (Verified Action Approval) | Guardian 签名后的跨链消息凭证 |
| 发射器 | Emitter | 源链上调用 publishMessage 的合约地址 |
| 一致性等级 | consistencyLevel | Guardian 等待的 finality 级别 |
| 守恒器 | Accountant | Wormchain 上维护全局 token 守恒的模块 |
| 节流器 | Governor | 按 token-chain 设每日限额的熔断 |
| 原生代币跨链 | NTT | 无 Wrapped 的 burn-mint 标准 |
| 中继器 | Relayer | 把 VAA 提交到目标链的链下服务 |

---

*Last verified: 2026-04-27*
