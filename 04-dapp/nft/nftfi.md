---
title: NFTfi：NFT 借贷（BendDAO / Blur Blend / Arcade）与碎片化
module: 04-dapp/nft
priority: P2
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://docs.benddao.xyz/
  - https://docs.blur.io/blend/overview
  - https://docs.arcade.xyz/
  - https://docs.nftfi.com/
  - https://paradigm.xyz/2023/05/blend
---

# NFTfi：NFT 借贷（BendDAO / Blur Blend / Arcade）与碎片化

> **TL;DR**：NFT 流动性差、估值困难、最小交易单位为 1，这些特征导致早期 NFT 没有 DeFi 乐高。NFTfi（P2P 定制借贷，2020）首开先河；BendDAO（2022）把 ERC-20 池化模式嫁接到 NFT 上做 P2Pool 闪电贷；Blur Blend（2023，Paradigm 提出）借助 EIP-712 签名 + 荷兰拍强平，把 NFT 借贷做成「永续式 + 链下匹配」；Arcade 与 MetaStreet 聚焦大宗抵押与机构。碎片化（fractionalization）则通过 ERC-20 包装 NFT（NFTX / Fractional / unic.ly）和 NFT AMM（Sudoswap、NFTX v3）解决高价 NFT 的进入成本。本篇对比四大借贷协议的利率机制、清算曲线、预言机设计与已知危机（BendDAO 2022 年 ETH bank run）。

## 1. 背景与动机

以 BAYC 为例，2022 年初地板价约 100 ETH (~$300K)，普通用户无力购入，也没有方式「用这张图借钱周转」。传统 DeFi 解决方案（Aave / Compound）只接受同质化资产作抵押，因为它们假设抵押品有 Chainlink 预言机、滑点线性、清算可以原子 swap。NFT 的 price discovery 本身就不稳定（floor 与稀有之间差 100 倍）、流动性窗口可能是周级的。

NFTfi（Stephen Young，2020）用点对点撮合解决：借款人指定 NFT 抵押，出借人报价（本金、期限、利率），双方签名链下订单，链上托管 NFT，到期还款或 default 后 NFT 转给出借人。这种模式风险可控（每笔合约单独定价）但流动性低。BendDAO（2022-03）想用 Compound 式池化模型加速：用户把 ETH 存入 Pool，借款人抵押 BAYC 等蓝筹 NFT，按池化利率借出，按 `floorPrice * LTV` 控制风险，达到阈值时荷兰拍 + 48 小时宽限期清算。Blur Blend（2023-05）把这个模式再推一步：借贷匹配仍是 P2P 签名，但引入「永续」——贷款默认无期，任一方可触发 refinance 拍卖；出借人随时可退出。Arcade（v1 2022, v3 2024）服务机构化大宗抵押，支持 ERC-721 / ERC-1155 篮子抵押与多资产组合。

碎片化（fractionalization）源于 2021 年 Fractional.art（后改名 Tessera）。一个 NFT 被锁进 vault，发行 ERC-20 代表所有权；当持有足够比例时可发起 buyout 拍卖。该模式合规风险（SEC 关注「投资合同」）与退市问题使其热度下降；后继者 NFTX（2020 起）用 vault 换取 vToken，vToken 在 Uniswap / Sushiswap 上自由交易。Sudoswap（2022）用 bonding curve 做 NFT AMM，同样提供流动性入口。

## 2. 核心原理

### 2.1 形式化定义

NFT 借贷合约可定义为函数 $L$：给定抵押物 $n$（NFTId）、本金 $P$、期限 $T$、利率 $r$、LTV $\alpha$，输出贷款流水：

$$
\mathrm{Loan} := (\text{borrower}, \text{lender}, n, P, r, T, \text{maturity}, \text{state})
$$

其中 $P \le \alpha \cdot V(n)$，$V(n)$ 是估值（floor / oracle / appraisal），$\alpha \in (0, 1)$。到期 $t > T$ 时，若未还款则 $\text{state} \leftarrow \text{default}$，NFT 转给 lender 或进入拍卖。

### 2.2 P2P vs P2Pool 模型

