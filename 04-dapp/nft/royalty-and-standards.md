---
title: NFT 版税标准与创作者版税之争（EIP-2981 / Operator Filter）
module: 04-dapp/nft
priority: P2
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://eips.ethereum.org/EIPS/eip-2981
  - https://github.com/ProjectOpenSea/operator-filter-registry
  - https://docs.opensea.io/docs/creator-fee-enforcement
  - https://github.com/limitbreakinc/creator-token-standards
  - https://docs.blur.io/royalty-policy
---

# NFT 版税与标准：EIP-2981、创作者版税之争与 Operator Filter

> **TL;DR**：创作者版税（Royalty）让 NFT 创作者在每次二级市场转售时获得 2.5–10% 分成。EIP-2981 定义了「如何查询」，但未规定「如何强制」。2022 年 Sudoswap / X2Y2 / Blur 相继推出零版税或可选版税，引发持续一整年的版税战争。OpenSea 2022 年 11 月发布 Operator Filter Registry 试图用黑名单限制 NFT 转移到不 enforce royalty 的市场；限制仅在新项目生效，Yuga 等头部旧合约无法追溯。2023 年 Limit Break 的 Creator Token Standards（ERC-721-C）通过合约内置 `transferValidator` 把强制权收回创作者手中。本篇讲清 EIP-2981 字段语义、Operator Filter 的两个版本 / 三种 deny-list 策略、ERC-721-C 的执行机制以及各市场的现行版税政策。

## 1. 背景与动机

在传统艺术品市场，ARR（Artist Resale Right）法律上在欧盟等地区强制要求二级销售分成创作者 3–4%。Web3 NFT 在 2017–2021 年期间普遍接受 5–10% 版税作为社会契约，成为创作者主要收入来源之一（例：Yuga Labs 2021 年版税收入估 > $127M）。但协议层没有任何机制能强制让新的买家执行 `royaltyInfo` 返回的转账——这是一个 **社会共识而非技术共识**。

版税执行链路的薄弱点在于 NFT 转移本身（`safeTransferFrom`）与销售无绑定：合约层看不到「这次转账是不是买卖」。市场（如 OpenSea）在撮合合约里插入版税转账逻辑是完全自愿的。2022 年 Sudoswap 推出 NFT AMM（把 NFT 和 ETH 做成曲线型流动性池），一笔成交本质上是 pool swap 而非带 royalty 的 sale；很多项目因此发现 Sudoswap 上的销量不付版税。X2Y2 随后允许 buyer 在前端选择版税 0–10%。Blur 默认 0.5% 最低版税并允许 0% 挂单（2022.10，之后多次调整）。2023 年初创作者版税收入下降 80%+。

为阻止这股趋势，OpenSea 2022 年 11 月发布 Operator Filter Registry：新项目若 opt-in，其 NFT 合约的 `transferFrom` 会检查 `operator`（通常是市场合约）是否在黑名单中。若在，转移 revert，销售自然失败。但 Filter 是可选的，且只能用于新部署合约。2023 年 8 月 OpenSea 宣布放弃 Operator Filter，回归「可选版税」。创作者不得不寻找更强的技术手段，Limit Break 推出的 ERC-721-C 把转账授权器直接内置到合约层。

## 2. 核心原理

### 2.1 EIP-2981 形式化定义

标准接口（MUST 实现）：

```solidity
interface IERC2981 is IERC165 {
  function royaltyInfo(uint256 _tokenId, uint256 _salePrice)
      external view returns (address receiver, uint256 royaltyAmount);
}
```

语义：给定 tokenId 与销售价，返回应支付的收款地址与金额。合约开发者可以按 tokenId 动态返回（例：不同 artist 归不同 receiver），或 collection-wide 返回同一值。注意：**标准不规定计算基准**（整 collection 还是 per-token）、**不规定由谁支付**（卖家让利还是买家加价）、**不规定谁强制**（完全依赖市场）。因此 EIP-2981 只是 **可查询协议**（discoverable），并非 **可强制协议**（enforceable）。

EIP-165 supportsInterface ID = `0x2a55205a`。

### 2.2 版税计算公式与发放路径

当成交价 $P$、版税比例 $r$、平台费 $f$、时：
- 卖家到手：$P \cdot (1 - r - f)$
- 创作者收到：$P \cdot r$
- 市场收到：$P \cdot f$

发放有两种模式：(a) **撮合合约直接分配**（OpenSea Seaport 的 consideration 列表写死 royalty 收款人）；(b) **代理合约分配**（市场调用 NFT 合约的 payout 函数）。前者方便但依赖市场自觉，后者更灵活但需在 NFT 合约里实现。

### 2.3 Operator Filter 子机制

