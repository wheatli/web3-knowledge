# Web3 行业知识库

> 一份系统化、深度化、**中文为主 + 保留英文术语** 的 Web3 技术知识库。每个知识点按统一模板深入剖析（原理、架构、代码、演进、安全、对比），所有断言需 Tier 1 官方源 + Tier 2/3 交叉校验。

**范围**：基础设施（公链/共识/跨链/预言机/L2/DA）、钱包、智能合约、DApp（DeFi/NFT/RWA/稳定币）、安全、第三方服务、隐私（ZKP/FHE/MPC）。

**维护规范**：见 `CONTRIBUTING.md`；引用源分级见 `SOURCES.md`；文章模板见 `TEMPLATE.md`；进度看板见 `PROGRESS.md`。

---

## 快速导航

### 00. 全景与历史

- [Web3 行业全景图](00-overview/web3-landscape.md)
- [区块链发展史](00-overview/blockchain-history.md)
- [术语表（中英对照）](00-overview/glossary.md)

### 01. 基础设施层

#### 公链

- [公链总览](01-infrastructure/public-chains/_index.md)
- [Bitcoin](01-infrastructure/public-chains/bitcoin.md) · [Ethereum](01-infrastructure/public-chains/ethereum.md) · [Solana](01-infrastructure/public-chains/solana.md)
- [Sui](01-infrastructure/public-chains/sui.md) · [Aptos](01-infrastructure/public-chains/aptos.md) · [Avalanche](01-infrastructure/public-chains/avalanche.md)
- [Cosmos](01-infrastructure/public-chains/cosmos.md) · [Polkadot](01-infrastructure/public-chains/polkadot.md) · [Near](01-infrastructure/public-chains/near.md)
- [TON](01-infrastructure/public-chains/ton.md) · [Hyperliquid](01-infrastructure/public-chains/hyperliquid.md) · [BNB Chain](01-infrastructure/public-chains/bnb-chain.md) · [Tron](01-infrastructure/public-chains/tron.md)

#### 共识协议

- [共识总览](01-infrastructure/consensus/_index.md)
- [PoW](01-infrastructure/consensus/pow.md) · [PoS](01-infrastructure/consensus/pos.md) · [DPoS](01-infrastructure/consensus/dpos.md)
- [BFT 家族](01-infrastructure/consensus/bft-family.md) · [PoH](01-infrastructure/consensus/poh.md) · [Avalanche Consensus](01-infrastructure/consensus/avalanche-consensus.md)
- [概率终局 vs 确定性终局](01-infrastructure/consensus/nakamoto-vs-finality.md) · [质押经济学](01-infrastructure/consensus/staking.md)

#### 记账模型

- [UTXO](01-infrastructure/ledger-model/utxo.md) · [Account](01-infrastructure/ledger-model/account.md) · [Hybrid / Object](01-infrastructure/ledger-model/hybrid-models.md)

#### 跨链

- [跨链总览](01-infrastructure/cross-chain/_index.md)
- [协议原理（HTLC/Relay/Light Client）](01-infrastructure/cross-chain/protocols-htlc-relay.md)
- [桥分类](01-infrastructure/cross-chain/bridge-taxonomy.md)
- [LayerZero](01-infrastructure/cross-chain/layerzero.md) · [Wormhole](01-infrastructure/cross-chain/wormhole.md) · [Axelar](01-infrastructure/cross-chain/axelar.md) · [Chainlink CCIP](01-infrastructure/cross-chain/chainlink-ccip.md) · [IBC](01-infrastructure/cross-chain/ibc.md) · [Hop/Across/Synapse](01-infrastructure/cross-chain/hop-across-synapse.md)
- [桥安全事件复盘](01-infrastructure/cross-chain/bridge-security-incidents.md)

#### 预言机

- [总览](01-infrastructure/oracle/_index.md) · [Chainlink](01-infrastructure/oracle/chainlink.md) · [Pyth](01-infrastructure/oracle/pyth.md) · [API3](01-infrastructure/oracle/api3.md) · [UMA](01-infrastructure/oracle/uma.md) · [RedStone](01-infrastructure/oracle/redstone.md)
- [预言机设计](01-infrastructure/oracle/oracle-design.md) · [预言机安全](01-infrastructure/oracle/oracle-security.md)

#### Layer2 与 DA

