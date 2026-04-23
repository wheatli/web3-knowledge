---
title: 代币化国债（Ondo OUSG/USDY、BlackRock BUIDL、Franklin FOBXX、Superstate）
module: 04-dapp/rwa
priority: P0
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://ondo.finance/
  - https://securitize.io/primary/blackrock
  - https://www.franklintempleton.com/investments/options/money-market-funds/products/29386/SINGLCLASS/franklin-on-chain-u-s-government-money-fund/FOBXX
  - https://superstate.com/
  - https://rwa.xyz/treasuries
  - https://github.com/OndoFinance
---

# 代币化国债（Ondo OUSG/USDY、BlackRock BUIDL、Franklin FOBXX、Superstate）

> **TL;DR**：代币化美国国债 / 货币市场基金（MMF）是 2023–2026 RWA 最成熟赛道。2026 Q1 总规模约 70–80 亿美元，BlackRock BUIDL（Securitize 发行）超过 30 亿美元居首，Ondo OUSG（12 亿）/USDY（8 亿）、Franklin FOBXX（7 亿）、Superstate USTB/USCC（5 亿）、Hashnote USYC/WisdomTree WTGXX 各约 1–5 亿。产品底层均为短久期 T-Bills + 逆回购 + 现金；差异主要在投资者门槛（零售/合格/机构）、合规包装（Reg D、Reg S、1940 Act MMF、3c-7）、链分布与可组合性。DAO 国库（Maker、Frax、Sky、Arbitrum、Lido）是最大买方，DeFi 协议通过 Morpho/Pendle/Aave 进一步把 RWA 折叠为抵押/收益。

## 1. 背景与动机

2022 年 6 月美联储启动加息周期，2023 T-Bills 年化收益率达 5.3%，吸引大量稳定币闲置资金。但传统 MMF（Vanguard VMFXX、Fidelity SPAXX）对加密原生用户/协议并不友好：需要券商账户、T+1 结算、无法作为链上抵押。代币化国债解决：(1) **24/7 铸赎**：T+0 USDC 换代币；(2) **链上可组合**：Morpho 借贷、Pendle 分拆固定收益、Curve 流动性池；(3) **透明 NAV**：每日/每分钟更新；(4) **DAO 友好**：国库可直接参与而无需 KYB。2024-03 BlackRock 联合 Securitize 推出 BUIDL（全称 BlackRock USD Institutional Digital Liquidity Fund），Ondo 被作为 anchor investor（5 亿美元）引入；BUIDL 成为 RWA 里程碑。2025 Circle 收购 Hashnote，把 USYC 直接纳入 USDC 生态；Ondo 启动 Ondo Chain（L1），力求成为 RWA 中心枢纽。

## 2. 核心原理

### 2.1 形式化定义：NAV Token 与累积收益

代币化国债主要有两种形态：
1. **NAV Token（价格浮动，rebasing-free）**：单价每日按 NAV 更新，持有人份额数量不变。示例：BUIDL、OUSG、USTB。
$$P_{\text{token}}(t) = \frac{\text{NAV}_t}{\text{Shares}_t}, \quad P(0) = \$1$$
每日收益通过 NAV 上涨体现。

2. **Rebasing Token（数量累积）**：持有人份额按日增长，价格稳定在 $1。示例：USDY、USDtb、Backed blB01。
$$\text{Balance}_{\text{user}}(t+1) = \text{Balance}_{\text{user}}(t) \cdot (1 + r_{\text{daily}})$$

选择取决于会计偏好与 DeFi 集成（如 ERC-4626 更适合 NAV 模式；rebasing 适合日常使用）。

### 2.2 关键数据结构：产品一览