OpenSea Operator Filter Registry（OFR）合约位于 `0x000000000000AAeB6D7670E522A718067333cd4E`，提供两个核心接口：

- `isOperatorAllowed(address registrant, address operator) external view returns (bool)`：查询某个 operator 是否被某 registrant（通常是 NFT collection）允许。
- `OperatorFilterer` 抽象合约：在 ERC-721 的 `_beforeTokenTransfer` 或 `transferFrom` 注入 modifier `onlyAllowedOperator(from)`，若 operator（`msg.sender`）被阻止则 revert。

三种策略（按严格度）：
1. **Deny list（默认）**：OpenSea 维护的「零版税市场」黑名单（Blur、LooksRare、Sudoswap 等）。
2. **Custom list**：项目自定义允许/禁止的 operator。
3. **Opt-out（2023.08 后默认）**：不启用 filter。

### 2.4 ERC-721-C（Limit Break，2023）

核心思想：把 transfer 授权判断外置到 `transferValidator` 合约，collection owner 可以随时切换验证器。伪代码：

```solidity
function _beforeTokenTransfer(address from, address to, uint256 tokenId) internal {
    address v = getTransferValidator();
    if (v != address(0)) {
        ICreatorTokenTransferValidator(v).validateTransfer(msg.sender, from, to, tokenId);
    }
}
```

Validator 合约通常采取 **payment processor** 模式：只允许通过认可的 payment processor（内含版税强制逻辑）调用 transfer。未经 processor 的 transfer 直接 revert。这样即便 Blur 接收到用户签名，也无法完成交易——因为 transferFrom 会失败。

### 2.5 参数与常量

| 参数 | 默认值 | 可治理 | 说明 |
| --- | --- | --- | --- |
| ERC-2981 royalty bps | 项目自定 | 可 | 返回 `salePrice * bps / 10000` |
| OperatorFilter 默认 deny list | ~12 个 operator | OpenSea 维护 | Blur、LooksRare、Sudo 等 |
| ERC-721-C Validator | 项目自设 | 随时切换 | 默认指向 Limit Break V2 |
| PaymentProcessor min royalty | 0 | 项目自设 | 强制下限 |

### 2.6 边界条件与失败模式

- **Fork 市场绕过**：即使 OFR 阻止 Blur Exchange 合约，Blur 可部署新合约并更换 operator 地址，绕过黑名单。黑名单类方案属于「运营对抗」。
- **P2P 转账豁免**：OFR 通常只拦截 operator，用户自己手动 transfer 到朋友地址不受限。但这也意味着私下场外交易仍可零版税。
- **Royalty Splitting**：多 receiver 场景下 ERC-2981 只能返回一个地址，项目通常用分账合约（PaymentSplitter）中转。
- **Dynamic Royalty**：有的项目想按交易额阶梯收，需链下前端辅助。
- **Marketplace 恶意**：撮合合约可能 bypass 版税，需要 ERC-721-C 式验证器才真正强制。

## 3. 架构剖析

### 3.1 分层视图

1. **标准层**：EIP-2981（查询）、ERC-165（接口发现）、EIP-7572（contract-level metadata，含版税）。
2. **合约层**：OZ `ERC721Royalty`、Limit Break `ERC721C`。
3. **执行层**：Seaport zones、Operator Filter、Payment Processor。
4. **市场层**：OpenSea、Blur、X2Y2、Magic Eden、Tensor 各自策略。
5. **社会层**：艺术家联合体（Yuga、Azuki、Nouns）与交易者联盟（Blur、Sudo）的博弈。

### 3.2 核心模块清单

| 模块 | 职责 | 代码仓库 | 可替换性 |
| --- | --- | --- | --- |
| ERC2981 (OZ) | 标准实现 | `openzeppelin-contracts/.../ERC2981.sol` | 高 |
| OperatorFilterRegistry | 维护 deny list | `ProjectOpenSea/operator-filter-registry` | 中等 |
| DefaultOperatorFilterer | 合约 modifier | 同上 | 高 |
| CreatorTokenStandards | ERC-721-C | `limitbreakinc/creator-token-standards` | 高 |
| PaymentProcessor V2 | 版税强制撮合 | Limit Break | 中等 |
| Seaport Zones | 版税执行策略 | OpenSea | 高 |
| Transfer Validator | ERC-721-C 注入点 | LB / 自研 | 高 |

### 3.3 数据流 / 生命周期

以「Alice 在 Magic Eden Ethereum 上卖一个 Azuki」为例：

