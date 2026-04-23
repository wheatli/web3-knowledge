---
title: 区块链发展史（Blockchain History 1980s–2026）
module: 00-overview
priority: P0
status: DRAFT
word_count_target: 5500
last_verified: 2026-04-22
primary_sources:
  - https://bitcoin.org/bitcoin.pdf
  - https://ethereum.org/whitepaper
  - https://nakamotoinstitute.org/
  - https://en.bitcoin.it/wiki/History
  - https://ethresear.ch
---

# 区块链发展史（Blockchain History 1980s–2026）

> **TL;DR**：区块链并非凭空诞生，而是 1980s 密码学（Diffie-Hellman、Merkle、Lamport、Chaum）、1990s 密码朋克运动（Hashcash、B-money、Bit Gold）、2008 Satoshi 综合为 Bitcoin 的结果。本篇沿「密码朋克 → Bitcoin → Ethereum → ICO → DeFi Summer → NFT/L2 → 模块化 → RWA → AI+Crypto」主线梳理 2026 年前的关键节点，辅以时间轴表格与分阶段图谱，帮助读者建立宏观叙事坐标。

## 1. 背景与动机

为什么读史？因为 Web3 每一个当下的技术选择——PoS、EVM、L2、Rollup、AA、Restaking——都背着沉重的历史路径依赖。不了解 Bitcoin OP_RETURN 的争论，就理解不了 Ordinals；不知道 2017 The DAO 攻击，就说不清 Ethereum 硬分叉与 ETC 的分裂；不懂 2020 的 DeFi Summer 超额流动性挖矿，就看不透 2024 Points 运动。**历史不是装饰，是读懂协议的背景逻辑。**

## 2. 核心原理：时间轴与六大阶段划分

### 2.1 形式化：把行业史拆成阶段的原则

我们用三个维度定义"阶段"：

- **技术突破**：有无新的密码学/协议原语？
- **资本结构**：资金来源和规模是否跃迁？
- **用户画像**：使用者从极客扩散到更广人群？

据此把 2008–2026 拆为 6 个阶段：Pre-Bitcoin（1980s-2008）→ Bitcoin 纪元（2009-2014）→ Ethereum 纪元（2015-2019）→ DeFi/NFT 浪潮（2020-2021）→ 模块化 + L2（2022-2023）→ RWA + AI + AA（2024-2026）。

### 2.2 Pre-Bitcoin：密码朋克的思想准备（1980s–2008）

- **1976** Diffie-Hellman：非对称密码学。
- **1979** Ralph Merkle：Merkle Tree。
- **1982** David Chaum：Blind Signature，成立 DigiCash（1990），是第一种可用的电子货币，但中心化，2008 倒闭。
- **1992-94** Cypherpunks Mailing List（Tim May、Hal Finney、Eric Hughes）：《A Cypherpunk's Manifesto》"Privacy is necessary for an open society"。
- **1997** Adam Back: **Hashcash**（PoW 先驱，原为反垃圾邮件）。
- **1998** Wei Dai: **B-money**（分布式电子现金提案，含 PoW + 全员维护账本）。Nick Szabo: **Bit Gold**（每个 PoW 解作为 chain 的"位"）。
- **2004** Hal Finney: **RPOW**（可复用 PoW）。
- **2008-08** Satoshi 注册 bitcoin.org。**2008-10-31** 发表 Bitcoin 白皮书 `bitcoin.pdf`（9 页，8 引用）。

这些前置工作解决的子问题：
1. **数字稀缺**（Hashcash、Bit Gold）；
2. **不可伪造**（公钥签名）；
3. **可验证历史**（Merkle tree + 时间戳）；
4. **抗女巫**（PoW）；
5. **账本一致**（B-money 设想全员维护但未解决）。

Satoshi 的贡献是把这 5 个原语**整合在一个系统里并解决了拜占庭共识**——用最长链规则 + 经济激励。

### 2.3 Bitcoin 纪元（2009-2014）

- **2009-01-03** Genesis Block（block 0），coinbase 含 "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"。
- **2009-01-12** Satoshi 向 Hal Finney 发送 10 BTC：首笔交易。
- **2010-05-22** Laszlo Hanyecz 花 10,000 BTC 买两个 Pizza：**Bitcoin Pizza Day**，首个现实世界兑换。
- **2010-07-17** Mt.Gox 上线，成为最大 BTC 交易所（2014 破产，损失 850,000 BTC）。
- **2010-08-15** "Overflow Bug" CVE-2010-5139：交易凭空产生 1,844 亿 BTC，紧急硬分叉修复（史上仅此一次）。
- **2010-12-12** Satoshi 最后一次公开发帖，之后消失。
- **2011** 开始 Silk Road 接受 BTC 支付；Charlie Lee 发布 Litecoin（LTC）。
- **2012** BIP-32 HD Wallet（Pieter Wuille）；BIP-39 Mnemonic（Marek Palatinus）。
- **2013-10** Silk Road 被 FBI 关闭，Ross Ulbricht 被捕；BTC 价格首破 $1,000。
- **2014-07** Ethereum 预售（众筹 31,000 BTC，约当时 $18M），Vitalik Buterin 2013 发布 Ethereum 白皮书。
- **2014** Mt.Gox 倒闭。

