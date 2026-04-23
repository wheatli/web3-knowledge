---
title: NFT 市场对比：OpenSea / Blur / Magic Eden / Tensor
module: 04-dapp/nft
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://github.com/ProjectOpenSea/seaport
  - https://docs.opensea.io/
  - https://docs.blur.io/
  - https://docs.magiceden.io/
  - https://docs.tensor.trade/
---

# NFT 市场对比：OpenSea / Blur / Magic Eden / Tensor

> **TL;DR**：NFT 市场是 NFT 经济中最核心的二级流通入口。OpenSea（2017 上线、Seaport 协议、多链覆盖）长期占统治地位，但 2022 年 Blur 以专业交易者体验（限价深度、Bid Pool、代币激励）在 Ethereum 上反超，2023–2024 年市场份额一度 70%+。Solana 生态中 Magic Eden 起家早、兼容性强（NFT + Compressed NFT + Ordinals 多链聚合），而 Tensor 则以「CEX 式订单簿 + SDK」在专业交易者中崛起。本篇从订单协议、流动性模型、激励机制、版税与 OFAC 策略、SDK 接口五维对比，并给出技术架构与安全事件复盘。

## 1. 背景与动机

NFT 市场本质是 **非同质化资产的撮合 + 结算层**。与 DEX 不同，NFT 资产单价差异巨大、并非完全可替代，因此无法直接用 AMM，而是主要依靠 **订单簿 + 签名撮合**（listing + matching）。

第一代 NFT 市场（OpenSea 2017、Rarible 2020、LooksRare 2022、X2Y2 2022）沿用 0x Protocol / Wyvern 等签名协议，体验偏向休闲藏家：侧重发现（discovery）、收藏（collection page）、社交（profile），交易深度和订单延迟不适合高频交易者。

2022 年 10 月 Blur.io 上线，针对专业交易者痛点进行重构：(a) 实时 floor price 流；(b) 多 NFT 批量 list / sweep；(c) Bid Pool 集合竞价；(d) 免平台费 + 代币激励。2023 年 2 月 BLUR 代币 TGE，Season 2/3 空投把 Ethereum NFT 现货 + bid 流动性全面拉到 Blur。到 2023 Q3，Blur 占 Ethereum NFT 交易量 70–80%。

Solana 生态走了一条略不同的路。Magic Eden 2021 年 9 月上线，抓住 Solana 低 gas + y00ts / DeGods / Okay Bears 等项目热度，一度跨链到以太坊 / Polygon / Bitcoin Ordinals，但在 2022–2023 年被 Tensor 等专业产品蚕食 Solana 高频市场。Tensor 采用类似 Blur 的激励 + 专业交易者界面，2023 年 TNSR 空投后占据 Solana 50%+。

## 2. 核心原理

### 2.1 形式化定义：签名撮合模型

一个 NFT 订单可以形式化为一个签名消息 $O$：

$$
O = (\text{offerer}, \text{offer}, \text{consideration}, \text{zone}, \text{startTime}, \text{endTime}, \text{salt}, \text{type}, \sigma)
$$

其中 $\text{offer}$ 是「卖出集合」（NFT 或 ERC-20），$\text{consideration}$ 是「买入集合 + 分配列表」（含买家/卖家/版税/平台分配），$\text{zone}$ 是可选的执行策略合约（校验时能否成交），$\sigma$ 是 EIP-712 签名。撮合合约 $M$ 执行 $M(O, \text{fulfill\_params})$ 需验证：签名有效、时间窗内、offer/consideration 存在可转移、订单未被取消、zone 返回 ok。

### 2.2 订单簿与流动性模型

- **Listing Book（挂单簿）**：卖家签名「我以 X ETH 卖 tokenId = T」。属于 ask side。
- **Collection Offer / Trait Offer**：买家对 floor 或特定 trait 批量出价。OpenSea 的 Collection Offer 与 Blur 的 Bid Pool 在此层。
- **Bid Pool（Blur 独创）**：多个买家按 tick 合并挂单（例：tick=0.01 ETH），任何一个卖家 accept 时 FIFO 吃单。Bid Pool 大幅提升 bid-side 深度并成为 BLUR 激励核心。
- **AMM 形态（Sudoswap / Caviar）**：用 LP 形式做市 NFT，曲线为 linear 或 exponential；适合流动性高、同质性强的 collection。

### 2.3 激励机制：交易挖矿 + 忠诚度

Blur 通过三个 Season 向用户发 BLUR：按 listing 质量（价差窄奖励高）、bid 规模与深度、成交量 loyalty 分数计算积分，TGE 后按积分 → BLUR 兑换。关键约束：**Loyalty**（只在 Blur 挂单，不去 OpenSea）、**Bid points**（挂 bid 而非 listing 得分更高）。这种设计刻意激发 bid-side 流动性，对专业交易者尤其有吸引力。