| 产品 | 发行人 | 法律包装 | 链 | 最低 | 投资者 |
| --- | --- | --- | --- | --- | --- |
| BUIDL | BlackRock Financial Mgmt (via Securitize) | 3c-7 fund | Ethereum/Polygon/Avalanche/Aptos/Optimism/Arbitrum | $5M | Qualified Purchaser |
| OUSG | Ondo Finance | BDC + RegD 506(c) | Ethereum/Polygon/Solana | $5K | Accredited |
| USDY | Ondo Finance | RegS / 非美投资者 | Ethereum/Solana/Mantle/Sui/Noble | 无（KYC 后） | 非美 |
| FOBXX / BENJI | Franklin Templeton | 1940 Act MMF | Stellar/Polygon/Arbitrum/Avalanche/Base | 零售可 | US retail |
| USTB | Superstate | BDC | Ethereum | $10K | Accredited |
| USCC | Superstate | BDC (basis trade) | Ethereum | $10K | Accredited |
| USYC | Hashnote (Cumberland) | Cayman Fund | Ethereum/Canton | $100K | Qualified |
| WTGXX | WisdomTree | 1940 MMF | Stellar | 零售 | US retail |

### 2.3 子机制拆解

1. **BUIDL 机制**：BlackRock Financial Management 管理底层 T-Bills + repo + cash；BNY Mellon 托管；Securitize 作为 Transfer Agent 铸销与记名账簿。月度分红以 BUIDL token 形式分发（不调整 price）。可与 Circle 1:1 `bidirectional` 换 USDC（2024-11 启动，实时兑换）。
2. **OUSG 机制**：Ondo 买入 BlackRock iShares $BIB 短期 T-Bill ETF 与 BUIDL，NAV token 模式；每日开放 mint/redeem。2025 起接受 stablecoin + 支持 Flux Finance 作为 DeFi 抵押。
3. **USDY**：Ondo 的零售非美产品，Rebasing，1:1 USD 面值。底层：现金 + 短 T-Bills。支持 Solana、Mantle、Sui、Noble（Cosmos）。
4. **FOBXX**：Franklin Templeton 的 1940 Act 政府货币市场基金，Stellar 为主链，BENJI 代币为份额记录；允许 T+0 申购赎回；美国零售投资者可购。
5. **Superstate USTB / USCC**：USTB 对标短 T-Bills，USCC 是 basis trade（现货 BTC + 空 BTC 期货）加短 T-Bills，类似 Ethena 思路但机构合规包装。
6. **Hashnote USYC**：2024 支持 DeFi 组合（通过 Canton 许可链连接 Ethereum），2025 被 Circle 收购后用于 USDC 底层之一。
7. **治理白名单与 Transfer Hook**：大部分采用 ERC-20 + onlyWhitelisted 修饰符；BUIDL 使用 Securitize DS Protocol；USDY 用 Ondo 自研 `AllowList` 合约。

### 2.4 参数与常量（2026 Q1 典型）

| 参数 | 取值 |
| --- | --- |
| BUIDL NAV 更新 | 每 24h |
| OUSG 赎回 SLA | T+0（工作时段） |
| USDY APY | ~4.25% |
| FOBXX expense ratio | 0.20% |
| 托管行 | BNY Mellon / State Street |
| 管理费 | 0.15–0.50% |

### 2.5 边界条件与失败模式

- **美债降息**：美联储降息直接压低 RWA 国债收益，抗通胀代币（黄金、TIPS）可能替代。
- **NAV break-the-buck**：货币市场基金极端情况（2008 Reserve Primary Fund）跌破 $1，代币价格同步下破。
- **Securitize/Transfer Agent 单点**：BUIDL 依赖 Securitize DS Protocol，其系统故障将阻塞赎回。
- **白名单碎片化**：同一用户在 BUIDL 与 OUSG 重复 KYC；ERC-3643 + ONCHAINID 尝试统一。
- **监管回转**：SEC 对 DeFi 与 RWA 结合点（例如 Morpho USDY 市场）可能认定未注册经纪商。
- **跨链桥风险**：BUIDL 通过 Wormhole NTT 跨链（2024 启动），需信任 DVN。
- **Ondo Chain 与 Canton 封闭**：特殊链上 RWA 与通用 DeFi 组合性差。

### 2.6 图示

```mermaid
flowchart LR
    Investor -->|USDC + KYC| Platform[Securitize / Ondo Portal]
    Platform -->|subscribe| SPV[BDC / 3c-7 Fund]
    SPV --> Custodian[BNY Mellon]
    Custodian --> TBills[Short T-Bills / Repo]
    Platform -->|mint whitelisted| Token[BUIDL / OUSG / USDY / BENJI]
    Token --> Investor
    Admin[NAV Oracle] --> Token
    Token -->|redeem| Platform
    Platform -->|USDC 1:1 (BUIDL + Circle)| Investor
```