- **P2P（NFTfi / Arcade / X2Y2 Lending）**：出借人对单笔贷款报价，借款人接单。估值完全个性化，无预言机依赖。缺点：匹配慢。
- **P2Pool（BendDAO）**：出借人放 ETH 到 Pool，借款人抵押并借出。利率由利用率公式（类似 Compound 的 JumpRate Model）决定。清算基于 floor oracle。优点：即时借贷；缺点：预言机与池深绑定，存在 bank run 风险。
- **混合式（Blur Blend）**：P2P 签名匹配 + 永续无期 + Refinance 拍卖。Lender 随时退出（触发拍卖让新 lender 接盘），若无人接则进入违约拍卖。

### 2.3 关键数据结构

BendDAO 的 `LendPool` 合约（`contracts/protocol/LendPool.sol`）关键字段：
- `reserves[asset]`：利率模型、cToken 地址、借款指数。
- `nfts[collection]`：LTV、liquidation threshold、liquidation bonus、auction duration。
- `loans[loanId]`：borrower、nft、scaledAmount、status。

Blend 订单的 EIP-712 类型：
```
LoanOffer(
  address lender,
  Collection collection,
  uint256 loanAmount,
  uint256 maxLoanAmount,
  uint256 minLoanAmount,
  uint256 auctionDuration,
  uint256 expirationTime,
  uint256 rate,
  address oracle,
  uint256 nonce
)
```
borrower 签名 `BorrowOffer` 包含 tokenId 与期望条件，二者撮合生成 loan。

### 2.4 子机制拆解

1. **估值 / 预言机**：BendDAO 用 OpenSea + LooksRare + X2Y2 floor TWAP；Blend 采用「无预言机 + 拍卖发现」（Paradigm blog 重点）；Arcade 依赖链下 appraisal。
2. **利率模型**：BendDAO 借鉴 Compound JumpRate：利用率 < Uoptimal 时线性，超过后陡峭。Blend 的利率由 lender 自行报价，市场竞价。
3. **清算机制**：BendDAO 双阶段——触发拍卖（48h 宽限期，借款人可还款赎回）→ 拍卖成交；Blend 采用荷兰拍 refinance：lender 不想持有时发起，利率从基础上涨直到新 lender 接手；到期无人接则 NFT 归原 lender。
4. **违约处置**：NFT 直接转给 lender（BendDAO / NFTfi / Arcade）。
5. **闪电贷 / Instant Buy**：BendDAO 的「Down Payment」让买家只付 40% 首付，合约先借 60% 支付 OpenSea，再把 NFT 抵押进池。

### 2.5 关键参数（BendDAO 默认，截至 2026-04）

| 参数 | 值 | 可治理 |
| --- | --- | --- |
| 支持 collection | 10–20 蓝筹（BAYC/MAYC/Doodles 等）| 治理提案 |
| 基础 LTV | 40% | 是 |
| Liquidation Threshold | 90% | 是 |
| Liquidation Bonus | 5% | 是 |
| Auction Duration | 48h | 是 |
| 预言机源 | OpenSea + LooksRare + X2Y2 floor | 是 |
| Reserve Factor | 30% | 是 |

### 2.6 边界条件与失败模式

- **Floor Price 操纵**：少量 wash trade 可拉动 floor；BendDAO 要求多市场 + TWAP 缓解。
- **Bank Run**：2022-08 BendDAO 因 ETH 利用率达 100% 引发恐慌，lender 无法取 ETH；最终通过调整 LTV、利率曲线、扩大 Reserve Factor 熬过。
- **Oracle 迟滞**：NFT floor 变动极快，TWAP 在暴跌时显得滞后，容易让 lender 吃坏账。
- **用户 grief attack**：Blend 中 borrower 可以频繁 refinance 压榨 lender（已引入 cool-down）。
- **碎片化退市**：若 buyout 阈值难触发，vault 里的 NFT 可能永久锁死。

## 3. 架构剖析

### 3.1 分层视图

