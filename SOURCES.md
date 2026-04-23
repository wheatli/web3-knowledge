# 引用源白名单与采信分级

本知识库严格按下述 **Tier 分层** 采信。任何技术断言需满足：至少 1 条 Tier 1 + 1 条 Tier 2/3 交叉引用；数字/价格类断言必须标注查询日期与数据源。

---

## Tier 1 — 官方/一手源（最高权威）

### 规范与白皮书

- **Bitcoin**：`bitcoin.org/bitcoin.pdf`、`en.bitcoin.it/wiki/Protocol_documentation`、BIPs `github.com/bitcoin/bips`
- **Ethereum**：`ethereum.org/en/whitepaper/`、Yellow Paper `ethereum.github.io/yellowpaper/paper.pdf`、EIPs `eips.ethereum.org`、Consensus Specs `github.com/ethereum/consensus-specs`
- **Solana**：`solana.com/solana-whitepaper.pdf`、`solana.com/docs`、SIMDs `github.com/solana-foundation/solana-improvement-documents`
- **Move 系**：Diem/Aptos/Sui 官方 docs 与 Move Book
- **Cosmos**：`docs.cosmos.network`、IBC Spec
- **跨链协议**：LayerZero/Wormhole/Axelar 官方 Whitepaper
- **Uniswap / Aave / Compound / Maker**：各自官方 docs + whitepaper

### GitHub 官方实现（必须标 commit hash 或 tag）

- `bitcoin/bitcoin`、`ethereum/go-ethereum`、`prysmaticlabs/prysm`、`sigp/lighthouse`
- `anza-xyz/agave`、`firedancer-io/firedancer`、`anza-xyz/anchor`
- `MystenLabs/sui`、`aptos-labs/aptos-core`
- `OffchainLabs/nitro`（Arbitrum）、`ethereum-optimism/optimism`
- `Uniswap/v2-core`、`Uniswap/v3-core`、`Uniswap/v4-core`、`aave/aave-v3-core`

---

## Tier 2 — 权威机构研究

- **a16z crypto** — `a16zcrypto.com`
- **Paradigm Research** — `paradigm.xyz/writing`
- **Messari** — `messari.io/research`
- **Galaxy Digital Research**
- **Delphi Digital** — `members.delphidigital.io`
- **Electric Capital Developer Report**
- **L2BEAT** — `l2beat.com`（L2 数据与 Stage 评级）
- **DefiLlama** — `defillama.com`（TVL、协议数据）
- **Dune Analytics** — `dune.com`（链上数据）
- **Chainalysis / Elliptic** — 反洗钱/合规报告

---

## Tier 3 — 知名个人 / 团队博客

- **Vitalik Buterin** — `vitalik.eth.limo`
- **Dankrad Feist** — `dankradfeist.de`
- **Hasu** — `uncommoncore.co`
- **0xfoobar** — `0xfoobar.substack.com`
- **Jameson Lopp** — `lopp.net`
- **Flashbots** blog — `writings.flashbots.net`
- **Helius** blog — `helius.dev/blog`
- **SlowMist / 慢雾科技** — `slowmist.com`
- **登链社区** — `learnblockchain.cn`（中文，含专栏、教程、图谱）
- **OpenZeppelin** blog — `blog.openzeppelin.com`
- **RareSkills** blog — `rareskills.io/blog`

---

## Tier 4 — 社区教学

- CryptoZombies — `cryptozombies.io`
- Solidity by Example — `solidity-by-example.org`
- Uniswap V3 Book (Jeiwan) — `uniswapv3book.com`
- LearnWeb3 — `learnweb3.io`
- WTF Academy — `wtf.academy`

---

## Tier 5（仅辅助引用，不作为唯一出处）

- Medium / Mirror / Substack 个人文章
- Twitter/X thread（存档到 wayback 或截图）
- 维基百科

---

## 引用格式规范

**文内引用**：
- 代码引用：`` `仓库名/路径/文件.ext:行号` ``，并在 frontmatter 中登记 commit 或 tag。
- URL 引用：`[显示文字](完整 URL)`
- 论文/EIP：`[EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)`

**"延伸阅读"节**：按 Tier 顺序列出，每条给出一句话说明。

**数据时效**：凡提到具体数值（TVL、TPS、Staking 率、Validator 数），必须写明 "截至 YYYY-MM-DD，来源 X"。

---

## 安全事件档案专用源（08-security-incidents/）

以下源专用于 `08-security-incidents/` 事件档案。事件类断言要求：至少 1 条 Tier 1（受害项目官方 post-mortem / FBI / OFAC / 慢雾或 PeckShield 链上证据）+ 1 条独立交叉源。

### 实时监控与告警
- **SlowMist Hacked 数据库** — <https://hacked.slowmist.io>（中文，慢雾自研 AML 归因，含历史累计）
- **CertiK Skynet** — <https://skynet.certik.com>（实时监控 + 项目风险评分）
- **PeckShield Alert**（X/Twitter 账号）— <https://twitter.com/PeckShieldAlert>（事发后 <15 min 首发链上异动，tx hash 可靠）
- **BlockSec Incidents** — <https://blocksec.com/incidents>（MetaSleuth 资金流追踪，含多跳 + 混币还原）
- **Cyvers Alerts** — <https://cyvers.ai>（实时 ML 风险信号，对 EOA 行为聚类）

### 事件深度复盘
- **rekt.news leaderboard** — <https://rekt.news/leaderboard/>（英文深度复盘 + 金额排行榜，写作风格偏杂志）
- **DeFiHackLabs** — <https://github.com/SunWeb3Sec/DeFiHackLabs>（Foundry 可复现 POC，覆盖 150+ 历史事件）

### 行业年度报告
- **Chainalysis Crypto Crime Report**（年度）— <https://www.chainalysis.com/crypto-crime-report/>（口径权威，Lazarus / 制裁归因的 de-facto 标准）
- **Elliptic Typologies Report** — <https://www.elliptic.co>
- **TRM Labs Annual Crypto Crime Report**
- **Immunefi Crypto Losses Report**（季度）

### 归因与执法
- **FBI Internet Crime Complaint Center (IC3)** 公告 — `ic3.gov`
- **OFAC SDN List**（朝鲜 / 伊朗 / 混币器地址）— `home.treasury.gov/policy-issues/financial-sanctions/specially-designated-nationals-and-blocked-persons-list-sdn-human-readable-lists`
- **ZachXBT investigation** — <https://twitter.com/zachxbt>（独立调查员，Tier 3，但长期记录准确）
- **Mandiant / Google TAG**（APT 归因报告）

### 交叉验证
- 受害项目官方 **post-mortem** 博客（优先级最高，视为 Tier 1）
- Etherscan / Solscan / Arkham 链上证据（原始证据，Tier 1）
- 交易所官方声明 + 社区 AMA 记录
