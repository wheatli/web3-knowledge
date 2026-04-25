---
title: Aptos
module: 01-infrastructure/public-chains
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-25
primary_sources:
  - https://aptos.dev/
  - https://github.com/aptos-labs/aptos-core
  - https://aptos.dev/en/network/blockchain/blockchain-deep-dive
  - https://aptosfoundation.org/whitepaper
  - https://github.com/aptos-labs/aptos-core/tree/main/consensus
---

# Aptos

> **TL;DR**：Aptos 是 Meta / Diem 工程团队离职后创立的 Aptos Labs 于 2022-10 主网启动的 **通用型高性能 L1**。核心工程亮点有三：(1) 自研共识 **AptosBFT**（目前运行的是 Jolteon / AptosBFTv4，相当于 HotStuff 的乐观分支版本），两轮投票、线性通信复杂度、理论最坏 2Δ 终局。(2) **Block-STM** 并行执行引擎：借鉴软件事务内存的乐观并发，交易按预定顺序乐观并行执行，冲突时按依赖拓扑回滚重做。(3) **Aptos Move**：保留了 Diem Move 的 `global storage` 与"资源权限类型系统"，合约以 `move_to/move_from` 管理账户下的资源，强调 **线性类型 + 形式化可验证**。Aptos 与 Sui 虽然同为 Move 系，但设计哲学迥异：Aptos 选择"像 Ethereum 一样的统一账户状态，但把并行化交给运行时动态发现"；Sui 选择"在静态类型层面就把并行化表达出来"。生态覆盖 DeFi（Thala、Econia）、稳定币（USDT 原生、USDC）、游戏（Blocklords）、支付（Tether / Aptos Pay 集成）。据官方估计主网活跃验证者超 150 名。

---

## 1. 背景与动机

2022 年 2 月，Mo Shaikh（前 Meta Novi 合作负责人）与 Avery Ching（前 Meta 首席工程师）离开解散中的 Diem 项目，创办 Aptos Labs，沿用 Diem 留下的 Move 语言与 DiemBFT 技术栈重新打造公链。与同源的 Sui 路线不同，Aptos 保留了 Diem 的**账户 + 资源**模型，延续"合约兼容性优先"的工程思路，更接近 Ethereum 账户模型的直觉。

Aptos 的工程重心是 **高吞吐 + 亚秒级终局 + 形式化安全**，为此在执行层不走"静态声明账户"（Solana）或"对象冲突图"（Sui）的路线，而是押注 **Block-STM**——对开发者完全透明的动态并行化。这使 Aptos 能在不改变 Move 编程模型的前提下，把 Smart Contract 并行化推到极限。

主网 Launch 2022-10-17，代号 "Mainnet 1". 随后路线：2023-Q2 Shardines 研究、2023-Q3 AptosBFTv4 升级、2024 Indexer v2、2025 引入 **Zaptos / Shoal++** 减流水线等待。

## 2. 核心原理

### 2.1 形式化定义：账户与资源

Aptos 状态是一个 **账户字典** $\mathcal{S} : \text{Address} \to \text{Account}$。每个账户包含：

$$
\text{Account} = (\text{sequence\_number}, \text{authentication\_key}, \text{resources}: \text{TypeTag} \to \text{Resource})
$$

- **Authentication key**：支持 Ed25519 / Multi-Ed25519 / Secp256k1 ECDSA / Passkey / Keyless（OIDC）；地址是它的 SHA3-256 前 32 字节。
- **Resource**：Move 类型 `T` 的一个全局实例通过 `move_to<T>(signer, value)` 挂到账户下。类型系统保证 **同一账户同一类型至多 1 份资源**。这是 Aptos Move 与 Solidity 最根本的语义差异——**资源不能被复制，只能移动**（线性类型）。

交易是状态转移函数 $T : (\text{sender}, \text{sequence}, \text{payload}, \text{gas}) \to \Delta\mathcal{S}$，其中 payload 是一条 `EntryFunction` 调用或 `Script`。验证器用 **Aptos VM（Move VM 的 fork）** 执行。