1. **资产层**：蓝筹 PFP NFT (BAYC/CryptoPunks/Azuki) + ERC-20 稳定币 / ETH。
2. **估值层**：Chainlink NFT Floor Feed（有限）、Reservoir Oracle、MetaStreet Oracle、链下 appraisal。
3. **协议层**：BendDAO LendPool / Blend Exchange / Arcade / NFTfi。
4. **撮合层**：P2P 签名 API、Pool 池化合约、Refinance 拍卖。
5. **应用层**：钱包 UI、聚合器（Gondi、DropsDAO 聚合借贷 TVL）。

### 3.2 核心模块清单

| 模块 | 协议 | 职责 | 依赖 |
| --- | --- | --- | --- |
| LendPool.sol | BendDAO | 池化借贷 | NFTOracle / bNFT |
| NFTOracle.sol | BendDAO | 聚合 floor | 多市场 API |
| Blend.sol | Blur | 签名撮合 + 永续贷 | Seaport-like |
| LoanCore.sol | Arcade | 多资产抵押记录 | ERC-721 |
| PromissoryNote | Arcade | 贷款凭证 NFT | ERC-721 |
| NFTfi DirectLoan | NFTfi | P2P 合同记账 | — |
| NFTX Vault | NFTX | NFT → vToken | ERC-20 |
| SudoAMM Pair | Sudoswap | NFT-ETH AMM | LSSVMPair |

### 3.3 数据流 / 生命周期

以 BendDAO 借 30 ETH 抵押 BAYC #1234 为例：

1. 用户 approve BAYC 合约给 `LendPool`。
2. 调用 `LendPool.borrow(asset=WETH, amount=30e18, nftAsset=BAYC, nftTokenId=1234, onBehalfOf=me)`。
3. 合约铸造 bBAYC（bound NFT）给用户；把 BAYC 从用户转入 BendDAO。
4. 从 Reserve 划出 30 WETH 给用户，更新 borrow index。
5. 随时间累计利息：`debt = scaledAmount * index`。
6. 若 `floor * LTV < debt`，合约 `auction` 开始 48h 宽限期；借款人可 `redeem` 还款（多收 5% 惩罚）。
7. 宽限期后，出价最高者赢得 NFT；清算奖励 5% 归拍卖者。

### 3.4 客户端多样性 / 参考实现

- **BendDAO**：Solidity，地址 `0x70b97A0da65C15dfb0FFA02aEE6FA36e507C2762`（LendPool），开源 GitHub。
- **Blend**：Blur 闭源后端 + 开源 Exchange 合约；可直接通过 Reservoir SDK 聚合调用。
- **Arcade**：v3 全合约开源（GitHub `arcadexyz/v2-contracts`）。
- **NFTfi**：DirectLoan v2 地址 `0xE52Cec0E90115AbeB3304BaA36bc2655731f7934`，半开源。

### 3.5 扩展 / 互操作接口

- **Blur Bid Pool + Blend 组合**：卖家挂 bid，买家用 Blend 贷款接单。
- **Reservoir Loans API**：统一 BendDAO / Blend / NFTfi 订单查询。
- **MetaStreet Vaults**：把 pool LP tokenize 成 ERC-4626 Vault。
- **Gondi**：大型 P2P 借贷 + appraisal 聚合，借鉴 NFTfi 思路。

## 4. 关键代码 / 实现细节

BendDAO `LendPool.borrow` 简化版（参考 `BendDAO/bend-protocol/contracts/protocol/LendPool.sol:~200`，v1 tag）：

```solidity
function borrow(
    address asset, uint256 amount,
    address nftAsset, uint256 nftTokenId,
    address onBehalfOf, uint16 referralCode
) external override nonReentrant {
    _borrow(BorrowLocalVars({
        initiator: msg.sender, asset: asset, amount: amount,
        nftAsset: nftAsset, nftTokenId: nftTokenId,
        onBehalfOf: onBehalfOf, referralCode: referralCode
    }));
}

function _borrow(BorrowLocalVars memory vars) internal {
    // 1. 读取 reserve / NFT 配置
    DataTypes.ReserveData storage reserve = _reserves[vars.asset];
    DataTypes.NftData storage nftData = _nfts[vars.nftAsset];

    // 2. 预言机读取 floor price，并计算最大可贷
    uint256 nftPrice = INFTOracle(_addressesProvider.getNFTOracle())
        .getAssetPrice(vars.nftAsset);
    uint256 maxBorrow = nftPrice.percentMul(nftData.ltv);
    require(vars.amount <= maxBorrow, "borrow > ltv");

    // 3. 转入 NFT，铸造 bNFT，记录 loan
    IERC721(vars.nftAsset).safeTransferFrom(vars.initiator, address(this), vars.nftTokenId);
    uint256 loanId = ILendPoolLoan(_addressesProvider.getLendPoolLoan())
        .createLoan(vars.onBehalfOf, vars.nftAsset, vars.nftTokenId, vars.asset, vars.amount);

    // 4. 从 Reserve 拨付 asset 给 onBehalfOf
    reserve.updateState();
    IERC20(vars.asset).safeTransfer(vars.onBehalfOf, vars.amount);
    reserve.updateInterestRates(vars.asset, 0, vars.amount);
}
```