Tensor 沿用类似思路：TNSR 代币 + Season + Loyalty。Magic Eden 2023 年推出 ME Rewards，2024 年 ME 空投 $ME 代币。

### 2.4 Seaport 协议关键子机制

OpenSea 2022 年发布 Seaport 1.x（MIT 许可），后续升级到 Seaport 1.5 / 1.6。核心创新：

- **Offer/Consideration 抽象**：一笔订单可以任意组合 ERC-20/721/1155，支持「以 NFT 换 NFT」。
- **Zones**：可组合的授权层，实现时间锁、版税执行、私有拍卖。
- **Criteria-based Orders**：用 Merkle root 在一条订单覆盖多个 tokenId（collection offer 底层）。
- **Gas 优化**：大量使用 assembly，批量 match 相比 Wyvern 节省 35% gas。

### 2.5 参数与常量

| 参数 | OpenSea | Blur | Magic Eden (SOL) | Tensor (SOL) |
| --- | --- | --- | --- | --- |
| 平台费 | 0.5%（2023 调整，曾为 2.5%） | 0% | 2% | 1.5% |
| 版税执行 | 可选（创作者可设置 enforce zone） | 最低 0.5%，可选更高 | 强制（曾经） → 可选 | 可选 |
| 支持链 | 以太坊、Arbitrum、Base、Polygon、Optimism、Solana、BNB 等 | Ethereum（含 L2 有限） | Solana、Ethereum、Polygon、Bitcoin Ordinals | Solana |
| 订单类型 | Seaport 签名 | Blur.io proprietary + Seaport | Tensor 风格订单簿 | 订单簿 + LP |
| 代币激励 | — | BLUR Season | ME Rewards, $ME | TNSR |

### 2.6 边界条件与失败模式

- **签名失效**：用户 transfer NFT 后未 cancel listing，新钱包 approve 同 conduit 时旧签名仍可被触发（OpenSea 2022 历史 exploit）。
- **Front-run**：Mempool 中抢 accept 他人 bid；Blur 私有 RPC 缓解。
- **订单取消 gas**：OpenSea 允许链下取消（off-chain cancel），Blur 则链上 + 链下混合。
- **版税绕过**：市场通过 zone / operator filter 强制，但买卖双方可转向不 enforce 的市场。
- **Wash Trading**：自刷交易获取激励；2022 LooksRare 被批评 95%+ wash，Blur 与 Tensor 后续引入反刷量检测。

## 3. 架构剖析

### 3.1 分层视图

1. **前端与 API 层**：web / mobile SPA，调用聚合索引 API。
2. **索引层**：Reservoir（跨市场）、OpenSea API、Tensor API、Magic Eden API、Simplehash、Alchemy NFT API。
3. **订单协议层**：Seaport（EVM）、Blur proprietary + Seaport fallback、Magic Eden AuctionHouse（Anchor 合约）、Tensorswap / Tensor 订单簿。
4. **结算层**：EVM NFT 合约直接 transfer；Solana SPL + Metaplex。
5. **代币激励与治理层**：BLUR、TNSR、$ME、OpenSea（尚未发币）。

### 3.2 核心模块清单

| 模块 | 市场 | 职责 | 依赖 |
| --- | --- | --- | --- |
| Seaport.sol | OpenSea / Reservoir / Zora 通用 | 订单撮合合约 | EIP-712 |
| OpenSea Conduit | OpenSea | 统一 token approval 管道 | ERC721/1155 |
| Blur Exchange | Blur | 专有撮合合约，支持批量 | 自研签名协议 |
| Bid Pool | Blur | 按 tick 聚合买方流动性 | Blur Exchange |
| Auction House Program | Magic Eden (SOL) | PDA 承载订单 | Anchor |
| Tensorswap | Tensor | LP AMM + 订单簿 | Anchor |
| Reservoir Router | 聚合器 | 跨市场一键购买 | 各市场 SDK |
| Operator Filter | 版税执行 | 注册表 + 合约 hook | — |

### 3.3 数据流 / 生命周期

以「Alice 在 Blur 挂 BAYC #1234 给 Bob」为例：

