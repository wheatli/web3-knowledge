---
title: 公链总览与横向对比
module: 01-infrastructure/public-chains
priority: P0
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://l2beat.com
  - https://messari.io
  - https://defillama.com
  - https://chainspect.app
  - https://ethereum.org/en/developers/docs/
---

# 公链总览与横向对比

> **TL;DR**：公链（Public Blockchain, L1）是整个 Web3 堆栈的信任基座。本页作为 `01-infrastructure/public-chains/` 的门户，从 **定位光谱 → 共识类型 → 账本模型 → 横向对比总表 → 阅读路径 → 延伸阅读** 六个维度把子页面串起来。公链不是同质竞品：Bitcoin 作价值存储与结算层、Ethereum 作可编程结算与数据可用性底座、Solana/Aptos/Sui 作高性能执行层、Cosmos/Polkadot 作主权应用链网络、TON/Tron 作支付/消费级大众链、Hyperliquid 则是为衍生品而生的专用执行环境。把它们放在同一张对照表上只是为了**建立参考系**，而不是用一个指标分高下。

---

## 1. 公链的定位光谱

公链之间最本质的差异是**定位而不是 TPS**。按"用户-开发者-机构"三方关心的价值主张，可把主流 L1 归入 5 个定位带：

### 1.1 价值存储 / 结算层（Store of Value）

**代表**：Bitcoin。

- 目标：最小攻击面、最高抗审查、最简单的信任假设。
- 设计选择：非图灵完备脚本、UTXO、PoW、10 分钟块、绝对保守的协议升级节奏。
- 经济模型：2100 万硬顶 + 减半 + 手续费长期收敛。
- 外延扩展：Lightning、Ordinals/Runes、Stacks / RSK / BitVM 系 L2。
- 读这一类的目的：理解"为什么越简单越安全"、"为什么 finality 可以是概率的"、"为什么能量消耗是 feature 而非 bug"。

### 1.2 可编程智能合约平台（World Computer）

**代表**：Ethereum；子代表 BNB Chain、Avalanche C-Chain、Tron（兼容 EVM 但不是完全同级）。

- 目标：让任意逻辑可以以"信任中立"的方式运行。
- 设计选择：图灵完备 VM + Gas 计费、Account 模型、PoS、Rollup-centric 扩容路线。
- 经济模型：EIP-1559 销毁 + PoS 发行 + MEV 拍卖。
- 读这一类的目的：理解状态机抽象、MEV、L2 与 DA 分层、账户抽象、共识层-执行层解耦。

### 1.3 高性能单体执行层（Monolithic High Throughput）

**代表**：Solana、Aptos、Sui、Sei、Hyperliquid（链下撮合 + 链上结算的混合体）。

- 目标：把"世界计算机"从 15 TPS 提升到千级甚至万级 TPS，同时保留单一共识域内的原子可组合性。
- 设计选择：并行执行（Sealevel / 对象依赖图 / Block-STM）、定制 VM（sBPF、MoveVM、AptosVM）、硬件友好协议（QUIC、GPU 验签）。
- 代价：节点硬件门槛高、客户端多样性弱、历史数据保留压力大。
- 读这一类的目的：理解硬件 - 协议 co-design、原子可组合性 vs 模块化的取舍。

### 1.4 主权应用链网络（Sovereign App-Chain Network）

**代表**：Cosmos Hub / Polkadot / Avalanche Subnets / Berachain-IBC 家族。

- 目标：每个应用拥有自己的共识实例（validator 集合、gas 代币、升级节奏），通过标准化 IBC / XCM 做跨链。
- 设计选择：Tendermint BFT / GRANDPA+BABE、跨链消息协议、shared security 可选。
- 读这一类的目的：理解"多链互操作"vs"单一 L1 + rollup"的不同模块化哲学。

### 1.5 支付 / 消费链（Consumer-Scale Payments）