Blend `takeLoan` 伪代码（Paradigm 2023 blog + audit 报告）：

```solidity
function takeLoan(
    LoanOffer calldata offer,
    bytes calldata offerSig,
    uint256 loanAmount,
    uint256 collateralTokenId
) external {
    // 1. 验证 lender 签名 EIP-712
    _verifyOffer(offer, offerSig);
    require(loanAmount >= offer.minLoanAmount && loanAmount <= offer.maxLoanAmount, "amount");

    // 2. 发送 ETH 给 borrower，锁定 NFT
    IERC721(offer.collection).transferFrom(msg.sender, address(this), collateralTokenId);
    (bool ok,) = msg.sender.call{value: loanAmount}("");
    require(ok, "eth xfer");

    // 3. 建立永续贷记录（无 maturity 字段，通过 refinance 拍卖淘汰）
    _loans[nextLoanId++] = Loan({
        lender: offer.lender, borrower: msg.sender,
        collection: offer.collection, tokenId: collateralTokenId,
        principal: loanAmount, rate: offer.rate, startTime: block.timestamp
    });
}
```

## 5. 演进与版本对比

| 协议 | 时间 | 模型 | 创新点 |
| --- | --- | --- | --- |
| NFTfi | 2020-05 | P2P | 最早 NFT 借贷 |
| Arcade v1 | 2022-01 | P2P | 多抵押篮子、Promissory Note |
| BendDAO | 2022-03 | P2Pool | 即时借贷 + 闪电首付 |
| MetaStreet | 2022-Q4 | Pool + 分级 | Vault/ERC-4626 |
| Paraspace | 2022-11 | P2Pool + 借贷 | 支持 LP token 抵押 |
| JPEG'd | 2022 | CDP | NFT 抵押铸 PUSd 稳定币 |
| Blend | 2023-05 | P2P + 永续 | Refinance 拍卖、无预言机 |
| Gondi | 2023 | P2P 优化 | 机构化 UX |
| MetaStreet Automatic | 2024 | Vault 自动化 | 池化 + 再贷款 |

## 6. 实战示例

用 ethers.js 与 BendDAO 合约交互，抵押 BAYC 借 30 WETH：

```ts
import { ethers } from "ethers";
const provider = new ethers.JsonRpcProvider(process.env.RPC);
const wallet = new ethers.Wallet(process.env.PK!, provider);

const BendLendPool = "0x70b97A0da65C15dfb0FFA02aEE6FA36e507C2762";
const BAYC = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";

const lendPool = new ethers.Contract(BendLendPool, [
  "function borrow(address asset,uint256 amount,address nftAsset,uint256 nftTokenId,address onBehalfOf,uint16 referralCode)"
], wallet);

// 1. Approve BAYC 给 LendPool
const bayc = new ethers.Contract(BAYC, [
  "function setApprovalForAll(address,bool)"
], wallet);
await bayc.setApprovalForAll(BendLendPool, true);

// 2. 发起借款
const tokenId = 1234; // 用户持有的 BAYC
const amount = ethers.parseEther("30");
const tx = await lendPool.borrow(WETH, amount, BAYC, tokenId, wallet.address, 0);
console.log("tx:", tx.hash);
```