1. **签名挂单**：Alice 前端生成 Blur 订单，调用 wallet `eth_signTypedData_v4`（EIP-712）。此步不产生 gas。
2. **API 广播**：前端把签名订单 POST 到 Blur API，Blur 执行反欺诈检查后写入订单簿。
3. **索引同步**：Reservoir 通过 webhook 拉取并聚合，OpenSea 则共享部分订单簿。
4. **Bob 接单**：Bob 在 Blur 点击 buy，前端调用 `BlurExchange.bulkExecute(orders, signatures, takerInputs)`，gas 由 Bob 付。
5. **链上转移**：Blur Exchange 校验签名 → 转账 ETH/WETH → 转移 NFT → 触发版税分配（若设置）。
6. **后置事件**：Transfer / OrderFilled 事件被多家 indexer 同步，钱包 UI 刷新。
7. **激励积分**：Blur 后端根据成交额和 loyalty 计入 Season 积分。

### 3.4 客户端多样性 / 参考实现

- **Seaport**：开源 Solidity（`ProjectOpenSea/seaport`），被 Zora、Reservoir、Magic Eden ETH、Flow 等多家使用。
- **Blur Exchange**：合约部分 verified on Etherscan，但后端撮合闭源。
- **Magic Eden**：Solana 版是开源 Anchor program；ETH / BTC 聚合器采用第三方接入。
- **Tensor**：Anchor program 开源，TensorSDK TypeScript 公开。

### 3.5 扩展 / 互操作接口

- **Reservoir API**：`/orders/asks/v4`、`/orders/bids/v5`、`/execute/buy/v7`，面向所有 EVM NFT 市场的统一 SDK；聚合 OpenSea、Blur、LooksRare、X2Y2。
- **Blur API**：需要 API Key 的私有端点；Reservoir 是事实上的二级封装。
- **OpenSea API v2**：REST + GraphQL，支持 listing / bid / collection / trait 查询。
- **Magic Eden API**：SOL/ETH/BTC 多链统一，支持 inscription 交易。
- **Tensor SDK**：TS SDK + REST，支持订单簿 WS 推送。

## 4. 关键代码 / 实现细节

Seaport `_matchAdvancedOrders` 核心在 `Seaport.sol:matchAdvancedOrders`（`ProjectOpenSea/seaport@1.6`，contract 地址 `0x0000000000000068F116a894984e2DB1123eB395`）。简化片段：

```solidity
// ProjectOpenSea/seaport@1.6, contracts/Seaport.sol:~260 (参考实际实现)
function matchAdvancedOrders(
    AdvancedOrder[] calldata orders,
    CriteriaResolver[] calldata criteriaResolvers,
    Fulfillment[] calldata fulfillments,
    address recipient
) external payable returns (Execution[] memory) {
    // 1. 校验每一笔订单 (EIP-712 签名 / zone / time window)
    _validateOrdersAndPrepareToFulfill(orders, criteriaResolvers, false, type(uint120).max, recipient);
    // 2. 匹配 offer 和 consideration（多对多 NFT 互换）
    return _fulfillAdvancedOrders(orders, criteriaResolvers, fulfillments, recipient);
}
```

Blur Bid Pool 在 `BlurPool.sol:deposit`（合约地址 `0x0000000000A39bb272e79075ade125fd351887Ac`，Etherscan verified）：

```solidity
// Simplified: BlurPool deposit + execute
contract BlurPool {
    mapping(address => uint256) public balances;
    function deposit() external payable { balances[msg.sender] += msg.value; }
    function execute(
        bytes32 orderHash, address bidder, uint256 tokenId,
        address collection, uint256 price, bytes calldata sig
    ) external onlyExchange {
        require(balances[bidder] >= price, "insufficient");
        balances[bidder] -= price;
        IERC721(collection).transferFrom(msg.sender, bidder, tokenId);
        payable(msg.sender).transfer(price);
    }
}
```

关键点：Blur 通过把买家流动性统一池化，成交可 O(1) 完成，且买家无需每次 approve；卖家 accept bid 时避免 front-run。

## 5. 演进与版本对比

| 版本 / 事件 | 时间 | 关键变化 |
| --- | --- | --- |
| OpenSea 上线 | 2017-12 | 第一代通用 NFT 市场 |
| Wyvern v2 | 2018 | 订单协议基础 |
| LooksRare | 2022-01 | 交易挖矿 + 通证激励先例 |
| X2Y2 | 2022-02 | Royalty Flexible + Loan 尝试 |
| Seaport 1.0 | 2022-05 | OpenSea 自研协议替换 Wyvern |
| Blur beta | 2022-10 | 专业交易者 UI + 空投 |
| Blur TGE | 2023-02 | BLUR 激励 + Bid Pool 崛起 |
| Seaport 1.5 | 2023 | 版税 zone、Creator Fee 恢复 |
| Blur Blend 借贷 | 2023-05 | NFT 点对点借贷（详见 nftfi.md）|
| Ordinals 兼容 | 2023-04 | Magic Eden / OKX 上线 Ordinals |
| TNSR 空投 | 2024-04 | Tensor Solana 激励 |
| ME 空投 | 2024-10 | Magic Eden 代币化 |