**代表**：TON、Tron、BNB Chain、NEAR。

- 目标：极低手续费、极大流通量、与中心化社交/交易所深度绑定（Telegram、USDT、Binance、NEAR ↔ Phone Numbers）。
- 设计选择：PoS/DPoS 降低门槛、预留大量免费 quota（NEAR storage staking、TON gas credits）。
- 读这一类的目的：理解链作为"消费级支付基础设施"时的产品决策。

### 1.6 隐私链（Privacy-First）

**代表**：Monero、Zcash、Aleo、Namada。

- 目标：交易隐私（金额、收发方不可关联）。
- 设计选择：RingCT/Bulletproofs、zk-SNARK、MASP（Multi-Asset Shielded Pool）。
- 本知识库暂不单独列页，但未来会在 `02-cryptography/zkp/` 与 `07-privacy/` 交叉覆盖。

---

## 2. 共识类型对照矩阵

共识算法决定了**谁能打包区块**以及**区块在何种意义上是最终的**。下表按"安全假设 / 终局类型 / 代表链 / 典型缺陷"梳理：

| 共识家族 | 安全假设 | 终局类型 | 代表链 | 典型缺陷 |
| --- | --- | --- | --- | --- |
| **Nakamoto PoW** | 诚实算力 > 50% | 概率（6 确认 ~ 1h） | Bitcoin | 能耗、吞吐低 |
| **Ethash-like PoW** | 同上，ASIC 抗性 | 概率 | Ethereum (pre-Merge)、ETC | 同上 |
| **PoS + GHOST/FFG (Gasper)** | 诚实质押 > 2/3 | 经济终局（~12.8 min） | Ethereum (post-Merge) | 无"客观" re-sync 能力（弱主观性） |
| **Tendermint BFT** | 诚实 validator > 2/3 | 即时（1 block） | Cosmos Hub、BNB Chain BC | 100 以内 validator 上限 |
| **HotStuff / HotStuff-2** | 同上 | 即时 | Diem → Aptos、Sui（variant） | Leader 轮换复杂度 |
| **DPoS** | 21–100 超级节点 | 即时/准即时 | Tron、EOS、Hive | 去中心化程度弱 |
| **PoH + Tower BFT** | 同 PoS BFT | 确定性 ~12.8s | Solana | 硬件门槛高 |
| **Avalanche (Metastable)** | 概率收敛到确定 | <2s | Avalanche P/X/C-Chain | 协议新颖，工程调优门槛高 |
| **Bullshark / Mysticeti (DAG BFT)** | >2/3 | 子秒级 | Sui、Aleo | 实现复杂 |
| **NPoS + GRANDPA+BABE** | 质押 + validator 集合 | Finality 独立于出块 | Polkadot | 吞吐受共享安全机制约束 |
| **BFT + VRF (如 Algorand)** | 2/3 honest stake | 即时 | Algorand、NEAR（Doomslug variant） | VRF 公平性依赖诚实大多数 |
| **Catchain / REGNUM**（TON） | BFT + shard committee | 子秒级 | TON | 架构复杂度高 |

**关键观察**：PoS 并不是一个算法，而是一组"如何选主 / 如何投票 / 如何惩罚"的设计空间；不同链的 PoS 差异远大于 PoS 与 PoW 的差异。阅读具体链页面时应先定位其共识家族，再读细节。

---

## 3. 账本模型对照

账本模型决定了**"状态"的表达方式**，进而影响并行度、可组合性、钱包/索引体验。