### 2.4 Ethereum 纪元与 ICO 泡沫（2015-2019）

- **2015-07-30** Ethereum Frontier 主网上线，block 0 硬编码 73 个预分配地址（pre-sale 参与者）。
- **2016-06** The DAO 被攻击（$60M），催生 2016-07-20 硬分叉，分裂出 Ethereum Classic（ETC）。
- **2016-10** Ethereum Spurious Dragon 硬分叉（DoS 攻击后修复）。
- **2017** **ICO 狂潮**：全年 ICO 募资超 $6B（EOS $4.1B、Telegram TON 私募 $1.7B、Filecoin $257M、Tezos $232M）。ERC-20 标准（EIP-20，Fabian Vogelsteller）确立。
- **2017-12** BTC 首破 $19,000，CME、CBOE 推出 BTC 期货。
- **2018-01** "加密寒冬"开始，BTC 跌至 $3,200（2018-12）。
- **2018-11** Bitcoin Cash 分裂为 BCH 和 BSV。
- **2019** DAI 1.0（单抵押 SAI）→ MCD（多抵押 DAI, 2019-11）上线。Binance Launchpad 兴起 IEO 模式。Libra（Facebook）发布白皮书，受监管围剿。

### 2.5 DeFi Summer 与 NFT 浪潮（2020-2021）

**2020 DeFi Summer**：

- **2020-03-12 "黑色星期四"**：ETH 一天跌 44%，MakerDAO 清算拍卖 bug 导致 $8.3M 坏账。
- **2020-05** Compound 启动 $COMP 流动性挖矿，开启 Yield Farming 时代。
- **2020-06** Yearn Finance（$YFI 零预挖，AC 发行）。
- **2020-09** SushiSwap "Vampire Attack" Uniswap，迫使 Uniswap 2020-09-17 发 $UNI 空投（400 UNI/地址）。
- **2020-10** Harvest Finance 遭 flash loan 攻击（$24M）。
- **2020-12-01** Ethereum Beacon Chain 创世（PoS 信标链启动）。

**2021 NFT/L2 浪潮**：

- **2021-03-11** Beeple 的《Everydays》在 Christie's 拍出 $69M。
- **2021-04** Axie Infinity 月入过亿，菲律宾 GameFi 热潮。
- **2021-05** BTC 首破 $60,000；Elon Musk 令 Tesla 停止收 BTC。
- **2021-08** Poly Network 被黑 $611M，后全部归还。OpenSea 月交易量破 $3B。
- **2021-08** Ethereum London 硬分叉，EIP-1559 费用市场（base fee + burn）。
- **2021-09** **Arbitrum One** 主网上线；El Salvador 官方采用 BTC。
- **2021-10** Solana 首次宕机（之后 2022 多次）。
- **2021-11** BTC 历史高点 $69,000；OpenSea $13B/月；Facebook 改名 Meta。
- **2021-12** **Optimism** 公开主网；**dYdX**（StarkEx）TVL 破 $1B。

### 2.6 模块化、L2 爆发与 Bear Market（2022-2023）

- **2022-04-07** Axie Ronin 桥被黑 $625M（Lazarus Group）。
- **2022-05** UST/LUNA 崩盘（UST 脱锚，LUNA $80 → $0.0001），$40B+ 市值蒸发；3AC 倒闭；Celsius 暂停提款。
- **2022-08** Tornado Cash 被 OFAC 制裁；USDC 冻结 $75K 黑名单地址。
- **2022-09-15** **The Merge**：Ethereum 合并为 PoS，能耗 -99.95%。
- **2022-11** FTX 崩盘（$8B 客户资金亏空）、SBF 被捕。
- **2023-01** Bitcoin Ordinals（Casey Rodarmor）让 BTC NFT/BRC-20 成为现实。
- **2023-03** USDC 因 SVB 临时脱钩 0.87；Euler 被黑 $197M（后返还）。
- **2023-04** Ethereum Shapella 硬分叉，开启质押提款。
- **2023-06** SEC 起诉 Binance、Coinbase。
- **2023-08** **EigenLayer** 主网（Restaking 概念落地）；PayPal PYUSD 发布。
- **2023-10** Celestia 主网，"模块化 DA" 落地。
- **2023-11** HTX、Poloniex 被黑 > $200M；Sam Bankman-Fried 定罪。
- **2023-12-06** Cosmos IBC Euclid 升级。