## 6. 实战示例

使用 Reservoir SDK 聚合 OpenSea + Blur 挂单，一键 sweep BAYC floor：

```bash
npm i @reservoir0x/reservoir-sdk viem
```

```ts
import { createWalletClient, http } from "viem";
import { mainnet } from "viem/chains";
import { getClient, createClient, reservoirChains } from "@reservoir0x/reservoir-sdk";
import { privateKeyToAccount } from "viem/accounts";

createClient({ chains: [{ ...reservoirChains.mainnet, active: true }], source: "demo" });

const account = privateKeyToAccount(process.env.PK as `0x${string}`);
const wallet = createWalletClient({ account, chain: mainnet, transport: http() });

const bayc = "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D";
await getClient().actions.buyToken({
  items: [{ collection: bayc, quantity: 2 }],
  wallet,
  onProgress: (steps) => console.log(steps),
});
```

预期：SDK 自动从 Reservoir orderbook 聚合 OpenSea/Blur/X2Y2 的最便宜两单，构造一笔或多笔交易批量执行；控制台打印 step 状态；最终钱包收到 2 个 BAYC。

## 7. 安全与已知攻击

1. **OpenSea 旧挂单 bug（2022-01）**：用户 transfer NFT 到新钱包时旧挂单未取消，攻击者以旧价格买走。根因：OpenSea 链下挂单，`approve` 仍有效。修复：前端强制提示 revoke + 提供 bulk cancel。
2. **OpenSea 钓鱼攻击（2022-02）**：Discord 钓鱼链接让用户签署恶意订单，损失 ~$1.7M NFT。
3. **LooksRare Wash Trading 危机（2022-02）**：自刷生成激励高达 $8B 刷量，项目后期调整 reward。
4. **Blur Airdrop Sniping**：Loyalty 机制导致卖家把 NFT 挂 Blur 不挂 OpenSea，间接推动版税崩盘（详见 `royalty-and-standards.md`）。
5. **Magic Eden Solana Enforce 回滚（2022-11）**：ME 一度强制 Royalty 引发 DeGods 等项目迁 Opensea 分发，后 ME 软化立场。
6. **Tensor Wormhole 流动性事件**：某些桥接 NFT 在 Tensor 上无法正常 list，原因是 metadata 格式不兼容。

## 8. 与同类方案对比

| 维度 | OpenSea | Blur | Magic Eden | Tensor |
| --- | --- | --- | --- | --- |
| 旗舰链 | Ethereum + 多链 | Ethereum | Solana + 多链 | Solana |
| 协议 | Seaport（开源）| 专有 + Seaport | Anchor + 多链接入 | Anchor |
| 平台费 | 0.5% | 0% | 2% | 1.5% |
| 代币 | 无 | BLUR | ME | TNSR |
| 专业功能 | 一般 | 最佳（sweep/bid pool）| 中等 | 接近 Blur |
| 创作者版税策略 | 2023 恢复 enforce | 0.5% 强制 + 可选 | 可选 | 可选 |
| 聚合能力 | 站内为主 | 站内 + Gem | 多链聚合 | 主要 Solana |
| OFAC 政策 | 严格 | 严格 | 严格 | 严格 |

## 9. 延伸阅读

- **官方文档**：docs.opensea.io、docs.blur.io、docs.magiceden.io、docs.tensor.trade。
- **Seaport 合约**：ProjectOpenSea/seaport GitHub + Seaport whitepaper。
- **博客**：OpenSea Engineering Blog、Blur Twitter Spaces 回放、a16z《NFT Marketplaces: The Unbundling》、Delphi Digital NFT 季报。
- **数据源**：Dune（@hildobby、@rchen8）、NFTGo、CryptoSlam、tiltMeta。
- **学术**：*The NFT Market: A Study of the OpenSea Ecosystem* (ACM WWW 2023)。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 挂单 | Listing | 出售订单 |
| 出价 | Bid / Offer | 买家出价 |
| 地板价 | Floor Price | 最低挂单价 |
| Sweep | Sweep the Floor | 批量扫货地板 |
| Bid Pool | Bid Pool | Blur 集合买单池 |
| Loyalty | Loyalty | Blur 只挂一家的忠诚度 |
| 聚合器 | Aggregator | 跨市场订单聚合（Gem/Reservoir） |
| Conduit | OpenSea Conduit | 统一授权管道合约 |
| Zone | Seaport Zone | 订单授权与策略层合约 |
| 签名撮合 | Off-chain Order | 链下签名 + 链上结算 |
| Operator Filter | Operator Filter Registry | 版税执行名单 |
| Metaplex | Metaplex | Solana NFT 标准实现 |

---

*Last verified: 2026-04-22*