- [L2 总览](01-infrastructure/layer2/_index.md)
- [Optimistic Rollup](01-infrastructure/layer2/optimistic-rollup.md) · [ZK Rollup](01-infrastructure/layer2/zk-rollup.md)
- [Arbitrum](01-infrastructure/layer2/arbitrum.md) · [Optimism](01-infrastructure/layer2/optimism.md) · [Base](01-infrastructure/layer2/base.md) · [zkSync](01-infrastructure/layer2/zksync.md) · [Starknet](01-infrastructure/layer2/starknet.md) · [Scroll](01-infrastructure/layer2/scroll.md) · [Linea](01-infrastructure/layer2/linea.md) · [Polygon zkEVM](01-infrastructure/layer2/polygon-zkevm.md)
- [Validium / Volition](01-infrastructure/layer2/validium-volition.md) · [Plasma](01-infrastructure/layer2/plasma.md) · [State Channels](01-infrastructure/layer2/state-channels.md)
- [Celestia](01-infrastructure/data-availability/celestia.md) · [EigenDA](01-infrastructure/data-availability/eigenda.md) · [Avail](01-infrastructure/data-availability/avail.md) · [Ethereum Danksharding](01-infrastructure/data-availability/ethereum-danksharding.md)
- [模块化区块链](01-infrastructure/modular-blockchain.md)

### 02. 钱包层

- [密码学基础](02-wallet/cryptography-basics.md) · [HD 钱包 / BIP](02-wallet/hd-wallet-bip.md)
- [全托管（交易所）](02-wallet/custodial.md) · [自托管 EOA](02-wallet/self-custody-eoa.md) · [MPC 钱包](02-wallet/mpc-wallet.md) · [智能合约钱包](02-wallet/smart-contract-wallet.md)
- [账户抽象（EIP-4337/7702）](02-wallet/account-abstraction.md) · [社交恢复](02-wallet/social-recovery.md) · [硬件钱包](02-wallet/hardware-wallet.md) · [钱包安全](02-wallet/wallet-security.md)

### 03. 智能合约

- [智能合约概览](03-smart-contract/_index.md)
- EVM：[架构](03-smart-contract/evm/evm-architecture.md) · [Solidity 语法](03-smart-contract/evm/solidity-syntax.md) · [Solidity 模式](03-smart-contract/evm/solidity-patterns.md) · [可升级性](03-smart-contract/evm/upgradeability.md) · [Vyper](03-smart-contract/evm/vyper.md) · [Yul/Huff](03-smart-contract/evm/yul-huff.md) · [开发工具链](03-smart-contract/evm/dev-toolchain.md)
- Solana：[Sealevel Runtime](03-smart-contract/solana/sealevel-runtime.md) · [账户模型](03-smart-contract/solana/account-model.md) · [Anchor](03-smart-contract/solana/anchor-framework.md) · [Program 开发](03-smart-contract/solana/program-development.md)
- Move：[Move 语言](03-smart-contract/move/move-language.md) · [Aptos Move](03-smart-contract/move/aptos-move.md) · [Sui Move](03-smart-contract/move/sui-move.md)
- [Fuel Sway](03-smart-contract/fuel-sway.md) · [CosmWasm](03-smart-contract/cosmwasm.md) · [ink!](03-smart-contract/ink-substrate.md) · [VM 对比](03-smart-contract/cross-vm-comparison.md)

### 04. DApp 生态

- Token 标准：[ERC-20/ICO](04-dapp/token-standards/erc20-ico.md) · [ERC-721/1155](04-dapp/token-standards/erc721-1155-nft.md) · [ERC-3643/RWA](04-dapp/token-standards/erc3643-rwa.md) · [ERC-4626](04-dapp/token-standards/erc4626-vault.md) · [ERC-6551](04-dapp/token-standards/erc6551-tba.md) · [其他](04-dapp/token-standards/other-standards.md)
- DeFi：[总览](04-dapp/defi/_index.md) · [Uniswap 演进](04-dapp/defi/uniswap-evolution.md) · [Curve](04-dapp/defi/curve-stableswap.md) · [Balancer](04-dapp/defi/balancer.md) · [聚合器](04-dapp/defi/dex-aggregators.md) · [Aave](04-dapp/defi/lending-aave.md) · [Compound](04-dapp/defi/lending-compound.md) · [Morpho](04-dapp/defi/lending-morpho.md) · [Spark/Maker](04-dapp/defi/lending-spark-maker.md) · [dYdX/GMX](04-dapp/defi/perps-dydx-gmx.md) · [Hyperliquid Perps](04-dapp/defi/perps-hyperliquid.md) · [期权](04-dapp/defi/options-lyra-ribbon.md) · [收益](04-dapp/defi/yield-yearn-pendle.md) · [LST](04-dapp/defi/liquid-staking.md) · [Restaking/EigenLayer](04-dapp/defi/restaking-eigenlayer.md) · [MEV/Flashbots](04-dapp/defi/mev-flashbots.md)
- NFT：[历史](04-dapp/nft/nft-history.md) · [市场](04-dapp/nft/marketplaces.md) · [版税与标准](04-dapp/nft/royalty-and-standards.md) · [NFT-Fi](04-dapp/nft/nftfi.md)
- 稳定币：[总览](04-dapp/stablecoin/_index.md) · [USDT](04-dapp/stablecoin/usdt-tether.md) · [USDC](04-dapp/stablecoin/usdc-circle.md) · [PYUSD/Paxos](04-dapp/stablecoin/pyusd-paxos.md) · [DAI/Maker](04-dapp/stablecoin/dai-maker.md) · [Frax](04-dapp/stablecoin/frax.md) · [USDe/Ethena](04-dapp/stablecoin/usde-ethena.md) · [算法稳定币史](04-dapp/stablecoin/algo-stable-history.md) · [监管](04-dapp/stablecoin/regulatory-landscape.md)
- RWA：[总览](04-dapp/rwa/_index.md) · [国债代币化](04-dapp/rwa/tokenization-treasury.md) · [私募信贷](04-dapp/rwa/private-credit.md) · [房地产](04-dapp/rwa/real-estate.md) · [合约设计](04-dapp/rwa/rwa-contract-design.md)
- Social/DAO：[DAO 治理](04-dapp/social-dao/dao-governance.md) · [ENS](04-dapp/social-dao/ens.md) · [Lens/Farcaster](04-dapp/social-dao/lens-farcaster.md) · [SBT](04-dapp/social-dao/sbt-soulbound.md)
- [游戏与元宇宙](04-dapp/gaming-metaverse.md)

