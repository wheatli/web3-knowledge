---
title: Avalanche
module: 01-infrastructure/public-chains
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-25
primary_sources:
  - https://build.avax.network/docs
  - https://github.com/ava-labs/avalanchego
  - https://www.avalabs.org/whitepapers
  - https://build.avax.network/docs/quick-start/avalanche-consensus
  - https://github.com/ava-labs/avalanche-rs
---

# Avalanche

> **TL;DR**：Avalanche 是 Emin Gün Sirer（康奈尔大学教授）团队 2018 年提出、Ava Labs 2020-09 主网启动的 **新型 L1 公链**。核心创新是 **Avalanche 共识家族**（Snowflake / Snowball / Avalanche / Snowman）——一种 **亚稳定重复随机抽样（metastable repeated random subsampling）** 的新范式，介于经典 BFT 与 Nakamoto 之间：每个节点每轮随机抽取 $k$ 个节点询问"你偏好什么"，若 $\alpha$ 个回答一致就翻转偏好，连续 $\beta$ 轮不变即接受。以 $k=20, \alpha=14, \beta=20$ 的默认参数，网络能在 **亚秒级** 达成概率趋近 1 的确定终局。独特的 **三链架构**：**P-Chain**（平台链，管 validator / staking / subnet）、**X-Chain**（资产链，UTXO + Avalanche 共识，原生代币与发行）、**C-Chain**（合约链，EVM 兼容 + Snowman）。在 2024 Durango + Etna 升级后，过去的 Subnet 演进为可自治的 **Avalanche L1**，通过 **ICM（Interchain Messaging / Teleporter）** 与 **ACP-77 自定义验证者管理** 获得极大灵活性。据官方估计主网活跃 validator 超过 1700 名。

---

## 1. 背景与动机

Avalanche 诞生于 2018 年的匿名白皮书 "Snowflake to Avalanche"（后署名 "Team Rocket"，团队后来公开为 Cornell 博士生 Kevin Sekniqi、Stephen Buttolph 与 Maofan "Ted" Yin，由 Emin Gün Sirer 教授带领）。当时 PoW 面临速度瓶颈，经典 BFT 在节点数超过数百就因 $O(n^2)$ 通信爆炸；Team Rocket 的观察是：**共识不一定要让所有节点两两同步**——只要每个节点随机询问少量邻居，通过 **偏好传染** 即能在对数时间内趋同，这对应伽尔顿-瓦特森过程的相变现象。

三链架构源于"分工优化"：资产转账（UTXO）不需要全局排序，交给 Avalanche DAG 共识；EVM 合约需要线性排序，交给 Snowman（Avalanche 的线性化变体）；而质押、Subnet 管理需要全局权威，交给 P-Chain 专职维护。

关键里程碑：2018 发表论文 → 2020-09-21 主网 → 2021-08 Apricot 升级支持大量新功能 → 2022 Banff + C-Chain 性能优化 → 2023 Cortina + BLS → 2024-03 Durango（ICM / Teleporter）→ 2024-12 Etna + ACP-77（Subnet → L1 范式转变）。

## 2. 核心原理

### 2.1 形式化定义：亚稳定共识

令每个节点 $u$ 在时间 $t$ 有一个偏好值 $c_u(t) \in \{0, 1\}$（Snowflake 二值情况）。共识过程定义为：

1. 随机抽样：每轮 $u$ 均匀无重复抽样 $k$ 个节点 $Q$。
2. 询问：向 $Q$ 查询其当前偏好。
3. 阈值反应：若收到 $\ge \alpha$ 个相同的值 $v$ 且 $v \ne c_u$，则 $c_u \leftarrow v$ 并计数器归零；否则计数器 $+1$。
4. 终止：当计数器 $\ge \beta$ 即 **接受** $c_u$。

数学上，若网络中偏好 $v=1$ 的节点比例 $p > 1/2$，则期望 $\alpha/k$ 查询返回 1，以指数速度拉高所有节点的 $v=1$ 偏好；反之亦然。这是一个 **亚稳定平衡**：$p=0.5$ 是不稳定点，任意微小扰动都会使系统滑向 0 或 1。

四阶段协议增量：
- **Slush**：无计数器，纯多轮抽样。
- **Snowflake**：加计数器 $\beta$，提供"决定点"。
- **Snowball**：加 **置信度 $d(v)$**——累计投票次数，切换偏好时需要置信度差距大于当前，避免频繁翻转。
- **Avalanche**：DAG 化，每个事务引用多条历史事务，针对 UTXO 模型批量决定冲突集（conflict set）。
- **Snowman**：线性化 Avalanche（DAG 退化为链），适用于需要全序的智能合约（C-Chain）。

