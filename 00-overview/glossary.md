---
title: 术语表（中英对照）
module: 00-overview
priority: P0
status: DRAFT
word_count_target: 持续更新
last_verified: 2026-04-23
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
| [Merkle Tree](http://www.merkle.com/papers/Thesis1979.pdf) | 默克尔树 | |

## B. 共识

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Consensus | 共识 | |
| [Proof of Work (PoW)](https://bitcoin.org/bitcoin.pdf) | 工作量证明 | |
| [Proof of Stake (PoS)](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/) | 权益证明 | |
| Delegated PoS (DPoS) | 委托权益证明 | |
| [BFT / Byzantine Fault Tolerance](https://pmg.csail.mit.edu/papers/osdi99.pdf) | 拜占庭容错 | |
| Validator | 验证者 | 不用"验证人" |
| [Slashing](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/rewards-and-penalties/) | 罚没 | |
| [Nothing-at-Stake](https://vitalik.eth.limo/general/2017/12/31/pos_design.html) | 无利害关系问题 | |
| Long-range Attack | 远程攻击 | |
| [51% Attack](https://www.investopedia.com/terms/1/51-attack.asp) | 51% 攻击 | |
| [Proof of History (PoH)](https://solana.com/solana-whitepaper.pdf) | 历史证明 | Solana 特有 |
| [Tower BFT](https://docs.solanalabs.com/consensus/commitments) | Tower BFT | Solana 共识 |
| [LMD-GHOST](https://eth2book.info/latest/part2/consensus/lmd_ghost/) | LMD-GHOST | Ethereum PoS 分叉选择 |
| [Casper FFG](https://arxiv.org/abs/1710.09437) | Casper FFG | Ethereum 终局工具 |

## C. 账户与记账

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| [UTXO](https://en.wikipedia.org/wiki/Unspent_transaction_output) | 未花费交易输出 | Unspent Transaction Output |
| [Account Model](https://ethereum.org/en/developers/docs/accounts/) | 账户模型 | |
| [EOA](https://ethereum.org/en/developers/docs/accounts/#externally-owned-accounts-and-key-pairs) | 外部账户 | Externally Owned Account |
| [Contract Account](https://ethereum.org/en/developers/docs/accounts/#contract-accounts) | 合约账户 | |
| State | 状态 | |
| [World State](https://ethereum.github.io/yellowpaper/paper.pdf) | 全局状态 | Ethereum 术语 |
| [State Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/) | 状态树 | |
| [Storage Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/) | 存储树 | |
| Receipt | 收据 | |

## D. 密码学

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Hash Function | 哈希函数 | |
| [SHA-256](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf) | SHA-256 | 保留原文 |
| [Keccak-256](https://keccak.team/keccak.html) | Keccak-256 | |
| [ECDSA](https://www.secg.org/sec1-v2.pdf) | 椭圆曲线数字签名 | 曲线：[secp256k1](https://www.secg.org/sec2-v2.pdf) / secp256r1 |
| [Schnorr Signature](https://bips.dev/340/) | Schnorr 签名 | |
| [Ed25519](https://ed25519.cr.yp.to/) | Ed25519 | 保留原文 |
| Public Key | 公钥 | |
| Private Key | 私钥 | |
| Address | 地址 | |
| [BIP-39 Mnemonic](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) | 助记词 | |
| [HD Wallet](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) | 分层确定性钱包 | Hierarchical Deterministic |
| [Zero-Knowledge Proof (ZKP)](https://ethereum.org/en/zero-knowledge-proofs/) | 零知识证明 | |
| [SNARK](https://eprint.iacr.org/2013/279) / [STARK](https://eprint.iacr.org/2018/046) | SNARK / STARK | 保留原文 |
| [Fully Homomorphic Encryption (FHE)](https://en.wikipedia.org/wiki/Homomorphic_encryption) | 全同态加密 | |
| [Multi-Party Computation (MPC)](https://en.wikipedia.org/wiki/Secure_multi-party_computation) | 多方安全计算 | |

## E. 虚拟机与合约

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Smart Contract | 智能合约 | |
| [EVM](https://ethereum.org/en/developers/docs/evm/) | EVM / 以太坊虚拟机 | 保留缩写 |
| [SVM](https://solana.com/docs/core) | SVM / Solana 虚拟机 | |
| [MoveVM](https://move-language.github.io/move/) | MoveVM | |
| [Opcode](https://www.evm.codes/) | 操作码 | |
| Gas | Gas | 保留原文 |
| Gas Limit / Gas Price | Gas 上限 / Gas 价格 | |
| [Base Fee / Priority Fee](https://eips.ethereum.org/EIPS/eip-1559) | 基础费 / 优先费 | EIP-1559 |
| Call / Delegatecall / Staticcall | call / delegatecall / staticcall | 保留原文 |
| [Proxy Pattern](https://docs.openzeppelin.com/contracts/4.x/api/proxy) | 代理模式 | [UUPS](https://eips.ethereum.org/EIPS/eip-1822) / [Transparent](https://eips.ethereum.org/EIPS/eip-1967) / [Diamond](https://eips.ethereum.org/EIPS/eip-2535) |
| [ABI](https://docs.soliditylang.org/en/latest/abi-spec.html) | 应用二进制接口 | 保留缩写 |

## F. DeFi

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| DEX | 去中心化交易所 | 保留缩写 |
| [AMM](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) | 自动做市商 | Automated Market Maker |
| Liquidity Pool | 流动性池 | |
| LP Token | LP 代币 / 流动性凭证 | |
| [Impermanent Loss](https://academy.binance.com/en/articles/impermanent-loss-explained) | 无常损失 | |
| Slippage | 滑点 | |
| [TVL](https://defillama.com) | 总锁仓价值 | 保留缩写 |
| Lending / Borrowing | 借贷 | |
| Over-Collateralization | 超额抵押 | |
| Liquidation | 清算 | |
| [Flash Loan](https://docs.aave.com/developers/guides/flash-loans) | 闪电贷 | |
| Liquid Staking Token (LST) | 流动性质押代币 | [Lido stETH](https://lido.fi) / [Rocket Pool rETH](https://rocketpool.net) |
| [Restaking](https://www.eigenlayer.xyz) | 再质押 | EigenLayer |
| [MEV](https://ethereum.org/en/developers/docs/mev/) | 最大可提取价值 | Maximal Extractable Value |

## G. L2 / 跨链

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| [Rollup](https://ethereum.org/en/developers/docs/scaling/#rollups) | Rollup | 保留原文 |
| [Optimistic Rollup](https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/) | 乐观 Rollup | |
| [ZK Rollup](https://ethereum.org/en/developers/docs/scaling/zk-rollups/) | ZK Rollup | |
| Fraud Proof | 欺诈证明 | |
| Validity Proof | 有效性证明 | |
| [Data Availability (DA)](https://ethereum.org/en/developers/docs/data-availability/) | 数据可用性 | |
| [Sequencer](https://l2beat.com) | 排序器 | |
| Bridge | 跨链桥 | |
| [HTLC](https://en.bitcoin.it/wiki/Hash_Time_Locked_Contracts) | 哈希时间锁合约 | Hash Time-Locked Contract |
| Light Client | 轻客户端 | |
| [IBC](https://ibc.cosmos.network) | 区块链间通信协议 | Cosmos |

## H. Token 标准

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| Fungible Token | 同质化代币 | [ERC-20](https://eips.ethereum.org/EIPS/eip-20) |
| Non-Fungible Token (NFT) | 非同质化代币 | [ERC-721](https://eips.ethereum.org/EIPS/eip-721) / [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) |
| Semi-Fungible | 半同质化 | [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) |
| Tokenization | 通证化 / 代币化 | |
| [RWA](https://app.rwa.xyz/) | 现实世界资产 | Real World Asset |
| [Stablecoin](https://defillama.com/stablecoins) | 稳定币 | |

## I. 其他

| 英文 | 中文 | 说明 |
| --- | --- | --- |
| DApp | 去中心化应用 | |
| [DAO](https://ethereum.org/en/dao/) | 去中心化自治组织 | |
| [Oracle](https://ethereum.org/en/developers/docs/oracles/) | 预言机 | |
| Indexer | 索引器 | |
| [RPC Node](https://ethereum.org/en/developers/docs/apis/json-rpc/) | RPC 节点 | |
| [Subgraph](https://thegraph.com/docs/en/subgraphs/developing/introduction/) | 子图 | The Graph |
| KYC / KYT / KYA | 客户/交易/地址身份识别 | |
| [TEE](https://en.wikipedia.org/wiki/Trusted_execution_environment) | 可信执行环境 | Trusted Execution Environment |

---

*收录范围持续扩充。新增术语请按字母分组并给出首次出现的文章路径。*