### 05. 安全

- [总览](05-security/_index.md) · [链共识安全](05-security/chain-consensus-security.md) · [合约漏洞](05-security/contract-vulnerabilities.md) · [审计方法论](05-security/audit-methodology.md) · [形式化验证](05-security/formal-verification.md) · [静态分析](05-security/static-tools.md) · [模糊测试](05-security/fuzzing-tools.md) · [桥攻击复盘](05-security/bridge-hack-postmortems.md) · [DeFi 漏洞复盘](05-security/defi-exploit-postmortems.md) · [最佳实践](05-security/security-best-practices.md)

### 06. 第三方服务

- RPC/节点：[rpc-node-providers](06-third-party/rpc-node-providers.md)
- 索引：[The Graph](06-third-party/indexing/the-graph.md) · [Subsquid](06-third-party/indexing/subsquid.md) · [Goldsky](06-third-party/indexing/goldsky.md)
- 分析：[Dune](06-third-party/analytics/dune.md) · [Nansen](06-third-party/analytics/nansen.md) · [Arkham](06-third-party/analytics/arkham.md) · [DefiLlama](06-third-party/analytics/defillama.md)
- KYT/KYA：[Chainalysis](06-third-party/kyt-kya/chainalysis.md) · [Elliptic](06-third-party/kyt-kya/elliptic.md) · [TRM Labs](06-third-party/kyt-kya/trm-labs.md)
- 审计：[CertiK](06-third-party/audit-firms/certik.md) · [SlowMist](06-third-party/audit-firms/slowmist.md) · [Trail of Bits](06-third-party/audit-firms/trail-of-bits.md) · [OpenZeppelin](06-third-party/audit-firms/openzeppelin.md) · [ChainSecurity](06-third-party/audit-firms/chainsecurity.md)
- 浏览器：[Etherscan](06-third-party/explorer/etherscan.md) · [Solscan](06-third-party/explorer/solscan.md) · [Blockscout](06-third-party/explorer/blockscout.md)
- 开发基础设施：[Thirdweb/Moralis](06-third-party/dev-infra/thirdweb-moralis.md) · [Tenderly](06-third-party/dev-infra/tenderly.md) · [Pimlico](06-third-party/dev-infra/pimlico-paymaster.md)

### 07. 隐私

- ZKP：[总览](07-privacy/zkp/_index.md) · [基础](07-privacy/zkp/zk-fundamentals.md) · [ZK-SNARK](07-privacy/zkp/zk-snark.md) · [ZK-STARK](07-privacy/zkp/zk-stark.md) · [Bulletproofs](07-privacy/zkp/bulletproofs.md) · [Halo/Nova](07-privacy/zkp/halo-nova.md) · [zkVM](07-privacy/zkp/zk-vm.md) · [应用](07-privacy/zkp/zk-applications.md)
- FHE：[总览](07-privacy/fhe/_index.md) · [半同态](07-privacy/fhe/partial-he.md) · [全同态](07-privacy/fhe/fully-he.md) · [链上 FHE](07-privacy/fhe/fhe-in-blockchain.md)
- MPC：[秘密分享](07-privacy/mpc/secret-sharing.md) · [MPC 协议](07-privacy/mpc/mpc-protocols.md) · [门限签名](07-privacy/mpc/threshold-signatures.md)
- [混币与隐私币](07-privacy/mixer-privacy-coins.md) · [可信执行环境](07-privacy/trusted-execution.md)

### 08. 安全事件大档案（时间线）