### 2.2 关键算法：Snowman 出块与最终性

Snowman 把 Avalanche DAG 约束为 **单亲链**：

```
Snowman::poll(block):
    sample k validators
    responses = query(sample, block.parent)
    if majority prefer_block in responses:
        confidence++
        if confidence >= beta2: accept
    else:
        confidence = 0
```

- $\beta_1$（virtuous commit）与 $\beta_2$（rogue commit）可不同：前者针对"无冲突块"较松，后者针对"有竞争分叉"较严。
- 错误概率 $\varepsilon$ 可由参数量化：$P(\text{safety violation}) \le \varepsilon \approx (1-\text{adv})^{k\beta}$；合理参数下 $\varepsilon < 10^{-9}$。

拜占庭容错上限：理论证明 **任意 <1/2 stake 恶意节点**都无法破坏 Snowman（相比 BFT 的 1/3）。但该上限以"绝大多数样本正确回答"为前提，需要假设网络 ≠ 持续分裂。

### 2.3 子机制拆解

1. **Subnet → L1 (ACP-77)**：Subnet 原本是"P-Chain 上的一个 validator 子集"，Etna 后演化为 **完全自治的 L1**：validator 集合由 L1 自己维护（支持 PoS、PoA、自定义规则），不再强制每个 subnet validator 同时验证 Primary Network。大幅降低运行成本。
2. **BLS 聚合签名**：Cortina 引入，每个 validator 注册 BLS 公钥，支持跨 subnet 的 **Warp Message** 验证——一个 subnet 的 validator 集合签名被任意其它 subnet 高效验证。
3. **Teleporter / ICM**：跨 L1 的异步消息传递协议，基于 Warp，Sender L1 签名 → Relayer 抓取 → Target L1 验证；无信任第三方。
4. **P-Chain staking & slashing**：AVAX stake 最少 2,000 用于 validator / 25 用于 delegator，锁仓 2 周–1 年。Avalanche 历史上 **不做 slash**（长期被社区诟病），仅扣奖励；2024 社区正讨论引入部分 slashing 条件。
5. **C-Chain EVM**：基于 `coreth`（go-ethereum fork），支持 EIP-1559、Shanghai、Cancun 等主流 EVM 特性；Gas 用 nAVAX 计价。
6. **Fuji 测试网 / Dev-testnet**：便捷本地部署。

### 2.4 参数与常量（默认）

| 参数 | 值 | 说明 |
| --- | --- | --- |
| $k$ (sample size) | 20 | 每轮抽样 validators |
| $\alpha$ (quorum) | 14 (注：最新官方文档给出 0.7·k) | 翻转偏好阈值 |
| $\beta_1$ | 15 | virtuous 提交阈值 |
| $\beta_2$ | 20 | rogue 提交阈值 |
| Block time (C-Chain) | ~2 s | 实测 |
| Finality | ~1–2 s | 概率终局 |
| 最小 Validator 质押 | 2000 AVAX | 可治理 |
| Total AVAX supply cap | 720,000,000 | 软顶 |
| Avg Validator uptime | ≥ 80% | 拿奖励前置 |

### 2.5 边界条件与失败模式

- **偏好平衡攻击**：攻击者构造 50/50 冲突，使亚稳定点长期不坍缩；参数选择旨在最小化这种窗口，实测影响微弱。
- **网络分区**：若节点抽样无法触达足够正确 validator，共识"暂停"但不 fork——这是 Avalanche 选择活性 / 安全性的折衷（倾向安全）。
- **Validator 集中化**：如果少数实体托管大量 validator，共识安全降至该实体的诚实性；Ava Labs 长期推动地理分散。
- **缺 slashing**：历史事件表明验证者行为不当成本较低，需通过 uptime 奖励限制。

### 2.6 图示

```mermaid
graph TD
  A[Client Tx] --> B{Is EVM?}
  B -- Yes --> C[C-Chain Snowman]
  B -- No  --> D[X-Chain DAG Avalanche]
  A2[Validator Register] --> E[P-Chain]
  C --> F[Repeated Sample<br/>k=20, alpha=14, beta=20]
  D --> F
  F --> G[Confidence reaches threshold]
  G --> H[Accept / Finalize]
  E --> I[Subnet / L1 management<br/>ICM Warp Messaging]
```

## 3. 架构剖析

### 3.1 分层视图