```
BUIDL 生态
 DAO Treasury -> BUIDL (Ethereum)
     │             │
     │             ▼
     │        Securitize Portal
     │             │
     ▼             ▼
Pendle/Morpho    BlackRock Fund
 (collateral)     (T-Bills)
```

## 3. 架构剖析

### 3.1 分层视图

1. **Asset Manager**：BlackRock / Franklin / Ondo / Superstate。
2. **Administrator & Transfer Agent**：Securitize, Apex, State Street Digital。
3. **Custody**：BNY Mellon, State Street, Anchorage Digital。
4. **Blockchain Settlement**：Ethereum L1 为主，Polygon/Avalanche/Solana 等 secondary。
5. **Distribution**：机构合作 CEX（Coinbase Custody、Copper、Fireblocks）+ 自营门户。
6. **DeFi Integration**：Flux Finance (Ondo)、Morpho (USDY/BUIDL)、Pendle (sUSDY)、Spark (Maker)。

### 3.2 核心模块清单

| 模块 | 职责 | 依赖 | 可替换性 |
| --- | --- | --- | --- |
| Token Contract (whitelist) | 合规转账 | AllowList | 低 |
| Transfer Agent | 账簿 | 发行方 | 中 |
| NAV Oracle | 每日 NAV | Admin | 中 |
| Redemption Queue | 赎回排队 | 发行方 | 中 |
| Cross-chain Bridge (NTT/LZ OFT) | 多链部署 | Wormhole/LZ | 中 |
| DeFi Wrappers | sToken/Pendle PT-YT | 合作协议 | 高 |
| Attestation | PoR | Chainlink | 高 |

### 3.3 数据流：DAO 国库购入 BUIDL + 组合到 Pendle

1. Maker Treasury（已 KYB）通过 Securitize Portal 提交 10M USDC 订阅。
2. Securitize 记录订阅，BlackRock 把 USDC 换成 T-Bills 入库。
3. Securitize mint 10M BUIDL 到 Maker 地址（whitelisted）。
4. 每日 NAV 上涨（或 BUIDL token 分红）累积收益。
5. Maker 把 BUIDL 转到 Pendle（需 Pendle 市场加入白名单），分拆为 PT（本金）+ YT（收益）。
6. PT 回售到 Morpho 作为抵押借 USDS，形成杠杆收益策略。
7. Redeem：调用 `redeem(amount)` → Securitize 释放 USDC（T+0 via Circle bidirectional）。

### 3.4 客户端 / 参考实现