### 2.2 关键算法：AptosBFT v4（Jolteon）与 Block-STM

**AptosBFT v4** 把 DiemBFT（HotStuff）的三阶段 pipeline 优化为 **两阶段乐观路径 + 回落到线性视图同步**：

1. **Proposal**：Leader 提议块 $B_r$，引用上轮 Quorum Certificate（QC）。
2. **Vote**：validator 投 BLS 签名给 Leader。Leader 聚合 2f+1 签名形成 QC，这一动作 **即锁又提交**（相较 HotStuff 的 Lock/Decide 分离）。
3. **Pacemaker**：若 Leader 超时未产 QC，启动 Timeout Certificate (TC)，进入下一视图。

相比 HotStuff，Jolteon 在正常路径只需 2 轮，延迟下降一半；异常路径通过 TC 聚合保留线性复杂度。更多细节见 [Jolteon paper](https://arxiv.org/abs/2106.10362)。

**Block-STM** 是 Aptos 执行层的关键（[Block-STM paper](https://arxiv.org/abs/2203.06871)）：

1. Leader 已把一批 Tx 按索引 0, 1, 2 ... 顺序打包在 Block 中，**这定义了正确的"虚拟串行顺序"**。
2. 执行阶段 **乐观并行**：多线程同时跑 Tx，不等待依赖。每个 Tx 记录它读过哪些状态 Key（Read Set）和写过哪些（Write Set），写入一个版本化存储（MV-HashMap）。
3. **Validation 阶段**：按索引递增扫描 Tx，检查它的 Read Set 对应的最新版本号是否依然有效；若 Tx $i$ 读到的值在 Tx $j<i$ 那里被覆写，则 Tx $i$ **abort** 并重新执行。
4. 循环直至所有 Tx 的 validation pass。
5. 核心数据结构 **MVMemory / MVHashMap**：`Key -> (txn_idx -> Value)`，支持按 idx 读到最近的已写版本。

失败模式 / 边界：
- **热点账户**（每个 Tx 都读写同一资源）退化为串行；测试中 1 账户争用使并行度接近 1x。
- **回滚成本**：Read/Write Set 重建是 O(k)，k 是触达的状态 key 数。过度依赖随机访问可能引起 thrash。

### 2.3 子机制拆解

1. **Gas & Storage Fee**：`gas_used * gas_unit_price + storage_fee`。存储相关操作（`move_to`, 写入 Table）收取 **状态存储押金**，删除时退回。旨在对抗状态膨胀。参见 [Aptos Fee Schedule](https://aptos.dev/en/network/blockchain/gas-txn-fee)。
2. **Quorum Store**：将 Mempool 解耦出共识。所有 validator 预先广播交易批次 (batch)，生成 **Proof of Store (PoS certificate)**。Leader 提议块时只需引用 PoS，而非携带完整交易，解决了 Leader 带宽瓶颈。
3. **账户抽象（AA）/ Keyless / Passkeys**：原生支持。`aptos_std::keyless_account` 模块通过 OIDC JWT + Groth16 证明用户拥有 Google/Apple/FB 账户对应的地址，**链上即开即用**。
4. **Parallel Execution + Sparse Merkle Proof**：状态以 SMT 维护。区块提交后仅广播状态根与少量 Merkle proof，轻节点可验证。
5. **Dynamic Config / On-chain Governance**：所有协议参数（Gas schedule、timeout、validator set）保存在 `0x1::config`，通过链上提案升级，无需硬分叉。
6. **Resource Groups**：把多个经常一起读写的 Resource 打包存于同一 blob，减少 RocksDB IO。

### 2.4 参数与常量（可治理）

| 参数 | 值 | 说明 |
| --- | --- | --- |
| Block time | ~0.15–0.25 s | Jolteon 乐观路径 |
| Epoch | 2 h（可改） | Validator set 切换 |
| Max block gas | 数千万级 | 可治理 |
| 最小 Validator stake | 1,000,000 APT | 可治理 |
| Gas unit | 1 APT = 1e8 oct | |
| Max txn size | 64 KB | |
| Signature | Ed25519 / BLS12-381 for vote aggregation | |

### 2.5 边界条件与失败模式

- **> 1/3 Byzantine stake**：BFT 假设破裂，链停摆或分叉。
- **热点账户**：Block-STM 退化为串行；真实 DeFi 项目（AMM 池）会引入 Resource Splitting 规避。
- **Leader 失联**：Pacemaker 超时推进视图，约 2 s 内换 Leader，用户可见区块产速下降。
- **Move 资源丢失**：如果合约 `move_to` 但未提供取出函数，资源永久冻结（类似于 Solidity 的"误转 token to contract"）。

### 2.6 图示

```mermaid
flowchart LR
  A[Mempool] --> B[Quorum Store<br/>batch + PoS cert]
  B --> C[Leader Proposes Block]
  C --> D[Jolteon Vote + QC]
  D --> E[Block-STM Parallel Execute]
  E --> F[Validation Loop<br/>abort & retry]
  F --> G[Commit + Write State]
  G --> H[Epoch Manager]
```

## 3. 架构剖析

### 3.1 分层视图

```
┌────────────────────────────────────────────────┐
│ SDKs (TS/Python/Rust/Go/Kotlin/Swift)          │
├────────────────────────────────────────────────┤
│ REST API / Indexer (GraphQL)                   │
├────────────────────────────────────────────────┤
│ aptos-node:                                    │
│  ├── Mempool (shared mempool)                  │
│  ├── Quorum Store                              │
│  ├── Consensus (AptosBFTv4 / Jolteon)          │
│  ├── Execution (Block-STM + Aptos VM)          │
│  ├── State Sync                                │
│  └── Storage (AptosDB on RocksDB)              │
├────────────────────────────────────────────────┤
│ Storage: Jellyfish Merkle Tree (SMT variant)   │
└────────────────────────────────────────────────┘
```

### 3.2 核心模块清单（映射 `aptos-labs/aptos-core` v1.x）

| 模块 / Crate | 职责 | 依赖 | 可替换性 |
| --- | --- | --- | --- |
| `aptos-node/` | 进程入口，协调所有 service | 下列全部 | 不可替换 |
| `consensus/` | Jolteon / AptosBFTv4 实现 | crypto, network | 可替换 BFT 变体 |
| `execution/` | Block-STM 调度 + Aptos VM 宿主 | `aptos-vm`, `storage` | 可替换 VM |
| `aptos-vm/` | Move VM 的 Aptos 特化（gas、natives） | move-vm-runtime | 稳定接口 |
| `mempool/` | 共享 mempool（broadcast-and-fetch） | network | 可调策略 |
| `quorum-store/` | Batch + PoS cert | consensus, network | 可关 |
| `storage/aptosdb/` | AptosDB = 多列簇 RocksDB | storage-interface | 可换 SQLite/BadgerDB |
| `storage/jellyfish-merkle/` | 状态 SMT | storage | 可替换 |
| `aptos-framework/` | 系统 Move 包（aptos_framework, aptos_stdlib） | Move | 通过治理升级 |
| `network/` | Noise + Multiplexing | tokio | 可换 QUIC |
| `state-sync/` | 新节点同步 / 历史下载 | network, storage | |
| `indexer/` + `indexer-grpc/` | 链上数据投射到 Postgres + 推流 | | 独立进程 |

### 3.3 数据流：Tx 端到端

1. **SDK 构造 Tx**：指定 `sender`、`sequence_number`、`payload = EntryFunction / Script`。
2. **REST `submit_transaction`**：节点接入 mempool，广播到其它 validators (shared mempool)。
3. **Quorum Store**：validator 本地把 Tx 打包成 batch，收集 2f+1 签名，生成 PoS。
4. **Consensus 提议**：轮值 Leader 打包 N 条 PoS 到块头，BLS 签名提议。
5. **投票与 QC**：Validators BLS 签，2f+1 形成 QC，直接提交。
6. **Block-STM 执行**：执行引擎按索引并行跑，validation 阶段排除冲突，生成 `TxnOutput`。
7. **State Commit**：写入 AptosDB / Jellyfish Merkle，更新 State Root。
8. **Indexer 下游**：事件流推送到 Indexer-gRPC，供 Hasura / GraphQL / Postgres 订阅。

端到端用户可见延迟：通常 **600 ms 上下**（Jolteon commit 后写库 + 推 API）。

### 3.4 客户端多样性

- **aptos-node (Rust)**：唯一主实现。
- **轻节点 / Rosetta / Indexer**：基于同一 codebase 的不同二进制。
- 与 Sui 同样面临 **单客户端风险**。官方有多客户端研究，但时间线未公开。

### 3.5 扩展接口

- **REST API**：`/v1/transactions`、`/v1/accounts/:addr/resource/:type`。
- **Indexer gRPC Stream**：订阅完整 Tx 流，做定制索引。
- **GraphQL (Hasura)**：官方 Indexer 暴露的常用查询（余额、NFT、事件）。
- **Move CLI / ABI**：标准化调用入口，类似 EVM ABI。

## 4. 关键代码 / 实现细节

Block-STM 核心循环（简化自 `aptos-core/execution/block-stm/`）：

```rust
loop {
    // 1. 调度器分配任务
    match scheduler.next_task() {
        Task::Execute(txn_idx) => {
            let output = vm.execute(&txns[txn_idx], &mv_memory);
            mv_memory.apply_writes(txn_idx, output.writes);
            scheduler.finish_execution(txn_idx);
        }
        Task::Validate(txn_idx) => {
            let read_set = mv_memory.read_set(txn_idx);
            if !mv_memory.validate(read_set, txn_idx) {
                // 读过的 key 在更早的 idx 被改 → abort
                scheduler.abort(txn_idx);
                mv_memory.mark_estimate(txn_idx);  // 重要：其后依赖此 idx 的 txn 读到 ESTIMATE
                scheduler.finish_validation(txn_idx, false);
            } else {
                scheduler.finish_validation(txn_idx, true);
            }
        }
        Task::NoTask => if scheduler.done() { break; }
    }
}
```

> 真实实现更复杂：包含 ESTIMATE 值传播、优先级队列、递增快照；详见 `block-stm/src/executor.rs` v1.x。

## 5. 演进与版本对比

| 版本 | 时间 | 关键变化 | 影响 |
| --- | --- | --- | --- |
| 主网 Launch | 2022-10-17 | DiemBFT v4 + 初版 Block-STM | 启动 |
| AIT-3 / v1.2 | 2023-Q1 | Gas Schedule 降本、fine-grained storage fee | 费用合理化 |
| AptosBFTv4 (Jolteon) | 2023-Q3 | 共识两阶段化 | 延迟 ↓ |
| Quorum Store | 2023-2024 | Mempool 与共识解耦 | 带宽上限突破 |
| Keyless / Passkey | 2024-Q1 | OIDC 直连、WebAuthn | 用户体验 |
| Indexer v2 / Hasura | 2024 | GraphQL 官方化 | 生态接入容易 |
| Shoal++ / Zaptos | 2025 | 流水线化 & 预投票 | 进一步降延迟 |

## 6. 实战示例

```bash
# 安装 Aptos CLI
brew install aptos  # 或 pip install aptos-cli

# 初始化 account + 水龙头
aptos init --network devnet
aptos account fund-with-faucet

# 部署 Move 模块
aptos move init --name hello_aptos
# 编辑 sources/hello.move 里的 entry fun 与 struct
aptos move publish --named-addresses hello_aptos=default

# 调用 entry
aptos move run \
  --function-id default::hello::say_hi \
  --args string:"Aptos"

# 查询资源
aptos account list --query resources --account default
```

预期输出：返回 `txn_hash`, `vm_status=Executed successfully`；查询可看到 `hello::Greeting { message: "Aptos" }`。

## 7. 安全与已知攻击

1. **2024 Aptos-core fuzz 漏洞（GitHub Advisory）**：社区研究员向 `aptos-vm` fuzz 中发现多条 native function panic 路径，通过治理热修复；复盘见 `aptos-labs/aptos-core` 的 Security advisory 专区。
2. **Thala 清算风暴（2024）**：Thala Lab 等 Aptos DeFi 项目曾因价格喂价 TWAP 窗口过短被 MEV 套利，启发 Aptos 预言机生态引入 Pyth 推送 + 验证窗口。
3. **理论攻击面**：① Block-STM 的 "热点账户" 被恶意 Tx 填充可使吞吐退化到串行，需要协议 / 应用层配合（L2 sharding / 池分裂）；② Keyless 依赖 OIDC provider 与 JWK 公钥轮换，若 provider 被攻破需通过 "recovery service"（未来规划）阻断。

## 8. 与同类方案对比

| 维度 | Aptos | Sui | Solana | Ethereum |
| --- | --- | --- | --- | --- |
| 状态模型 | Account + Resource | Object-centric | Account | Account/Storage |
| 并行执行 | **Block-STM（动态乐观）** | 静态 object lock | Sealevel（静态声明） | 串行 |
| 共识 | Jolteon (AptosBFTv4) | Mysticeti (DAG BFT) | PoH+TowerBFT | LMD-GHOST+FFG |
| 合约语言 | Aptos Move | Sui Move | Rust/C → BPF | Solidity/Vyper |
| 终局 | ~600 ms | ~500 ms | ~1 s 乐观 | 12 min 硬终局 |
| 账户抽象 | 原生（Keyless/Passkey） | zkLogin | 社区方案 | ERC-4337 |
| Global storage | 有（`move_to`） | 无（全部 Object） | N/A | N/A |
| 客户端数 | 1 | 1 | 2 | 5+ |

**Aptos Move vs Sui Move**：Aptos Move 的资源依托于账户层层存储（`global<T>(addr)` 查询），开发者直观但并行化要靠 Block-STM 动态挖掘；Sui Move 把资源提升为一等 Object 独立存在，编译期类型系统即表达并行度，但需要学习全新的 UID / Ownership 思维。两者的 MoveVM 底层相同，但 stdlib 与 framework 差异巨大，不能互通部署。

## 9. 延伸阅读

- **官方文档 / Spec**：
  - [Aptos Developer Docs](https://aptos.dev/)
  - [Aptos Foundation Whitepaper](https://aptosfoundation.org/whitepaper)
  - [Aptos Framework Reference](https://aptos.dev/reference/move)
- **核心论文**：
  - [Jolteon & Ditto (2021)](https://arxiv.org/abs/2106.10362)
  - [Block-STM (2022)](https://arxiv.org/abs/2203.06871)
  - [Shoal++ (2024)](https://arxiv.org/abs/2405.20488)
- **权威博客**：
  - [Aptos Labs Blog](https://aptoslabs.medium.com/)
  - [Messari: Aptos State of Network](https://messari.io/report)
  - [a16z crypto research on Move](https://a16zcrypto.com/)
- **视频**：
  - Avery Ching 在 Consensus 关于 Block-STM 的演讲
  - MoveCon 2024 Aptos workshop
- **规范 / AIP**：
  - [Aptos Improvement Proposals (AIP)](https://github.com/aptos-foundation/AIPs)

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 资源 | Resource | 线性类型 Move 值，挂载于账户 |
| 资源账户 | Resource Account | 通过种子派生的无私钥账户（合约 PDA 风格） |
| Quorum Store | Quorum Store | Batch + PoS 证明，解耦 Mempool |
| 并行执行 | Block-STM | 乐观并发 + MVMemory 验证回滚 |
| 无密钥账户 | Keyless Account | OIDC JWT + SNARK 派生地址 |
| 共识超时 | Timeout Certificate (TC) | Pacemaker 推进视图的聚合凭证 |
| 链上配置 | On-chain Config | `0x1::config` 治理参数 |

## 11. FAQ

以下问题来自与学习者的真实对话，聚焦 Aptos 最容易被 EVM / Sui 经验"想当然"的几个点：账户记账、Quorum Store 解耦、重复交易在多源 batch 下的处理。

### Q1：Aptos 的账户体系如何设计？怎么给用户记账？用户转账和调用合约在底层做什么？

Aptos 在这三件事上都与 EVM 拉开了距离。

**账户三元组（§2.1）**：

$$
\text{Account} = (\text{sequence\_number},\ \text{authentication\_key},\ \text{resources}: \text{TypeTag} \to \text{Resource})
$$

- **Address = SHA3-256(auth_key_preimage)\[:32\]**，首次建账后固定；**auth_key 可旋转**，address 不变（与 "地址=pubkey hash" 的 EVM 不同）。
- **Auth key** 是可插拔指针，支持 Ed25519 / Multi-Ed25519 / Secp256k1 / Passkey / **Keyless (OIDC + Groth16)**。
- **Sequence number** 全局递增，防重放，同时是并发提交的顺序闸门。
- **Resources** 按 `TypeTag (module::struct<T>)` 索引，**同一账户同一类型至多 1 份**——线性类型系统在运行时的体现。
- 两类特殊账户：**Resource Account**（由 `(creator, seed)` 派生，私钥由合约 `SignerCapability` 代持，类似 Solana PDA）、**Keyless Account**（OIDC JWT + SNARK，无私钥）。

**记账不是字段，是资源**——这是与 EVM 最本质的语义差异：

| | EVM | Aptos |
| --- | --- | --- |
| 余额表达 | `mapping(address=>uint256)` 合约存储槽 | `CoinStore<T>` / `FungibleStore` Resource 挂在账户下 |
| 类型约束 | `uint256` 可随意 `+=/-=` | `Coin<T>` 线性值，只能 `move / merge / extract` |
| 新用户首次收款 | 零成本 | 需创建 `CoinStore`，收 **存储押金**（删除时退回） |
| "凭空增发" | 合约逻辑错误即可发生 | **编译期就禁止**（线性类型） |

APT 原生余额查的就是 `0x{user}::0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>` 这个 key。2024 起新稳定币（USDT / USDC）已迁到 `0x1::fungible_asset` 标准，记账单元变为 `FungibleStore`，但"余额=账户下的 Resource"这个语义不变。

**转账 = 调用 Move entry function**（没有特殊的"转账指令"）：

```
EntryFunction: 0x1::aptos_account::transfer(to, amount)
   │
   ├─ ensure_account_exists(to)                              // 目标账户不存在就创建
   ├─ coin::register<AptosCoin>(to) if needed                // 没 CoinStore 就建
   ├─ let c = coin::withdraw<AptosCoin>(&sender_signer, amt) // 从 sender CoinStore 取线性 Coin
   └─ coin::deposit<AptosCoin>(to, c)                        // merge 进 receiver CoinStore
```

授权模型的根是 **`&signer`**：它只能由 VM 在 Tx 入口根据 `sender` 字段注入，不可伪造、不可复制；合约永远不能替用户签名，只能被"用户带着 signer 调用"。转账的读写集典型是 `{sender/CoinStore, to/CoinStore, sender/Account.seq}`——这也正是 Block-STM 用来判断并发冲突的 key。

**合约调用的端到端路径（§2.2 + §3.3）**：

1. SDK 构造 Tx（`sender / seq / payload(EntryFunction|Script) / gas`）+ 签名
2. REST 提交 → shared mempool → Quorum Store 打 batch
3. Jolteon 共识 2 轮 QC 定序
4. Block-STM 并行执行
5. AptosDB / Jellyfish Merkle 写状态

VM 执行时：**Prologue** 校验签名 / `tx.seq == account.seq` / gas 可预付；**Body** 解释 bytecode，`move_to / move_from / borrow_global_mut` 是 Block-STM 记录读写集的唯一入口；**Epilogue** 结算 gas、`sequence_number += 1`（**即使 Tx abort 也会 +1 并扣 gas**，与 EVM 相同）。

### Q2：§2.3 里提到的 Quorum Store 具体原理是什么？如何做到 mempool 与共识解耦？

核心思想来自 [Narwhal](https://arxiv.org/abs/2105.11827)：**传播数据**与**定序数据**是两个可以独立并行的子问题，把它们合在同一条消息里做，就是 Leader 带宽瓶颈的根源。

**原来的瓶颈**——HotStuff / DiemBFT 是 Leader-broadcast 模式，Leader 打包完整 Tx 广播给所有 validator，上行带宽 = `N × |tx| × (n−1)` 全压在 Leader 单机上，follower 上行完全空转，块大小与共识延迟强耦合。

**Quorum Store 的三条规则**：

1. **每个 validator 都当数据源**——并行打 batch、并行广播、并行收签名。集群聚合上行带宽 ≈ n × Leader 单机上行。
2. **共识只流转"数据收据"**（Proof of Store, ~200B），Tx 数据不进共识消息。
3. **只要 PoS 成立，执行时一定拉得到数据**——因为 PoS 证明至少 f+1 诚实节点已持久化 batch。

**Batch 与 PoS 构造过程**：

```
Batch { author, epoch, batch_id, expiration, txns, digest }

Validator A                        Validators B..N
───────────                        ─────────────────
1. 按 gas price 选 Tx 打 batch
2. 本地持久化 → 广播     ────────►  3. 校验 + 持久化
                                    4. 对 digest BLS 签名返回
5. 聚合 2f+1 stake-weighted 签名
   → ProofOfStore {
       digest, expiration, author,
       aggregated_sig
     }
```

PoS 的核心定理（承袭 Narwhal）：**2f+1 签名里至多 f 个 byzantine，剩下的 f+1 诚实节点必定已落盘**——这是 Leader 后续只引用 digest 的安全基础。`expiration` 是经济机制：过期后诚实节点可 GC，避免 batch 数据永久占存储。

**Leader 提案的块体几乎没有 Tx**：

```
Block {
    round, epoch, author, parent_qc,
    payload: Vec<ProofOfStore>,    // ← 一堆引用
}
```

块体压缩比约 **1000×**（一条 PoS ≈ 200B，承载数百到数千条 Tx）。Validator 投票前只校验 PoS 聚合签名合法、未过期、未被前序块引用过——**不需要看 Tx 数据内容**。

**执行阶段按引用回填**：

```
for pos in block.payload:
    batch = local_batch_store.get(pos.digest)
    if batch is None:
        batch = fetch_from_peers(pos)    // f+1 诚实节点必有
    execute_batch(batch)                 // 交给 Block-STM
```

**解耦映射表**：

| 解耦维度 | 原模型 | Quorum Store |
| --- | --- | --- |
| 数据传播 vs 定序 | 同一消息 | 两套独立子协议 |
| 带宽利用 | Leader 单点 | n 节点并行 |
| 块大小 ↔ 延迟 | 强耦合 | 解耦（块只装 digest） |
| 共识消息复杂度 | O(n·\|block\|) | O(n·\|digest set\|) |
| Leader 切换成本 | 重广播 mempool 视图 | 复用已有 PoS 队列 |
| Mempool 角色 | 共识直接消费 | **退化为 QS 的上游缓冲池**，共识完全不看它 |

Aptos Labs 实测纯共识层 **12× TPS**、端到端 **3× TPS** 提升。实现位于 `consensus/src/quorum_store/`，作为 on-chain config 可关（灰度上线用过此开关）。

**血缘**：Quorum Store ≈ Narwhal 的 batch+certificate 数据层 + Jolteon 线性定序。Sui 走 Mysticeti 全 DAG 路线（数据+定序一体），Aptos 只吸收数据层，不吸收 DAG 定序——因为 Block-STM 的"按块内 idx 严格串行等价"前提不兼容 DAG 式拓扑定序。

### Q3：每个 validator 并行打 batch，不同 batch 里可能有相同的 Tx，这种情况怎么处理？

**Aptos 的选择：允许重叠发生，分层削减**。理由是强互斥需要 validator 间事先协商"谁打哪一片"，会把 Quorum Store 再次退化成需协调的子协议，违背"n 个并行数据源"的初衷。取而代之是 4 层兜底：

**L1 — Mempool 侧的概率错峰**

每条 Tx 在 mempool 里带 `ready_time` 与 `broadcast_peers`。`BatchGenerator` 抽 Tx 时：

- 优先抽本节点直接从客户端收到的 Tx（primary source）
- 对从 peer 收到的 Tx 加小延迟（让原始来源节点先打）

概率性降低期望重叠率，不是硬保证。

**L2 — PoS 队列按 TxnSummary 去重**

`BatchProofQueue`（`consensus/src/quorum_store/batch_proof_queue.rs`）对入队的每个 PoS 解析 batch 里每条 Tx 为：

```
TxnSummary { sender, replay_protector (= seq), hash, expiration }
```

去重单元是 `(sender, replay_protector, hash)`——与装在哪个 batch 无关。Leader 在 `ProofManager::pull_proofs()` 组装 block payload 时：

1. 按 `(gas_bucket DESC, batch_id DESC)` 扫 PoS 队列（高 gas 优先、新 batch 优先，保证确定性）
2. 每个 PoS 的**有效条数** = `txn_count − 已在更早选中 PoS 里出现过的 txn 数`
3. 按有效条数占 block payload 预算
4. 重叠 100% 的 PoS 有效条数 = 0 → 被跳过

**L3 — 跨 block 的 committed / pending 排除**

ProofManager 还维护 `committed_txn_summaries`（近期已提交）和 `pending_proposed_txn_summaries`（已进 pipeline 但未提交）。过滤管线：

```
candidate_pos_queue
  .filter(not in committed_txn_summaries)
  .filter(not in pending_proposed_txn_summaries)
  .dedup_by(txn_summary)
  .take(payload_budget)
```

诚实 Leader 下，同一笔 Tx 至多出现在一个最终提交 block 的有效执行集合里。

**L4 — 终极闸门：`sequence_number`**

前三层都是"优化"，**正确性保证来自 VM prologue**：

| 情况 | 结果 |
| --- | --- |
| `tx.seq == account.seq` | 正常执行，seq += 1 |
| `tx.seq < account.seq`（已被副本执行） | `SEQUENCE_NUMBER_TOO_OLD`，丢弃，**不写状态** |
| `tx.seq > account.seq` | `SEQUENCE_NUMBER_TOO_NEW`，丢弃 |

即使同一笔 (Alice, seq=42) 同时被两个 PoS 引用、都进了 block：

1. Block-STM 按 block 内 idx 顺序，第一个 Tx_42 成功执行，`Alice.seq` 变成 43
2. 第二个 Tx_42 prologue 就被 `SEQUENCE_NUMBER_TOO_OLD` 拒绝，无任何 Write Set
3. Validation 阶段也会看到读到的 `Alice.seq=43` 与期望的 42 不符，直接丢弃

**边界场景**：

| 场景 | 结果 |
| --- | --- |
| A、B batch 完全相同（同序同 Tx） | `digest` 一样 → 其实是同一个 PoS，无冗余 |
| A、B 有 80% 重叠但尾部不同 | digest 不同；Leader 按高 gas/新 batch 选一个，另一个贡献 ≈20% 有效条数 |
| 恶意 Leader 硬塞两个重叠 PoS | 块能过共识（PoS 本身合法），执行阶段靠 seq 淘汰重复；**攻击者只是在浪费 block 空间，无法双花** |
| 重叠 PoS 中永远不执行的 Tx | 由 batch `expiration` GC 掉 |

**一句话**：Aptos 用 4 层把"重复"从 mempool 一路削到执行层，前三层省算力、省 block 空间，最后由账户 `sequence_number` 保证账本一致性——这正是敢让 n 个 validator 并行打 batch 而不必事先协调的底气。

---

*Last verified: 2026-04-25*