```
┌────────────────────────────────────────────────┐
│ Wallets / SDKs / Teleporter Relayer            │
├────────────────────────────────────────────────┤
│ JSON-RPC (EVM) / Avalanche API                 │
├────────────────────────────────────────────────┤
│ AvalancheGo (node):                            │
│  ├── Consensus Engine (Snowman / Avalanche)    │
│  ├── VM Manager (plug in multiple VMs)         │
│  │     ├── PlatformVM (P-Chain)                │
│  │     ├── AVM (X-Chain)                       │
│  │     ├── Coreth (C-Chain EVM)                │
│  │     └── Subnet-EVM / Custom VM              │
│  ├── Networking (Avalanche p2p)                │
│  ├── Chain Manager                             │
│  └── Database (LevelDB)                        │
├────────────────────────────────────────────────┤
│ Primary Network: 3 chains                      │
└────────────────────────────────────────────────┘
```

### 3.2 核心模块清单（映射 `ava-labs/avalanchego` v1.11.x）

| 模块 / 目录 | 职责 | 依赖 | 可替换性 |
| --- | --- | --- | --- |
| `snow/consensus/` | Snowball / Snowman 核心算法 | math/sampler | 可切换策略 |
| `snow/engine/snowman/` | 线性链引擎 | consensus | 主力 |
| `vms/platformvm/` | P-Chain 验证者/子网逻辑 | staking, warp | 核心 |
| `vms/avm/` | X-Chain UTXO 资产 | secp256k1 | 核心 |
| `vms/proposervm/` | 在 Snowman 之上加 proposer 窗口 | | 防 MEV 抢跑 |
| `coreth/` (独立仓库) | C-Chain EVM，基于 geth | core-eth types | 可替换 subnet-evm |
| `network/p2p/` | TLS + msgpack 协议 | crypto | 可替换 QUIC |
| `indexer/` | Tx / Block 索引，供 API | database | 可关 |
| `api/` | JSON-RPC + HTTP Handler | 上述 | 标准接口 |
| `database/` | LevelDB / PebbleDB 封装 | | 可替换 |
| `chains/` | Chain Manager, 加载多 VM | vm manager | 核心 |
| `subnets/` | Subnet / L1 管理 | platformvm | 核心 |

### 3.3 数据流：EVM Tx 在 C-Chain 的生命周期

1. **dApp 签名 Tx**：通过 MetaMask / ethers，发到 `eth_sendRawTransaction`。
2. **coreth mempool**：与 geth 相同，按 gas price / nonce 排序。
3. **ProposerVM 随机选择 proposer 窗口**：给一批 validator 在短时间窗内 "优先" 提议区块，用于防止抢跑与控制网络风暴。
4. **Snowman poll**：提议块后，本节点抽样 20 个 validator，查询其 parent preference。
5. **连续 20 轮支持** → accept，写入 coreth BlockChain，触发 EVM 执行 → 生成 receipts。
6. **跨链 ICM**：若 Tx 调用了 `Teleporter::sendMessage`，Warp 预编译合约产出跨链消息，由 Relayer 捕获并投递至 target L1。
7. **终局性**：实测 1–2 s，绝大多数交易所 / bridge 设为 1 个确认。

### 3.4 客户端多样性

- **AvalancheGo (Go)**：唯一官方主实现。
- **Avalanche-rs (Rust)**：Ava Labs 在孵化的 Rust 节点，尚未主网生产。
- **Subnet-EVM / HyperSDK**：用户构建自定义 L1 的模板。
- 与多数新 L1 类似，客户端多样性较低，是共识层的重要风险。

### 3.5 扩展接口

- **EVM JSON-RPC**：C-Chain 完全兼容 MetaMask 生态。
- **Avalanche API**：平台链专有接口 `/ext/bc/P`, `/ext/bc/X`。
- **Warp + Teleporter Precompile**：`0x02...05` 地址调用跨链消息。
- **HyperSDK**：构建非 EVM L1 的框架（Rust + Go）。
- **ACP (Avalanche Community Proposal)**：协议级提案治理流程，对应 EIP。

## 4. 关键代码 / 实现细节

Snowman 投票循环（简化自 `avalanchego/snow/engine/snowman/transitive.go`）：

```go
// Poll 一个块是否被 accept
func (t *Transitive) poll(ctx context.Context, blkID ids.ID) {
    // 1. 随机采样 k 个 validator
    vdrs, _ := t.Config.Validators.Sample(t.Ctx.SubnetID, t.Params.K)
    // 2. 向每个发 PullQuery
    reqID := t.RequestID
    for _, v := range vdrs {
        t.Sender.SendPullQuery(ctx, v, reqID, blkID)
    }
    // 3. 收集响应，在 handler 中 t.Consensus.RecordPoll(...)
}

// Snowball 记录投票
func (sb *snowball) RecordSuccessfulPoll(blkID ids.ID) {
    sb.numSuccessfulPolls[blkID]++
    if sb.numSuccessfulPolls[blkID] >= sb.params.Beta {
        sb.finalized = true
    }
}
```