- **BUIDL 合约**：0x7712c34205737192402172409a8F7ccef8aA2AEc (Ethereum)，基于 Securitize DS Protocol。
- **OUSG / USDY**：Ondo Finance GitHub (https://github.com/OndoFinance)，采用 `rOUSG` rebasing wrapper + `OUSGRedemption` 合约。
- **FOBXX/BENJI**：Stellar 原生资产 + EVM mirrored（Polygon 上 ERC-20）。
- **Superstate USTB**：独立合约，OpenZeppelin 基础。

### 3.5 扩展接口

- **Wormhole NTT**：BUIDL 跨链原生转移。
- **Chainlink PoR**：Backed.fi 使用。
- **Pendle PT/YT**：将收益与本金分拆。
- **Morpho Market**：per-asset isolated pool。

## 4. 关键代码 / 实现细节

Ondo `OUSG.sol` 核心（https://github.com/OndoFinance/ousg 简化片段，约 80–150 行）：

```solidity
contract OUSG is ERC20, AccessControl {
    IKYCRegistry public kycRegistry;
    uint256 public constant KYC_REQUIREMENT_GROUP = 1;

    function _beforeTokenTransfer(address from, address to, uint256) internal view override {
        if (from == address(0) || to == address(0)) return; // mint/burn 忽略
        require(kycRegistry.getKYCStatus(KYC_REQUIREMENT_GROUP, from), "sender not KYCed");
        require(kycRegistry.getKYCStatus(KYC_REQUIREMENT_GROUP, to), "recipient not KYCed");
    }

    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }
}
```

`OUSGInstantManager.sol` 申赎：

```solidity
function subscribeRebasingOUSG(uint256 usdcAmount) external returns (uint256 ousgMinted) {
    usdc.transferFrom(msg.sender, address(this), usdcAmount);
    uint256 nav = oracle.getLatestPrice();
    ousgMinted = (usdcAmount * 1e12 * 1e18) / nav;
    ousg.mint(msg.sender, ousgMinted);
}
```

Securitize DS Protocol 身份模块（简化）：

```solidity
modifier dsRole(bytes32 role) {
    require(IDSToken(tokenAddr).getDSService("ROLE_MANAGER").hasRole(msg.sender, role));
    _;
}
```

## 5. 演进与版本对比

| 时间 | 事件 |
| --- | --- |
| 2021-09 | Sygnum 首发代币化公司债 |
| 2023-01 | Ondo OUSG 首发 |
| 2023-04 | Franklin FOBXX Stellar 扩展 |
| 2024-03 | BlackRock BUIDL |
| 2024-07 | Ondo USDY Solana/Mantle |
| 2024-11 | BUIDL ↔ USDC 双向兑换 |
| 2025-02 | Circle 收购 Hashnote USYC |
| 2025-Q3 | Ondo Chain 上线 |

## 6. 实战示例

DAO 直接调用 Ondo 合约（需已 KYB）：

```solidity
IERC20(USDC).approve(ousgManager, 100_000e6);
uint256 shares = IOUSGInstantManager(ousgManager).subscribeRebasingOUSG(100_000e6);
```

查询 BUIDL NAV：

```bash
cast call $BUIDL "latestRoundData()(uint80,int256,uint256,uint256,uint80)" \
  --rpc-url https://eth.llamarpc.com
```

## 7. 安全与已知攻击

- **USDR Tangible 崩盘（2023-10）**：房产 RWA 但属于"国债风格"流动性池设计失败，跌至 $0.5。
- **Securitize Transfer Agent 延迟**：BUIDL 周末赎回需等到周一处理。
- **Ondo Bridge 事件**：2024 早期 Ondo USDY 跨链时 Noble 通道短暂暂停。
- **Franklin Stellar 合规**：BENJI 在 Polygon mirror 引入时曾有会计对账延迟。
- **DeFi 合规冲突**：Morpho USDY 市场曾被 SEC 调研（2025 Q1），最终以合规"仅 KYC 用户可参与"结束。

## 8. 与同类方案对比

| 维度 | BUIDL | OUSG | USDY | FOBXX | USTB |
| --- | --- | --- | --- | --- | --- |
| 发行 | BlackRock | Ondo | Ondo | Franklin | Superstate |
| 投资者 | QP | Accredited | 非美零售/机构 | US retail | Accredited |
| 最小 | $5M | $5K | 无 | $20 | $10K |
| 链 | 7 链 | 3 链 | 5 链 | 4 链 | Ethereum |
| 申赎 | T+0 | T+0 | T+0 | T+0 | T+1 |
| 资产组成 | T-Bills + Repo | BIB ETF + BUIDL | 现金 + T-Bills | Gov MMF | T-Bills |

## 9. 延伸阅读

- BlackRock × Securitize BUIDL 白皮书
- Ondo USDY Whitepaper
- Franklin Templeton Benji Investments 文档
- Superstate USCC basis strategy 报告
- rwa.xyz Treasuries Dashboard
- JPM "Project Guardian" Tokenized Assets pilot

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 3c-7 Fund | — | 1940 Act 3(c)(7) 豁免基金 |
| BDC | Business Development Company | 美 RegD 私募载体 |
| Transfer Agent | — | 证券账簿管理方 |
| NTT | Native Token Transfer | Wormhole 跨链协议 |
| QP | Qualified Purchaser | 美合格买家（≥ $5M 投资资产） |
| Rebasing | — | 按比例调整余额的代币 |
| Basis Trade | — | 现货多 + 期货空 的套保交易 |

---

*Last verified: 2026-04-22*