### 2.7 RWA + AI + AA 纪元（2024-2026）

- **2024-01-10** **美国 SEC 批准 Bitcoin 现货 ETF**（BlackRock IBIT、Fidelity FBTC 等 11 只），$59B 涌入。
- **2024-03-13** **Dencun 升级**（EIP-4844 Blob），L2 费用 -90%。
- **2024-03** **BlackRock BUIDL** 发布（链上美债基金），3 个月 TVL 破 $500M。
- **2024-05** ETH Spot ETF 获批（2024-07 上线）。
- **2024-06** Ondo OUSG 迁移到 BUIDL，RWA 叙事加固。
- **2024-09** Worldcoin 改名 World；EigenLayer 启动 Slashing。
- **2024-11** Trump 当选美国总统，承诺"美国成为加密之都"；BTC 首破 $100,000（2024-12-05）。
- **2025-01** SEC SAB 121 撤销，银行可托管加密资产。
- **2025-03** Pectra 升级（EIP-7702 EOA→SCA 临时升级、EIP-7251 MaxEB 2048 ETH、EIP-7691 Blob 从 3/6 → 6/9）。
- **2025-06** Stablecoin 总市值破 $250B；GENIUS Act 稳定币法案通过。
- **2025-09** AI-Agent + Crypto 结合出现可持续产品（Virtuals Protocol、ai16z），TVL 破 $10B。
- **2026-Q1** BTC ~$120K，ETH ~$5K；RWA TVL ~$15B；L2 TVL ~$50B。

### 2.8 图示：里程碑时间线

```mermaid
timeline
    title 区块链关键节点
    1976 : Diffie-Hellman
    1997 : Hashcash (Adam Back)
    1998 : B-money (Wei Dai) / Bit Gold (Szabo)
    2008 : Bitcoin Whitepaper
    2009 : Bitcoin Genesis Block
    2010 : Pizza Day / Mt.Gox
    2013 : ETH Whitepaper
    2015 : ETH Mainnet
    2016 : The DAO Hack / ETC Fork
    2017 : ICO / ERC-20
    2020 : DeFi Summer / Beacon Chain
    2021 : NFT Boom / L2 Launch
    2022 : Merge / Luna Crash / FTX
    2023 : Shapella / Ordinals / Celestia
    2024 : BTC ETF / Dencun / BUIDL
    2025 : Pectra / GENIUS Act
    2026 : RWA + AI Integrated
```

## 3. 架构剖析：每个阶段的「栈转移」

不同阶段，行业栈的「主阵地」在不同层。

### 3.1 Bitcoin 纪元的栈：单层 Monolithic

Bitcoin 把 Consensus + Execution + DA + Settlement + 应用（只有转账）**全部压入 L1**。脚本语言刻意图灵不完备。没有 middleware、没有 wallet 抽象（所有用户跑 full node）。

### 3.2 Ethereum 纪元的栈：L1 + dApps

Ethereum 在 L1 上增加了图灵完备 EVM，直接孕育出**应用层**（ERC-20、DAO、DeFi）。但 middleware（Oracle、Indexer）尚未成熟，DeFi 早期预言机事故频发（如 bZx、Harvest 闪电贷操纵）。

### 3.3 DeFi Summer 的栈：中间件起飞

Chainlink（2019 主网，2020 起爆发）、The Graph（2020-12）、Gnosis Safe、WalletConnect 等 **中间件层**补齐。Uniswap V2/V3 奠定 AMM 标准。

### 3.4 模块化纪元的栈：DA 与 Execution 分离

Celestia、EigenDA 把 DA 单拎出来作为一层；Rollup as Service（RaaS，Caldera、Conduit、Gelato）把 Execution 层产品化。

### 3.5 客户端多样性史

| 年份 | Ethereum EL 客户端格局 |
| --- | --- |
| 2015 | geth 一家独大 |
| 2018 | Parity（~30%）崛起，后 Parity Wallet 冻结事件 |
| 2020 | Nethermind、Besu、Erigon 起势 |
| 2024 | Geth ~52%、Nethermind ~24%、Reth（Rust 新秀）增长快 |

### 3.6 典型生命周期：一个用户资产的穿越

- 2017 的用户：BTC/ETH on CEX → 提到 Metamask → 手动调 Uniswap → 等 5 分钟。
- 2026 的用户：法币入 → Coinbase Smart Wallet（ERC-4337）→ Uniswap X Intent → Solver 在 Base + Arbitrum + Solana 找最优价 → 链下 Off-chain 匹配 → 链上结算 → 30 秒完成。

## 4. 关键代码 / 实现细节：Bitcoin Genesis Block