预期：交易成功后，用户获得 30 WETH，同时钱包里出现一个 bBAYC（bound NFT，地址 `0xDBfD76AF2157Dc15eE4e57F3f942bB45Ba84aF24` 附近）代表抵押凭证；`getAssetPrice` 返回的 floor 可通过 Dune 监控。

## 7. 安全与已知攻击

1. **BendDAO Bank Run（2022-08）**：ETH 利用率飙至 100%，lender 想取 ETH 但池空。根因：熊市下 BAYC floor 跌，触发清算，但买方对拍卖冷淡（48h 宽限 + 5% bonus 不足以吸引底部接盘）；借款人 roll 过期继续借，池子循环枯竭。缓解：DAO 紧急降 LTV 至 40%、加清算 bonus、缩短拍卖期。
2. **JPEG'd Curve 事件（2023-07）**：读写重入导致 Curve pETH/ETH 池被攻，JPEG'd 损失 ~$11M。虽非核心 NFTfi 逻辑，但 PUSd 稳定币受影响。
3. **NFTfi Front-running**：P2P offer 未设 salt 时，borrower 可以等对方报价后 back-run 接受更优的，NFTfi v2 加入 nonce 修复。
4. **Blend 拍卖拖延 grief**：borrower 可刻意让 lender 触发 refinance 又接回，浪费 gas。Blend v2 增加冷却时间。
5. **Arcade Promissory Note Spoofing**：早期版本若 note NFT 被二次转移，可能出现贷款 ownership 争议；v3 加强了 `loanId ↔ note` 绑定。
6. **Paraspace 清算竞争（2023）**：空投 token 抵押率计算 bug 导致部分头寸不可清算。

## 8. 与同类方案对比

| 维度 | BendDAO | Blend | Arcade | NFTfi |
| --- | --- | --- | --- | --- |
| 模式 | P2Pool | P2P + 永续 | P2P | P2P |
| 借款人 UX | 即时 | 即时（匹配快） | 个性化 | 个性化 |
| 出借人 UX | 存币生息 | 报价 / 接管 | 报价 | 报价 |
| 预言机 | 多市场 TWAP | 无（拍卖驱动） | 链下 appraisal | 双方自定 |
| 清算 | 荷兰拍 48h | Refinance 荷兰拍 | 违约直转 | 违约直转 |
| 支持抵押 | 头部 10 种 | 所有 Blur 支持系列 | 多资产篮子 | 任意 |
| 历史风险事件 | 2022 Bank Run | Grief 攻击 | 小规模 | 老协议较稳 |

碎片化替代方案：NFTX（vToken + LP 挖矿）、Sudoswap（NFT AMM）、Unic.ly（已停运）、Fractional/Tessera（2023 停运）。当前更流行是「NFT AMM + 借贷」的组合：Sudoswap pool 作为抵押，MetaStreet 池化再贷款。

## 9. 延伸阅读

- **官方文档**：docs.benddao.xyz、docs.blur.io/blend/overview、docs.arcade.xyz、docs.nftfi.com。
- **Paradigm 文章**：*Blend: A Perpetual Peer-to-Peer NFT Lending Protocol* (2023-05)。
- **博客**：Dan Robinson / Transmissions11 / Tarun Chitra 等关于 NFTfi 设计的 Twitter 讨论。
- **论文**：Chitra & Evans *Approaches to Lending NFTs* (2022)。
- **数据**：Dune（`@rchen8/nft-finance-overview`、`@sealaunch/benddao`）、DappRadar NFTfi 榜。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| LTV | Loan-to-Value | 贷款价值比 |
| Liquidation Threshold | LT | 清算阈值 |
| P2P Lending | Peer-to-Peer Lending | 点对点贷款 |
| P2Pool | Peer-to-Pool | 池化借贷 |
| Refinance Auction | Refinance Auction | 再融资拍卖 |
| Floor Price | Floor | 地板价 |
| bNFT | Bound NFT | 借贷抵押凭证 |
| Buyout | Buyout | 碎片化 NFT 的回购拍卖 |
| Vault | NFT Vault | 托管 NFT 的容器合约 |
| Down Payment | Down Payment | 首付式借贷购买 |
| CDP | Collateralized Debt Position | 抵押债务仓位 |

---

*Last verified: 2026-04-22*