> 真实实现覆盖 ProposerVM 窗口、blocking / non-blocking poll、subnet 隔离；详见 commit tag `v1.11.x`。

## 5. 演进与版本对比

| 版本 | 时间 | 关键变化 | 影响 |
| --- | --- | --- | --- |
| Mainnet Launch | 2020-09-21 | Snowman + 三链 | 公链启动 |
| Apricot | 2021 | Atomic Tx, EIP-1559 提前 | EVM 生态繁荣 |
| Banff | 2022 | C-Chain 性能、ProposerVM | 防 MEV |
| Cortina | 2023 | BLS 密钥，铺垫 Warp | Subnet 互通 |
| Durango | 2024-03 | Teleporter / ICM 上线 | 子网互操作 |
| Etna (ACP-77) | 2024-12 | Subnet → L1，自定义验证者集 | 成本大幅下降 |
| Fuji + Frog/HyperSDK | 2025 持续 | 新应用链框架 | 据官方路线 |

## 6. 实战示例

```bash
# 本地运行 avalanchego
git clone https://github.com/ava-labs/avalanchego && cd avalanchego
./scripts/build.sh
./build/avalanchego \
  --network-id=local \
  --staking-enabled=false \
  --http-host=0.0.0.0

# 用 avalanche-cli 创建一个 Subnet / L1（Etna 风格）
avalanche subnet create myL1
avalanche subnet deploy myL1 --local

# 部署 EVM 合约到 C-Chain
forge create --rpc-url https://api.avax-test.network/ext/bc/C/rpc \
  --private-key $PK src/Counter.sol:Counter

# 查询 Tx 状态
curl -X POST --data '{"jsonrpc":"2.0","id":1,"method":"eth_getTransactionReceipt","params":["0x..."]}' \
  -H 'content-type:application/json' \
  https://api.avax.network/ext/bc/C/rpc
```

预期输出：节点启动后 RPC 可用；subnet 部署返回新的 chain ID；Tx 1–2 秒内有 receipt 且 `status=0x1`。

## 7. 安全与已知攻击

1. **2021-11 Pangolin/Snowball 网络拥堵**：C-Chain 因活动激增出现内存池爆满 / 节点掉线，被社区质疑 "2 秒终局" 与实际吞吐的兼容性；后续通过 atomic gas、合约优化修复。
2. **跨链桥 Avalanche Bridge（AB） 相关事件**：AB 本身由 Intel SGX 托管（可信执行环境），被批"中心化"；2022 年社区切向 LayerZero + Warp 组合，Warp/Teleporter 成为首选。
3. **理论攻击面**：① $\ge 50\%$ stake 攻击可破坏 Snowman（与 BFT 不同，不是 1/3 上限；但社区常误读 Snowman 为"4500 → 5000 才安全"）；② proposer 抢跑：当 ProposerVM 窗口内某 proposer 串通 MEV，可排序利己，缓解方案是随机延迟与 PBS；③ **无 slashing** 长期是治理与质押诚信的系统性隐患。

## 8. 与同类方案对比

| 维度 | Avalanche | Cosmos | Polkadot | Ethereum |
| --- | --- | --- | --- | --- |
| 共识 | Snowman (亚稳定抽样) | CometBFT (BFT) | BABE+GRANDPA | LMD-GHOST+FFG |
| BFT 容错上限 | < 50% | < 33% | < 33% | < 33% |
| 子链模型 | L1 (ACP-77) + ICM | App-Chain + IBC | Parachain 共享安全 | L2 Rollup |
| 共享安全 | 可选（原 Subnet 强制，Etna 后自选） | 可选（Interchain Security） | 强制共享 | L2 经 Ethereum DA |
| 智能合约 | EVM (C-Chain), 自定义 VM | CosmWasm / EVM 可选 | Wasm (ink!) | EVM |
| 跨链 | Warp / Teleporter | IBC | XCMP | CCIP / LayerZero 等 |
| 终局 | 1–2 s | 5–7 s | 12–60 s | 12 min |
| 客户端数 | 1 | 2+ (gaia, cosmos-sdk based) | 2 (Substrate, Kagome) | 5+ |

## 9. 延伸阅读