1. **前端获取版税**：ME 前端调用 Azuki 合约 `royaltyInfo(tokenId, salePrice)` 获取 receiver + amount。
2. **订单构造**：Seaport 订单中 `consideration` 列表包含 (卖家 95%)、(创作者 5%)、(平台 0.5%)。
3. **Operator 检查**：买家 `fulfillOrder` 触发 Azuki.transferFrom，ERC-721-C 的 `validateTransfer` 被调用，检查 operator 是否为认可 PaymentProcessor。
4. **通过 → 转账 ETH + NFT**：分账 + transfer。若操作方不在白名单，revert。
5. **链上事件**：Transfer + OrderFilled，版税收入写入 Alice 的链上记录。

### 3.4 客户端多样性 / 参考实现

- **OZ ERC2981**：最通用，适合 PFP / 艺术品。
- **ERC-721-C (Limit Break)**：适合希望强制版税的大型项目。2023 年末 DigiDaigaku、Nouns 的新系列、PROOF Collective 等采用。
- **Manifold Creator Contracts**：Manifold Studio 提供高度可配置的版税路由合约，服务艺术家。
- **ArtBlocks Engine**：平台侧把版税分账合约内建。

### 3.5 扩展 / 互操作接口

- **EIP-4494**：Permit for ERC-721，降低 approve 成本。
- **EIP-5553 / EIP-5773**：多收益人版税拓展。
- **EIP-6105**：NFT 市场订单标准（有限推广）。
- **EIP-7572**：合约级 metadata（含 royaltyInfo）。

## 4. 关键代码 / 实现细节

OpenZeppelin ERC2981 参考实现（`openzeppelin-contracts@5.0.0`，`contracts/token/common/ERC2981.sol:66`）：

```solidity
// @openzeppelin/contracts/token/common/ERC2981.sol:66
function royaltyInfo(uint256 tokenId, uint256 salePrice)
    public view virtual returns (address, uint256)
{
    RoyaltyInfo storage royalty = _tokenRoyaltyInfo[tokenId];
    if (royalty.receiver == address(0)) {
        royalty = _defaultRoyaltyInfo;
    }
    uint256 royaltyAmount = (salePrice * royalty.royaltyFraction) / _feeDenominator();
    return (royalty.receiver, royaltyAmount);
}

function _setDefaultRoyalty(address receiver, uint96 feeNumerator) internal virtual {
    require(feeNumerator <= _feeDenominator(), "ERC2981: royalty fee exceeds salePrice");
    require(receiver != address(0), "ERC2981: invalid receiver");
    _defaultRoyaltyInfo = RoyaltyInfo(receiver, feeNumerator);
}
```

Operator Filterer 合约（`ProjectOpenSea/operator-filter-registry@1.4.1`，`src/OperatorFilterer.sol:41`）：

```solidity
// src/OperatorFilterer.sol:41
modifier onlyAllowedOperator(address from) virtual {
    // Allow spending tokens from addresses with balance
    // Note that this still allows listings and marketplaces with escrow to transfer tokens if transferred
    // from an EOA.
    if (from != msg.sender) {
        _checkFilterOperator(msg.sender);
    }
    _;
}

function _checkFilterOperator(address operator) internal view virtual {
    address registry = address(OPERATOR_FILTER_REGISTRY);
    if (registry.code.length > 0) {
        if (!OPERATOR_FILTER_REGISTRY.isOperatorAllowed(address(this), operator)) {
            revert OperatorNotAllowed(operator);
        }
    }
}
```

Limit Break Creator Token `_beforeTokenTransfer`（`limitbreakinc/creator-token-standards@3.0`）：

```solidity
function _beforeTokenTransfer(address from, address to, uint256 tokenId) internal virtual override {
    super._beforeTokenTransfer(from, to, tokenId);
    address validator = transferValidator;
    if (validator != address(0)) {
        ICreatorTokenTransferValidator(validator).validateTransfer(msg.sender, from, to, tokenId);
    }
}
```

## 5. 演进与版本对比

| 事件 | 时间 | 关键变化 | 影响 |
| --- | --- | --- | --- |
| EIP-2981 final | 2020-09 | 标准化 royaltyInfo | 查询标准化 |
| Sudoswap 上线 | 2022-07 | AMM 绕过版税 | 创作者警惕 |
| X2Y2 可选版税 | 2022-08 | 买家选择付或不付 | Azuki 抗议 |
| Blur launch | 2022-10 | 0% 平台费，0.5% 最低版税 | 版税收入骤降 |
| OpenSea Operator Filter | 2022-11 | deny-list 强制 | 创作者联名 |
| Magic Eden 强制 → 可选 | 2022-10 → 2022-11 | 先强制后软化 | DeGods 迁出 |
| Blur 默认 0.5% → 0% | 2023-02 | BLUR 激励启动 | OpenSea 弃版税 enforce |
| OpenSea 放弃 Operator Filter | 2023-08 | 承认旧项目无法挽回 | 版税彻底可选 |
| Limit Break ERC-721-C | 2023-09 | 合约层强制 | 新项目可强制 |
| Yuga 采用 CTS 系列 | 2024 | BAYC 衍生项目使用 | 创作者回血 |