| 模型 | 代表链 | 并行度 | 可组合性 | 钱包心智模型 | 索引难度 |
| --- | --- | --- | --- | --- | --- |
| **UTXO** | Bitcoin、Litecoin、Cardano（eUTXO 变种） | 天然并行（不同 UTXO 无冲突） | 弱（需链下组合 PSBT） | "收款簿"：每笔收入是独立 coin | 高：需 tx-graph 级索引 |
| **Account** | Ethereum、BNB Chain、Avalanche、Tron、NEAR | 串行（默认）；Solana 通过预声明账户实现并行 | 强（原子 multi-call） | "银行账户"：单余额 | 低：按地址查即可 |
| **Object / Resource (Move)** | Sui、Aptos、Movement | 对象依赖图决定（强并行） | 强，且类型安全 | "数字资产 = 一等对象" | 中：需按对象 ID 追踪 |
| **Cell Model** | Nervos CKB | UTXO 扩展 + 脚本 | 中 | 类 UTXO | 高 |
| **Hybrid (Account + Resource)** | Aptos 的 object model、Sui 的 dynamic fields | 视场景 | 强 | 偏向"账户 + 附挂对象" | 中 |

**设计取舍小结**：UTXO 对并行和隐私友好但智能合约表达弱；Account 表达力最强但并行需要额外约束；Object/Resource 把资产写进类型系统，防伪造和重入最彻底但开发者心智负担大。

---

## 4. 横向对比总表

以下数据来自公开来源的 2026-Q1 综合统计（L2BEAT、DeFiLlama、Messari、Chainspect、各链官方 explorer）。TPS 采用 **实测 30 天 P95**（非"理论 TPS"）。确定性时间取 **经济终局**（probabilistic 链则标注 6 确认等价值）。

| 公链 | 实测 TPS (P95, 2026-Q1) | 终局时间 | 共识 | VM / 执行层 | 主要语言 | 主流客户端 | 生态 TVL (USD, 2026-04) | 数据来源 |
| --- | ---: | ---: | --- | --- | --- | --- | ---: | --- |
| **Bitcoin** | 7 | ~60 min (6 conf) | PoW + 最长链 | Script (非图灵完备) | — | Bitcoin Core, btcd, bitcoin-rs, Floresta | ~$1.5B (Stacks/RSK 等) | Chainspect / DeFiLlama |
| **Ethereum** | 20 L1 / 250+ aggregate | ~12.8 min | PoS Gasper | EVM | Solidity, Vyper, Yul | Geth, Nethermind, Besu, Erigon, Reth | ~$80B (含 L2 ~$55B) | L2BEAT / DeFiLlama |
| **Solana** | 1,200 (non-vote) | ~12.8s | PoH + Tower BFT | SVM (sBPF) | Rust | Agave, Firedancer, Frankendancer, Jito-Solana | ~$10B | Solana Compass / DeFiLlama |
| **Sui** | 800 | ~500ms (Mysticeti) | Mysticeti DAG BFT | MoveVM | Sui Move | sui-node | ~$2B | DeFiLlama / Sui Explorer |
| **Aptos** | 600 | ~900ms | AptosBFTv4 (HotStuff 变体) | AptosVM (Move) | Aptos Move | aptos-core | ~$600M | DeFiLlama |
| **Avalanche** | 50 (C-Chain) + N subnets | ~2s | Avalanche consensus | EVM (C-Chain), Custom (subnet) | Solidity | AvalancheGo, Coreth | ~$1.5B | DeFiLlama |
| **Cosmos Hub** | 10 (Hub), 100+ 单 appchain | ~6s | Tendermint BFT | CosmWasm / 自定义 | Go, Rust (wasm) | gaia, cosmos-sdk | ~$10B aggregate IBC | Mintscan / Map of Zones |
| **Polkadot** | 1,000 (跨 parachain 总和) | ~60s (GRANDPA) | NPoS + GRANDPA+BABE | WASM (Substrate) | Rust (ink!), Solidity (via Moonbeam) | polkadot-sdk (ex-substrate) | ~$600M | DeFiLlama |
| **TON** | 200 (主链) / shard 理论更高 | ~5s | BFT + dynamic sharding | TVM | FunC, Tact, Tolk | ton-node (validator-engine) | ~$300M | DeFiLlama / TON explorer |
| **BNB Chain** (BSC) | 100 | ~7.5s (fast finality, BEP-319) | PoSA (PoS + PoA hybrid) | EVM | Solidity | bsc (geth fork) | ~$6B | DeFiLlama |
| **Tron** | 60 | ~57s (~27 块) | DPoS (27 SR) | TVM (EVM 兼容) | Solidity | java-tron | ~$8B (主要 USDT) | DeFiLlama |
| **NEAR** | 100 (主分片) | ~2s (Doomslug) | Nightshade (sharded PoS) | NEAR Runtime (Wasm) | Rust, AssemblyScript | nearcore | ~$400M | DeFiLlama |
| **Hyperliquid** | 200,000 orders/s (链下撮合) | <1s on-chain state | HyperBFT (HotStuff 风) | HyperEVM (EVM) + L1 order book | Solidity | hyperliquid-node (闭源) | ~$3B (perp OI) | Hyperliquid stats |