- [档案总览 / 时间线索引](08-security-incidents/_index.md)（2011–2026 YTD，~55 篇独立事件复盘）
- 2011–2019：[Mt.Gox 首破](08-security-incidents/2011-mtgox-first-breach.md) · [Linode/Bitcoinica](08-security-incidents/2012-linode-bitcoinica.md) · [Mt.Gox 崩盘](08-security-incidents/2014-mtgox-collapse.md) · [Bitstamp](08-security-incidents/2015-bitstamp.md) · [The DAO](08-security-incidents/2016-the-dao.md) · [Bitfinex](08-security-incidents/2016-bitfinex.md) · [Parity Hack](08-security-incidents/2017-parity-wallet-hack.md) · [Parity Freeze](08-security-incidents/2017-parity-freeze.md) · [CoinDash DNS](08-security-incidents/2017-coindash-ico-dns.md) · [Coincheck NEM](08-security-incidents/2018-coincheck-nem.md) · [BitGrail](08-security-incidents/2018-bitgrail.md) · [Binance 7k BTC](08-security-incidents/2019-binance-7k-btc.md) · [Upbit](08-security-incidents/2019-upbit.md)
- 2020–2021：[LendfMe](08-security-incidents/2020-lendfme-reentrancy.md) · [bZx](08-security-incidents/2020-bzx-flashloan-series.md) · [Harvest](08-security-incidents/2020-harvest-finance.md) · [KuCoin](08-security-incidents/2020-kucoin.md) · [Poly Network](08-security-incidents/2021-polynetwork.md) · [BadgerDAO](08-security-incidents/2021-badgerdao.md) · [Cream](08-security-incidents/2021-cream-finance.md) · [PancakeBunny](08-security-incidents/2021-pancakebunny.md) · [BitMart](08-security-incidents/2021-bitmart.md) · [Vulcan Forged](08-security-incidents/2021-vulcan-forged.md)
- 2022：[Ronin](08-security-incidents/2022-ronin.md) · [Wormhole](08-security-incidents/2022-wormhole.md) · [Nomad](08-security-incidents/2022-nomad.md) · [Beanstalk](08-security-incidents/2022-beanstalk-governance.md) · [Harmony Horizon](08-security-incidents/2022-harmony-horizon.md) · [Mango](08-security-incidents/2022-mango-markets.md) · [BNB Bridge](08-security-incidents/2022-bnb-chain-bridge.md) · [FTX](08-security-incidents/2022-ftx-collapse.md) · [Wintermute](08-security-incidents/2022-wintermute.md)
- 2023：[Euler](08-security-incidents/2023-euler.md) · [Multichain](08-security-incidents/2023-multichain.md) · [Mixin](08-security-incidents/2023-mixin.md) · [Curve Vyper](08-security-incidents/2023-curve-vyper-reentrancy.md) · [Atomic Wallet](08-security-incidents/2023-atomic-wallet.md) · [Poloniex](08-security-incidents/2023-poloniex.md) · [Ledger Connect Kit](08-security-incidents/2023-ledger-connect-kit.md) · [HECO Bridge](08-security-incidents/2023-heco-bridge.md)
- 2024：[PlayDapp](08-security-incidents/2024-playdapp.md) · [DMM Bitcoin](08-security-incidents/2024-dmm-bitcoin.md) · [WazirX](08-security-incidents/2024-wazirx.md) · [Radiant](08-security-incidents/2024-radiant-capital.md) · [Orbit Chain](08-security-incidents/2024-orbit-chain.md) · [Munchables](08-security-incidents/2024-munchables.md) · [Penpie](08-security-incidents/2024-penpie-reentrancy.md) · [Gala Mint](08-security-incidents/2024-gala-games-mint.md)
- 2025–2026：[Bybit](08-security-incidents/2025-bybit.md) · [Abracadabra](08-security-incidents/2025-abracadabra.md) · [Infini](08-security-incidents/2025-infini.md) · [zkLend](08-security-incidents/2025-zklend.md) · [Cetus Sui](08-security-incidents/2025-cetus-sui.md) · [Mantra OM](08-security-incidents/2025-mantra-om-collapse.md) · [Hyperliquid JELLY](08-security-incidents/2025-hyperliquid-jelly.md) · [Phemex](08-security-incidents/2025-phemex.md) · [Four.meme](08-security-incidents/2025-four-meme.md) · [KiloEx](08-security-incidents/2025-kiloex.md) · [2026 YTD](08-security-incidents/2026-ytd-notable-incidents.md)

---

## 进度总览

查看 [`PROGRESS.md`](PROGRESS.md) 获取每个知识点的实时状态。

---

## 免责声明

本知识库内容仅供技术学习与研究，不构成投资建议。涉及协议与代币的描述以公开信息为准；任何资金操作请自行评估风险。