## 6. 实战示例

部署一个带 ERC-2981 的 NFT 并通过 OZ ERC2981 设置版税：

```solidity
// src/RoyaltyNFT.sol
pragma solidity ^0.8.24;
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/common/ERC2981.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract RoyaltyNFT is ERC721, ERC2981, Ownable {
    uint256 public nextId;
    constructor(address receiver, uint96 feeBps) ERC721("Royal", "RYL") Ownable(msg.sender) {
        _setDefaultRoyalty(receiver, feeBps); // 5% = 500
    }
    function mint(address to) external onlyOwner { _mint(to, ++nextId); }
    function supportsInterface(bytes4 id) public view override(ERC721, ERC2981) returns (bool) {
        return super.supportsInterface(id);
    }
}
```

```bash
forge create src/RoyaltyNFT.sol:RoyaltyNFT --constructor-args 0xArtist 500 \
  --private-key $PK --rpc-url http://localhost:8545
# 查询版税信息：1 ETH 价格
cast call $NFT "royaltyInfo(uint256,uint256)" 1 1000000000000000000
# 预期：receiver=0xArtist, amount=50000000000000000（0.05 ETH）
```

## 7. 安全与已知攻击

1. **Blur 创作者版税绕过**：Blur 默认 0.5% 最低版税的反面是允许用户手动调 0%；多数交易者选 0%，版税收入骤降。Yuga 最初 opt-in Operator Filter 阻断 Blur，但因 BAYC 合约已部署不可升级导致失效。
2. **Operator Filter 可绕过**：Blur 发布新合约地址逃过 OFR 黑名单，OFR 维护者持续更新但存在滞后。
3. **ERC-721-C PaymentProcessor 漏洞（假设）**：若 processor 合约逻辑存 bug（如 `validateSale` 未校验版税扣款），可能导致绕过；Limit Break 2023 年对 v1 → v2 进行审计升级。
4. **Royalty 合约 Re-entrancy**：分账合约若 call-then-effect 顺序错误，可能在版税接收人合约里回调 NFT 合约，需按 OZ ReentrancyGuard 保护。
5. **版税空投诱骗**：钓鱼者通过 Airdrop 可点击式 NFT 诱导受害者调用恶意 `transferValidator`，变相盗取 NFT。

## 8. 与同类方案对比

| 方案 | 强制力 | 可追溯旧合约 | 创作者控制力 | 代表用户 |
| --- | --- | --- | --- | --- |
| EIP-2981 裸接口 | 0 | 否 | 低 | 通用 |
| Seaport Zones | 弱（市场内） | 否 | 中 | OpenSea |
| Operator Filter | 中（新合约）| 否 | 中 | 2022 新系列 |
| ERC-721-C (Limit Break) | 强（合约层）| 否 | 高 | Yuga、PROOF |
| Manifold 智能合约分账 | 中 | 否 | 高 | 艺术家 |
| 法律诉讼 / 版权法 | — | 是 | 低 | — |

## 9. 延伸阅读

- **官方标准**：EIP-2981、EIP-4494、EIP-5773、EIP-7572。
- **GitHub**：ProjectOpenSea/operator-filter-registry、limitbreakinc/creator-token-standards、manifoldxyz/creator-core-solidity。
- **博客**：a16z 《NFT Royalties》、OpenSea Blog：Creator Fee Enforcement 时间线；Limit Break《The Future of NFT Royalties》。
- **辩论长文**：Jacob Martin (@jacobcmartin) vs Andrew Wang 等 Twitter 讨论。
- **数据**：Dune `@katgonewild/royalty-leakage`、Messari Royalty 季报。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 版税 | Royalty / Creator Fee | 二级市场分成 |
| Enforce | Enforce Royalty | 强制执行版税 |
| Deny List | Operator Deny List | 禁止的市场 operator 黑名单 |
| PaymentProcessor | PaymentProcessor | Limit Break 的强制版税撮合合约 |
| Transfer Validator | Transfer Validator | 转账授权器合约 |
| Royalty Splitter | Payment Splitter | 多方分账合约 |
| 可选版税 | Optional Royalty | 买家/卖家可选的版税 |
| ARR | Artist Resale Right | 传统艺术转售权 |
| Bps | Basis Points | 万分点（1% = 100 bps）|

---

*Last verified: 2026-04-22*