*注*：TVL 随市场波动剧烈，以上为 2026-Q1 综合中位数口径；TPS 以"链上非 vote 交易"计（Solana 特有，vote 占 70%+ 需剔除）。数据**仅供建立数量级参考**，精确数字请以一手链 explorer 为准。

### 4.1 TPS 与终局时间的权衡

**费曼式直觉**：TPS 和终局时间是一对**对冲变量**。想更快 finality，就要更少 validator 或更激进的 pipelining；想更多 validator + 去中心化，就要更长的 gossip/投票窗口。Bitcoin 是一个极端（长终局、低 TPS、极度去中心化），Solana/Sui 是另一极端（短终局、高 TPS、硬件门槛换节点多样性）。Ethereum 站在中间并通过 L2 把"执行 TPS"外包给了 optimistic/zk rollup。

### 4.2 客户端多样性的意义

任何单一客户端 > 2/3 网络份额时，一个客户端 bug 就能引发共识级别的资金损失（Ethereum 2021 Medalla、Solana 2022 duplicate block 均是教训）。阅读各链页面时请特别关注"客户端生态"一节：只有 Ethereum 与 Solana（2024 年起）拥有多客户端实战压力测试。

---

## 5. 读者路径指引

不同角色读公链的目的不同，建议按下列路径切入：

### 5.1 给开发者

1. 先读 **[Ethereum](./ethereum.md)** §2.1–2.4 建立 Account/EVM/Gas/1559 的基础心智模型（这是整个 Web3 的最大公约数）。
2. 再读 **[Solana](./solana.md)** §2.1–2.5 理解"为什么并行执行需要预声明账户"，这是所有高性能链的共同挑战。
3. 如果目标是 Move 家族，跳到 **[Sui](./sui.md)** 和 **[Aptos](./aptos.md)**，对比 Object vs Resource 的差异。
4. 应用上线后还需要 L2 策略：请参阅 `01-infrastructure/rollups/` 与 `01-infrastructure/data-availability/`。
5. 典型跨链需求：读 **[Cosmos](./cosmos.md)** 的 IBC 章节。

### 5.2 给研究者

1. **[Bitcoin](./bitcoin.md)** §2.5–2.7：HashCash 形式化、二项概率模型、Taproot MAST——这些是共识理论的 primitives。
2. **[Ethereum](./ethereum.md)** §2.7–2.10、§3.10：Verkle、fork choice、经济学公式——是 PoS 理论的活教材。
3. **[Solana](./solana.md)** §2.7–2.8：VDF / 弱 VDF 辩论与 lockout 指数数学——理解"时钟式共识"。
4. **[Polkadot](./polkadot.md)** 的 GRANDPA 证明与 NPoS Phragmén 算法——经典分布式系统教学素材。
5. 交叉阅读 `02-cryptography/` 获得密码学 primitives。

### 5.3 给投资人 / 产业研究

