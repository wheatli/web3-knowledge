---
title: 术语表（中英对照）
module: 00-overview
priority: P0
status: DRAFT
word_count_target: 持续更新
last_verified: 2026-04-22
---

# 术语表（中英对照）

> 本表为全库统一术语锚点。写作时遇到这里未收录的术语，须同步补入并给出至少一条权威出处。

## A. 基础概念

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Block | 区块 | |
| Block Header | 区块头 | |
| Block Height | 区块高度 | |
| Genesis Block | 创世区块 | |
| Transaction (Tx) | 交易 | |
| Nonce | 随机数 / 防重放计数 | PoW 中指难度随机数；账户模型中指防重放计数 |
| Mempool | 内存池 / 交易池 | |
| Finality | 终局性 | 概率终局（Probabilistic）vs 确定性终局（Deterministic） |
| Fork | 分叉 | 软分叉（Soft Fork）/ 硬分叉（Hard Fork） |
| Chain Reorg | 链重组 | |
| Merkle Tree | 默克尔树 | |

## B. 共识

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Consensus | 共识 | |
| Proof of Work (PoW) | 工作量证明 | |
| Proof of Stake (PoS) | 权益证明 | |
| Delegated PoS (DPoS) | 委托权益证明 | |
| BFT / Byzantine Fault Tolerance | 拜占庭容错 | |
| Validator | 验证者 | 不用"验证人" |
| Slashing | 罚没 | |
| Nothing-at-Stake | 无利害关系问题 | |
| Long-range Attack | 远程攻击 | |
| 51% Attack | 51% 攻击 | |
| Proof of History (PoH) | 历史证明 | Solana 特有 |
| Tower BFT | Tower BFT | Solana 共识 |
| LMD-GHOST | LMD-GHOST | Ethereum PoS 分叉选择 |
| Casper FFG | Casper FFG | Ethereum 终局工具 |

## C. 账户与记账

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| UTXO | 未花费交易输出 | Unspent Transaction Output |
| Account Model | 账户模型 | |
| EOA | 外部账户 | Externally Owned Account |
| Contract Account | 合约账户 | |
| State | 状态 | |
| World State | 全局状态 | Ethereum 术语 |
| State Trie | 状态树 | |
| Storage Trie | 存储树 | |
| Receipt | 收据 | |

## D. 密码学

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Hash Function | 哈希函数 | |
| SHA-256 | SHA-256 | 保留原文 |
| Keccak-256 | Keccak-256 | |
| ECDSA | 椭圆曲线数字签名 | 曲线：secp256k1 / secp256r1 |
| Schnorr Signature | Schnorr 签名 | |
| Ed25519 | Ed25519 | 保留原文 |
| Public Key | 公钥 | |
| Private Key | 私钥 | |
| Address | 地址 | |
| BIP-39 Mnemonic | 助记词 | |
| HD Wallet | 分层确定性钱包 | Hierarchical Deterministic |
| Zero-Knowledge Proof (ZKP) | 零知识证明 | |
| SNARK / STARK | SNARK / STARK | 保留原文 |
| Fully Homomorphic Encryption (FHE) | 全同态加密 | |
| Multi-Party Computation (MPC) | 多方安全计算 | |

## E. 虚拟机与合约

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Smart Contract | 智能合约 | |
| EVM | EVM / 以太坊虚拟机 | 保留缩写 |
| SVM | SVM / Solana 虚拟机 | |
| MoveVM | MoveVM | |
| Opcode | 操作码 | |
| Gas | Gas | 保留原文 |
| Gas Limit / Gas Price | Gas 上限 / Gas 价格 | |
| Base Fee / Priority Fee | 基础费 / 优先费 | EIP-1559 |
| Call / Delegatecall / Staticcall | call / delegatecall / staticcall | 保留原文 |
| Proxy Pattern | 代理模式 | UUPS / Transparent / Diamond |
| ABI | 应用二进制接口 | 保留缩写 |

## F. DeFi

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| DEX | 去中心化交易所 | 保留缩写 |
| AMM | 自动做市商 | Automated Market Maker |
| Liquidity Pool | 流动性池 | |
| LP Token | LP 代币 / 流动性凭证 | |
| Impermanent Loss | 无常损失 | |
| Slippage | 滑点 | |
| TVL | 总锁仓价值 | 保留缩写 |
| Lending / Borrowing | 借贷 | |
| Over-Collateralization | 超额抵押 | |
| Liquidation | 清算 | |
| Flash Loan | 闪电贷 | |
| Liquid Staking Token (LST) | 流动性质押代币 | Lido stETH / RPL rETH |
| Restaking | 再质押 | EigenLayer |
| MEV | 最大可提取价值 | Maximal Extractable Value |

## G. L2 / 跨链

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Rollup | Rollup | 保留原文 |
| Optimistic Rollup | 乐观 Rollup | |
| ZK Rollup | ZK Rollup | |
| Fraud Proof | 欺诈证明 | |
| Validity Proof | 有效性证明 | |
| Data Availability (DA) | 数据可用性 | |
| Sequencer | 排序器 | |
| Bridge | 跨链桥 | |
| HTLC | 哈希时间锁合约 | Hash Time-Locked Contract |
| Light Client | 轻客户端 | |
| IBC | 区块链间通信协议 | Cosmos |

## H. Token 标准

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Fungible Token | 同质化代币 | ERC-20 |
| Non-Fungible Token (NFT) | 非同质化代币 | ERC-721 / ERC-1155 |
| Semi-Fungible | 半同质化 | ERC-1155 |
| Tokenization | 通证化 / 代币化 | |
| RWA | 现实世界资产 | Real World Asset |
| Stablecoin | 稳定币 | |

## I. 其他

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| DApp | 去中心化应用 | |
| DAO | 去中心化自治组织 | |
| Oracle | 预言机 | |
| Indexer | 索引器 | |
| RPC Node | RPC 节点 | |
| Subgraph | 子图 | The Graph |
| KYC / KYT / KYA | 客户/交易/地址身份识别 | |
| TEE | 可信执行环境 | Trusted Execution Environment |

---

*收录范围持续扩充。新增术语请按字母分组并给出首次出现的文章路径。*