- **官方文档 / Spec**：
  - [Avalanche Developer Docs](https://build.avax.network/docs)
  - [Avalanche Whitepapers](https://www.avalabs.org/whitepapers)
  - [ACP Registry](https://github.com/avalanche-foundation/ACPs)
- **核心论文**：
  - [Snowflake to Avalanche (2018)](https://www.avalabs.org/whitepapers)
  - [Scalable and Probabilistic Leaderless BFT (2019)](https://arxiv.org/abs/1906.08936)
- **权威博客**：
  - [Ava Labs Blog](https://www.avalabs.org/blog)
  - [Messari Avalanche 研究](https://messari.io/)
  - [a16z — Avalanche Subnet 分析](https://a16zcrypto.com/)
- **视频**：
  - Emin Gün Sirer 于 Consensus 的 Avalanche 原理演讲
  - Luigi D'Onorio DeMeo 在 AvalancheSummit 的 Etna 升级介绍
- **规范 / ACP**：
  - [ACP-77 Reinventing Subnets](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/77-reinventing-subnets)
  - [ACP-30 Integrate Avalanche Warp Messaging](https://github.com/avalanche-foundation/ACPs)

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 亚稳定共识 | Metastable Consensus | Avalanche 共识家族核心思想 |
| 重复随机抽样 | Repeated Random Subsampling | 每轮抽 k 个、以置信度累积 |
| 雪花/雪球 | Snowflake / Snowball | 共识协议的两个中间阶段 |
| 雪人 | Snowman | 线性化 Avalanche |
| 三链 | P/X/C-Chain | Platform / Exchange / Contract Chain |
| 子网 | Subnet | 一组 validator 共同验证的若干链 |
| L1 (Avalanche) | Avalanche L1 | Etna 后自治的子网，自管 validator |
| 传送门 | Teleporter | 基于 Warp 的跨 L1 消息合约 |
| 跨链消息 | Interchain Messaging (ICM) | Warp + Teleporter 总称 |
| 提议 VM | ProposerVM | 在 Snowman 之上加入 proposer 窗口 |

## 11. FAQ

以下问题来自与学习者的真实对话，聚焦 Avalanche 最容易被 EVM / Cosmos / Polkadot 经验"想当然"的几个点：共识家族的真实差异、ProposerVM 的 MEV 语义、三链互通与 Subnet → L1 的范式转变。

### Q1：Snowflake / Snowball / Snowman 的差异与演进到底是什么？

共识家族的完整链条其实是 **Slush → Snowflake → Snowball → Avalanche(DAG) / Snowman(链)**，每一步都在补前一步的缺陷。核心原语都是"每轮随机抽 $k$ 个节点，若 $\ge \alpha$ 同意就响应"，差异在**状态记忆**与**决策规则**。

**Slush（起点，无记忆）**：每轮抽 $k$ 个邻居，若 $\ge \alpha$ 同色就切换到该色；固定 $m$ 轮后停。没有计数器、没有终局信号——只能说明"当前倾向"，不能"确认接受"。

**Snowflake（加 counter，给出决定点）**：一致响应则 `counter++`，不一致则清零并切换；$counter \ge \beta$ 即 finalize。提供了终局条件，但 **counter 清零太敏感**，网络抖动或攻击者轻微扰动就能让其前功尽弃，容易被 metastability 攻击拖住。

**Snowball（加 confidence 向量，稳健化）**：为每种选择维护独立的 **累计置信度** $d(v)$（不清零，只累加），当前 preference = 置信度最高者；切换要求对手选项置信度严格超过当前。摆脱了"一有反例就归零"的过度敏感，是 Avalanche 家族的**算法基石**。`avalanchego/snow/consensus/snowball/tree.go::RecordPoll` 就是这套逻辑。

**Avalanche（Snowball × DAG）**：每笔 Tx 是 DAG 节点，引用多条父 Tx；对每个 **conflict set**（竞争同一 UTXO 的 Tx）各跑一份 Snowball；无冲突（virtuous）交易可**并行确认**，阈值较松 $\beta_1=15$；有冲突（rogue）阈值较严 $\beta_2=20$。X-Chain 用的就是它。

**Snowman（Snowball × 链）**：把 DAG 退化为单父链，在整条链上跑 Snowball。失去并行性，换来**全序**——适合需要线性状态的 EVM 场景。C-Chain、P-Chain 均用 Snowman。后续又叠了 ProposerVM 作为窗口化 proposer 机制防 MEV 抢跑。

**一句话归纳**：

| 协议 | 记忆机制 | 解决的问题 |
| --- | --- | --- |
| Slush | 无 | — |
| Snowflake | counter（会清零） | 有决定点 |
| Snowball | 累计 confidence | 抗噪声、抗 50/50 拉锯 |
| Avalanche | Snowball 实例 × DAG | UTXO 场景并行确认 |
| Snowman | Snowball 实例 × 链 | EVM 场景全序 |

### Q2：ProposerVM 机制如何做到防 MEV？

**先看原始问题**：Snowman 本身**无 leader 选举**——任何 validator 在任何时刻都能发起 `poll` 提议新块。这在 MEV 视角下有两类灾难：(1) 抢跑机器人看到 mempool 里价格敏感的 swap，立刻用自己的节点打包"插在前面"的块广播，谁先被足够多节点 poll 中谁就赢；(2) 多 validator 同时提议候选块，Snowball 要花额外轮数收敛，放大抖动。

**ProposerVM（Banff 2022 升级）的核心机制**（源：`avalanchego/vms/proposervm/README.md`）——在 Snowman 之上包一层，引入**确定性的时间窗轮值 proposer**：

```
每到高度 H，所有节点用同一套确定性算法算出 proposer 列表：
  1. 拉取 P-Chain 高度 P 时的 validator 集合
  2. 按 nodeID 排序
  3. seed = H XOR chainID 作随机源
  4. 按 stake 加权无放回采样，取前 6 位 → [p0, p1, ..., p5]

时间窗规则：
  - 第 i 位 proposer 的窗口起点 = parent.Timestamp + i × 5s
  - 只有 p_i 能在自己窗口内提议块，其他人广播的块被拒绝
  - 30 秒（6 × 5s）后窗口耗尽，任意 validator 可提议（liveness fallback）
```

块头强制携带 `ParentID / Timestamp / PChainHeight / TLS Certificate / Signature`——其他节点在 Snowman poll 前先验证"这块是否由当前窗口合法 proposer 签出"，不合法直接丢弃。

**为什么能防 MEV 抢跑**：

| 攻击路径 | 无 ProposerVM | 有 ProposerVM |
| --- | --- | --- |
| 抢跑机器人看到 victim tx 后立刻自发块 | 只要抢在多数 poll 前广播就可能赢 | 必须**恰好是**当前 5s 窗口的 proposer，否则块无效被拒 |
| 多 proposer 同时竞争 | 常态，Snowball 需更多轮收敛 | 单 proposer 独占窗口，零竞争 |
| 排序权归属 | 全网可抢，链外竞价无序 | 明确落在当前窗口 proposer 手里，可监管、可审计 |

**但它不是"消灭 MEV"，而是"把 MEV 收归有序 proposer"**——窗口内的 proposer **依然可以自己吃 MEV**，跟 Ethereum PBS 出现前的 block builder 一样。ProposerVM 解决的是 **外部抢跑者 + 竞速 spam**，不是 proposer 内生性的 MEV。Ava Labs 后续方向是随机延迟 + PBS 式 builder/proposer 分离。

### Q3：P-Chain / X-Chain / C-Chain 三条链之间是怎么互通的？

**前提**：P/X/C 都跑在 AvalancheGo 这一个 binary 里，同一批 validator 验证所有三条链。因此**互通不是"跨链桥"，而是"同一个节点内部的跨数据库原子操作"**——这是理解整个机制的关键前提。

**核心原语：`Atomic Tx = ExportTx + ImportTx`**（源：`avalanchego/chains/atomic/README.md`）

```
ChainA (源)                     SharedMemory (每个节点本地)           ChainB (目)
─────────                       ─────────────────────────              ─────────
ExportTx ── Snowman finalize ── 写入 key: (destChainID, UTXO)
                                value: amount/owner
                                traits: owner address
                                                                      ImportTx ── Snowman finalize
                                                                      读取并消费 UTXO
                                                                      本地铸出到目标地址
```

**SharedMemory** 是每个节点本地的一个**分区 KV 库**，按 `(sourceChainID, destChainID)` 切片，带 traits 索引（按 owner 反查）。它不是"链外桥合约"，而是节点进程内的一块共享数据库。

**原子性保证**：
- ExportTx 在源链 accepted 的瞬间 shared memory 才写入条目 → 源链 UTXO 已不可再花
- ImportTx 在目标链接受后 shared memory 条目被消费并删除
- 批量 `Apply` 保证"写源 + 清 shared"在数据库层面是一个 batch

所以**不存在"同时提交"，是严格先后；但 shared memory + 两端共识共同保证 exactly-once，无双花无丢失**。

**实际用途矩阵**：

| 方向 | 典型场景 | 机制要点 |
| --- | --- | --- |
| X → P | 从交易所把 AVAX 打到 P-Chain 做 staking | UTXO → UTXO |
| P → X | validator 领奖励后取回流通 | 同上反向 |
| X → C | AVAX 送进 EVM 账户付 gas | UTXO → account 余额 |
| C → X | 从 C-Chain 转回 UTXO 形态 | account → UTXO |
| C → P | Cortina 后支持直接从 C-Chain 账户参与 staking | 原需 C→X→P 两跳，现一跳 |

**与"真跨链桥 / Warp"的区别**：Primary Network 内（P/X/C）走 Atomic Tx + SharedMemory，**零信任第三方**；Subnet/L1 之间 validator 集合可能不同，不能再直接读对方本地 DB，必须走 **Warp Messaging**（BLS 聚合签名 + Relayer + 目标链验签）。

一句话：P/X/C 互通 = **每个节点本地的分区 KV（SharedMemory）+ 两端各走自己的 Snowman 终局**——本质是"同一进程里的原子数据库搬运"。

### Q4：Subnet 的概念、原理、维护机制（Etna / ACP-77 前后）？

**原始概念（Etna 之前）**：Subnet = "一组 validator 共同验证的 ≥1 条链"，本质是 P-Chain 上的一条记录 `SubnetID → { validators[], chainIDs[] }`。三条核心规则决定了其早期形态：

1. Validator **必须先是** Primary Network validator（硬性前置，共享安全理念）
2. Validator 集合增删改走 P-Chain 的 `AddSubnetValidatorTx` / `RemoveSubnetValidatorTx`——所有权在 subnet owner 手里
3. Primary Network 给 subnet 提供 bootstrapping / peer discovery / P-Chain 作为元数据真源

**问题**：一个 validator 为了验证 1 条低活跃小链，却要同步整个 Primary Network 全历史状态——这是 2022-2024 年 subnet 生态跑不起来的根因。

**Etna + ACP-77（2024-12）的范式转变**（源：`avalanche-foundation/ACPs/77-reinventing-subnets`）：Subnet 重命名为 **Avalanche L1**，validator 的**所有权和计费模型**都改了。

| 维度 | Subnet（旧） | L1（ACP-77 后） |
| --- | --- | --- |
| Validator 是否必须验 Primary Network | 必须 | **不必须**，只需同步 P-Chain |
| 质押要求 | 2000 AVAX | **无 stake 要求**，改为"持续费" |
| Validator 管理权 | P-Chain + subnet owner 多签 | **L1 自己的合约**（validator manager） |
| 计费模型 | 一次性 2000 AVAX 锁仓 | 按秒连续付费 $M \cdot \exp(x/K)$ nAVAX/s（~1.33 AVAX/月） |
| 管理消息信道 | P-Chain 交易 | **Warp Message**（L1 manager 合约签 BLS → P-Chain 验签） |

**连续费细节**：每个 L1 validator 在 P-Chain 有 `Balance` 账户，每秒按 $M \cdot \exp(x/K)$ 扣除，其中 $x = \max(0, \text{当前 L1 validator 数} - 10{,}000)$，$M = 512$ nAVAX/s。Balance 归零 → validator 自动 inactive（仅存磁盘），任何人可通过 `IncreaseL1ValidatorBalanceTx` 充值复活。等价于把"一次性大额质押"替换成"按 pay-as-you-go 付租金"，尺度从 2000 AVAX/节点 降到十几 AVAX/节点/年。

**维护 L1 的 5 条核心 P-Chain Tx**，每条都要附带一条来自 L1 validator manager 合约的 Warp Message（BLS 聚合签名）作为授权证据：

| 交易 | 作用 | 授权方 |
| --- | --- | --- |
| `ConvertSubnetToL1Tx` | 一次性把老 Subnet 转 L1，指定 `(chainID, managerAddress)` | 原 Subnet owner |
| `RegisterL1ValidatorTx` | 加入新 validator；带 BLS PoP，expiry ≤ 24h，`validationID` 防重放 | L1 manager（Warp） |
| `SetL1ValidatorWeightTx` | 改权重 / 移除（weight=0）；nonce 防重放；禁止移除"最后一个" validator | L1 manager（Warp） |
| `DisableL1ValidatorTx` | validator 自行下线并提回余额，恢复通道 | `DisableOwner`（validator 本人） |
| `IncreaseL1ValidatorBalanceTx` | 任意人可充值，激活 inactive validator | 任意 |

**典型维护流**：

```
1. 项目方部署 L1 manager 合约（可在任何支持 Warp 的链上，如 C-Chain）
2. 调用 ConvertSubnetToL1Tx，初始 validator 集和 balance 写入 P-Chain
3. 新节点想加入：
     a. 节点方找 L1 manager 合约申请（业务逻辑由 L1 自定：PoS/PoA/whitelist 任选）
     b. manager 合约产出 RegisterL1ValidatorMessage + Warp BLS 签
     c. 节点方提交 RegisterL1ValidatorTx 给 P-Chain，附 Warp 签名
     d. P-Chain 验签通过 → 写入 validator 集 + 记录 validationID
4. 日常：任何人给 validator Balance 充值；归零则 inactive
5. 节点退出：manager SetWeight(0)，或 validator 自己 DisableTx 拿回余额
```

**P-Chain 角色**从"决策者"退化为"**中立公证员**"——不再判断谁该入集，只验证"是否有来自合法 manager 的 BLS 签名"，业务规则全部搬到 L1 自己的合约里。

### Q5：Avalanche L1 与 P-Chain / C-Chain / X-Chain 是什么关系？

**一句话**：**Primary Network（P+X+C）本身就是一个"特殊的、内置的 L1"**；其他 L1 是**对等（peer）**的独立链——不是 C-Chain/X-Chain 的子链，但**强制依赖 P-Chain 作为元数据注册表**。

```
                 ┌───────────────────────────────────┐
                 │   Avalanche 网络(全体节点的集合)  │
                 └───────────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
   Primary Network            L1: DFK               L1: Dexalot  …
   (SubnetID=0)               (独立 validator)       (独立 validator)
   ┌─────────────┐            ┌─────────────┐        ┌─────────────┐
   │ P-Chain     │◄──注册─────┤ 链 1        │        │ 链 1        │
   │ X-Chain     │   元数据    │ 链 2(可选) │        │ 链 2(可选) │
   │ C-Chain     │            └─────────────┘        └─────────────┘
   └─────────────┘
```

关键观察：**P-Chain 自己只是 Primary Network 里的一条链，但它同时扮演"整个 Avalanche 宇宙的元数据真源"**——每个 L1 在 P-Chain 上都有一行登记。

**四个维度拆解关系**：

**① 身份层面**：L1 和 Primary Network **同级对等**，不是父子。两者平等运行，DFK 的 validator 不是 C-Chain 的下游，它们平行存在。Primary Network 最大的特殊性只是"被 Ava Labs 用 AVAX stake + 高 validator 数兜底"。

**② 依赖层面**：L1 **必须**依赖 P-Chain，**不必**依赖 X/C。

| Primary Network 的链 | L1 validator 是否必须同步 |
| --- | --- |
| **P-Chain** | **必须**（validator 集、balance、Warp 签名验签、peer discovery 都在这里） |
| X-Chain | 否 |
| C-Chain | 否（但 L1 manager 合约可以部署在 C-Chain） |

Etna 后 **Primary Network validator 和 L1 validator 可以完全不重合**——这是 Etna 前后最大的范式差。

**③ 通信层面**：两种完全不同的机制。

```
Primary Network 内(P ↔ X ↔ C):
    Atomic Tx + SharedMemory（同一批 validator 的本地 KV 搬运，零信任第三方）

Primary Network ↔ L1 / L1 ↔ L1:
    Warp Message（BLS 聚合签名）+ Teleporter 合约 + Relayer
    （validator 集不同，必须靠密码学签名验签）
```

两套**不可混用**——C-Chain 到 X-Chain 用 Atomic Tx；C-Chain 到任何 L1 只能用 Warp。L1 的资产要回到 C-Chain，必须走跨链桥语义（Teleporter），不是数据库搬运。

**④ 治理层面**：P-Chain 从"决策者"退化为"公证员"（见 Q4）。

**L1 能否脱离 Primary Network 完全独立？不能**——这是 Avalanche 跟 Sui/Aptos 这类独立 L1 的分水岭：
- 每笔 `RegisterL1ValidatorTx` / `SetL1ValidatorWeightTx` 都必须提交到 P-Chain
- L1 validator 的连续费从 P-Chain 账户扣
- Warp BLS 公钥注册在 P-Chain，其他链验 Warp 消息时要查 P-Chain 的 canonical validator set

所以准确的说法：**L1 共享 P-Chain 作为"登记处 + 计费处 + 签名真源"，但各自独立运行业务、各自自治 validator 集，也可以完全不跑 X/C-Chain。**

**历史类比**：Etna 前像"分公司"——subnet 是 Primary Network 派出的，员工必须先属于总部；Etna 后像"加盟店"——L1 自主经营、自聘员工，只在总部（P-Chain）登记商标和交月租。Avalanche 架构师在 Etna 后把 Primary Network 改称 "**the only non-optional L1**"——这个说法最精准：一切都是 L1，只是 P/X/C 组成的那个 L1 是默认必装的。

---

*Last verified: 2026-04-25*