1. 先读本页 §4 横向对比表建立参考系。
2. 再挑 3–5 条感兴趣的链深入其 §1（背景动机）与 §5（演进史）——这两节直接解释"团队、融资、催化事件"。
3. 每条链的 §7（安全与已知攻击）给出风险清单，是估值模型的必要输入。
4. 最后回到本页 §6 的延伸阅读（L2BEAT / Messari / Galaxy / Electric Capital），与市场前瞻对照。
5. 长期主义的投资人应额外关注"减半"、"供应公式"、"质押率"等经济学参数——它们决定了 5–10 年的供给侧。

---

## 6. 延伸阅读

### 6.1 Tier-1 数据与仪表盘

- **[L2BEAT](https://l2beat.com)**：L2 TVL、数据可用性分类、安全评级——Ethereum 生态权威。
- **[DeFiLlama](https://defillama.com/chains)**：全链 TVL、稳定币分布、跨链桥流量。
- **[Chainspect](https://chainspect.app/)**：实测 TPS / gas / 区块填充率对比。
- **[Artemis](https://www.artemis.xyz/)**：链上活跃度、收入、手续费对比。
- **[Messari](https://messari.io/)**：机构级研究报告，含各链季报。
- **[Token Terminal](https://tokenterminal.com/)**：收入 / 利润 / P-E 风格估值。

### 6.2 权威一手源

- Ethereum: [Execution Specs](https://github.com/ethereum/execution-specs)、[Consensus Specs](https://github.com/ethereum/consensus-specs)、[EIPs](https://eips.ethereum.org)
- Bitcoin: [BIPs](https://github.com/bitcoin/bips)、[Bitcoin Core](https://github.com/bitcoin/bitcoin)
- Solana: [SIMDs](https://github.com/solana-foundation/solana-improvement-documents)、[Agave](https://github.com/anza-xyz/agave)
- Cosmos: [ADRs](https://docs.cosmos.network/main/architecture)
- Polkadot: [PIPs](https://github.com/polkadot-fellows/RFCs)

### 6.3 教学经典

- Antonopoulos《Mastering Bitcoin》、《Mastering Ethereum》（O'Reilly）。
- Vitalik Buterin 个人博客 <https://vitalik.eth.limo>。
- Paradigm Research <https://www.paradigm.xyz/writing>。
- a16z crypto <https://a16zcrypto.com>。
- 登链社区中文资源 <https://learnblockchain.cn>。

### 6.4 机构报告

- Galaxy Research 季度报告：<https://www.galaxy.com/research>
- Binance Research：<https://www.binance.com/en/research>
- Electric Capital Developer Report（年度）：<https://www.developerreport.com>
- Coin Metrics State of the Network（周报）：<https://coinmetrics.io/insights>

### 6.5 视频 / 会议

- Devcon、Breakpoint、Sui Basecamp、Cosmoverse、Polkadot Decoded 年度大会 YouTube 全录。
- EthGlobal / Solana Hacker House 黑客松 recap。
- 3Blue1Brown《But how does bitcoin actually work?》入门视频。

---

## 7. 本模块子页面索引

按字母顺序（每页均包含 §1–§10 十节结构）：

- [Aptos](./aptos.md)
- [Avalanche](./avalanche.md)
- [Bitcoin](./bitcoin.md) ⭐ P0
- [BNB Chain](./bnb-chain.md)
- [Cosmos Hub](./cosmos.md)
- [Ethereum](./ethereum.md) ⭐ P0
- [Hyperliquid](./hyperliquid.md)
- [NEAR](./near.md)
- [Polkadot](./polkadot.md)
- [Solana](./solana.md) ⭐ P0
- [Sui](./sui.md)
- [TON](./ton.md)
- [Tron](./tron.md)

⭐ = 优先深度阅读；其余链建议按 §5 路径或"读者实际需求"按需展开。

---

*Last verified: 2026-04-22*