```
// bitcoin/src/chainparams.cpp:52 (近似，各版本字段略有调整)
// Genesis block timestamp (UNIX): 1231006505 = 2009-01-03 18:15:05 UTC
// Merkle root: 4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
// Coinbase tx raw input script:
//   04ffff001d0104455468652054696d65732030332f4a616e2f3230303920436861
//   6e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f75
//   7420666f722062616e6b73
// 解码: "The Times 03/Jan/2009 Chancellor on brink of second bailout for banks"
```

这条字符串被载入链上，永久存在，成为**历史上最著名的时间戳**——既证明 genesis block 不早于 2009-01-03，也是 Satoshi 对当时金融体系的讽刺。

## 5. 演进与版本对比：Ethereum 硬分叉史

| 硬分叉 | 时间 | 关键 EIP | 影响 |
| --- | --- | --- | --- |
| Frontier | 2015-07 | — | 主网启动 |
| Homestead | 2016-03 | EIP-2/7 | 稳定性 |
| DAO Fork | 2016-07 | 硬编码 | 退款 The DAO |
| Byzantium | 2017-10 | EIP-649/658 | 难度炸弹延缓、区块奖励 5→3 ETH |
| Constantinople | 2019-02 | EIP-1234 | 3→2 ETH |
| Istanbul | 2019-12 | EIP-2028 | calldata gas 下调 |
| Berlin | 2021-04 | EIP-2929 | Gas 模型改 |
| London | 2021-08 | EIP-1559 | 费用市场 + burn |
| Altair | 2021-10 | — | Beacon chain 第一次升级 |
| Paris（The Merge） | 2022-09 | — | PoS |
| Shanghai/Capella | 2023-04 | EIP-4895 | 质押提款 |
| Dencun | 2024-03 | EIP-4844 | Blob |
| Pectra | 2025 | EIP-7702/7251/7691 | AA + MaxEB + 2x Blob |

## 6. 实战示例：访问创世块

```bash
# via Bitcoin Core RPC
bitcoin-cli getblockhash 0
# 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f

bitcoin-cli getblock 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f 2

# 解析 coinbase 里的 Times 信息：
bitcoin-cli getblock 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f 2 \
  | jq -r '.tx[0].vin[0].coinbase' \
  | xxd -r -p
```

## 7. 安全与已知攻击：历史上影响最大的十起事件

1. **Mt.Gox**（2014）：850,000 BTC 丢失。
2. **The DAO**（2016）：$60M，导致 ETC 硬分叉。
3. **Parity Wallet Freeze**（2017）：$150M ETH 被冻结。
4. **Coincheck**（2018）：NEM 被黑 $530M。
5. **Ronin Bridge**（2022）：$625M。
6. **LUNA/UST**（2022）：$40B+ 市值蒸发。
7. **FTX**（2022）：$8B 客户资金亏空。
8. **Wormhole**（2022）：$320M。
9. **Multichain**（2023）：$126M 跑路。
10. **DMM Bitcoin**（2024-05）：$305M BTC 被盗。

## 8. 与同类方案对比：比特币 vs 以太坊 vs Solana 起源叙事

| 维度 | Bitcoin | Ethereum | Solana |
| --- | --- | --- | --- |
| 创始人 | Satoshi Nakamoto（匿名） | Vitalik Buterin（公开） | Anatoly Yakovenko |
| 核心理念 | 数字黄金、抗审查货币 | 世界计算机 | 高吞吐单链 |
| 融资 | 无 ICO | 预售 31,000 BTC | VC 轮 |
| 治理 | BIP + 社区 | EIP + All Core Dev | 核心团队 + 基金会 |
| 典型硬分叉 | BCH/BSV | ETC、各 EIP 硬分叉 | 网络重启（历史 ~6 次宕机重启） |

## 9. 延伸阅读

- 《Digital Gold》Nathaniel Popper：Bitcoin 起源。
- 《The Infinite Machine》Camila Russo：Ethereum 起源。
- 《Mastering Bitcoin》Andreas Antonopoulos。
- 《Mastering Ethereum》Antonopoulos + Wood。
- Nakamoto Institute 归档：nakamotoinstitute.org。
- 中文：《精通以太坊》（汪晓明译）；巴比特早期报道。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 密码朋克 | Cypherpunk | 1990s 倡导用密码学保护隐私的技术社群 |
| 创世块 | Genesis Block | 区块链第 0 号区块 |
| 硬分叉 | Hard Fork | 不向下兼容的协议升级 |
| ICO | Initial Coin Offering | 代币首次发行 |
| DeFi Summer | — | 2020 年 DeFi 爆发期 |
| 合并 | The Merge | Ethereum PoW → PoS 升级 |
| Dencun | Cancun+Deneb | Ethereum 2024 升级，引入 Blob |
| Pectra | Prague+Electra | Ethereum 2025 升级，引入 AA |

---

*Last verified: 2026-04-22*
