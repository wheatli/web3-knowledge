---
title: PlayDapp PLA 无限增发攻击（2024-02-09 / 2024-02-12, ~$290M）
module: 08-security-incidents
category: Key-Mgmt | Protocol-Bug
date: 2024-02-09
loss_usd: 290000000
chain: Ethereum
severity: Tier-1
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/playdapp-rekt/
  - https://www.halborn.com/blog/post/explained-the-playdapp-hack-february-2024
  - https://slowmist.medium.com/slowmist-a-brief-analysis-of-the-playdapp-hack
  - https://twitter.com/peckshield/status/1756090062615748955
  - https://etherscan.io/address/0x3a983ef4d84abf69ea5b1459df8a9c7fa7ed6a73
tx_hashes:
  - 0xebf677d7423d51e742b44b0e88728e1ab3d3ef47c3b68f77ac14c91da84b9303 (Ethereum, 2024-02-09 首次 mint 200M PLA)
  - 0x4e7fbf3d90b2afa5dafcfe50d36f874d2d4ba907a63ac68bd3dd16cde5e9cbe4 (Ethereum, 2024-02-12 二次 mint 1.59B PLA)
  - 0x1dfa4dbd5b5b6f8dabcb6ad9c7b95b10b19e3f48aa87e8c3e9c77cefa5a5e40f (Ethereum, 攻击者向 OKX/Bybit 等出金)
---

# PlayDapp PLA 无限增发攻击

> **TL;DR**：2024-02-09 至 02-12，基于 Ethereum 的游戏平台 PlayDapp（原 PLA，2024 更名为 PDA）被攻击者通过控制 PLA ERC-20 合约的 `MINTER_ROLE`（AccessControl 权限）两次无限增发，共铸造约 17.9 亿枚 PLA（先 200M，后 1,590M），按当时市价总损失约 $290M。攻击者拒绝白帽协商并在中心化交易所开始抛售，PlayDapp 紧急暂停合约、与 Tether/Circle/各大 CEX 冻结，并发起代币迁移至新合约（PDA）。根因是旧 PLA 合约的 Admin 私钥或具有授权权限的账户被攻陷，本质是典型的 Key-Management / Access-Control 失守而非代码 bug。

## 1. 事件背景

- **项目**：PlayDapp 是一家韩国区块链游戏与 NFT 平台，2018 年由 Brian Choi 创立，旗下有《Along with the Gods》《Dragon Steel》等游戏，合作方包括 Samsung、Wemade。PLA 是其平台实用代币，2024-02 前在 Upbit、Bithumb、Binance、Coinbase 等头部 CEX 上线，攻击前 FDV ≈ $170M，流通市值 $65M。
- **合约架构**：PLA 代币合约 `0x3a983ef4d84abf69ea5b1459df8a9c7fa7ed6a73` 使用 OpenZeppelin `AccessControlUpgradeable`，具备 `MINTER_ROLE` 与 `DEFAULT_ADMIN_ROLE`。合约为可升级代理，初始化时将 mint 权限授予 PlayDapp 官方多签 + 业务 EOA 账户。
- **时间轴**：
  - **2024-02-09 13:42 UTC**：攻击者地址 `0xb50721bcf8d664dc...` 调用 `mint()`，一次性铸造 200,000,000 PLA。
  - **2024-02-09 晚间**：PlayDapp 官方承认"未授权 mint"，尝试与攻击者在链上留言谈判，提供 $1M 白帽赏金，攻击者已读不回。
  - **2024-02-12 10:19 UTC**：攻击者再次 `mint()`，增发 1,590,000,000 PLA（约 12 倍总供应量）。
  - **2024-02-13**：PlayDapp 宣布暂停合约、发起链上冻结请求。Tether 冻结部分 USDT 通路。
  - **2024-02-20**：PlayDapp 启动迁移方案，新合约 PDA（PlayDapp Token 2.0）按 1:1 对合法持有者快照兑换。
- **发现者**：PeckShield 率先在 Twitter 告警并追踪两笔 mint；慢雾 MistTrack 协同追踪资金流向 Tornado Cash 与 OKX、Gate、Bybit 等交易所。

## 2. 事件影响

- **直接损失**：两次合计 1,790,000,000 PLA，按 2024-02 PLA 现货价 $0.15–$0.18 估值 $290M（rekt.news 记为 $290M，为历史前十 ERC-20 增发类损失之一）。
- **抛售规模**：链上数据显示攻击者在 02-09 至 02-12 期间通过 Uniswap v3 PLA/WETH 池与 1inch 聚合器共抛售约 250M PLA，套现约 $32M ETH+USDT；后续抛售被 CEX 冻结阻断。
- **受害方**：PLA 持币用户（二级市场价格当日下跌 64%，从 $0.18 跌至 $0.065）、LP（Uniswap v3 PLA/WETH 池 TVL 蒸发 85%）、CEX 现货市场。
- **资金去向**：约 $6.3M 经由 Tornado Cash 混币，其余分散至 Bybit、OKX、Gate、MEXC 等 CEX 钱包，多家交易所响应冻结请求；最终约 $13.4M 被冻结，剩余大部分仍在攻击者地址。
- **连带**：同期同样采用 OZ AccessControl + EOA 管理 mint 权限的 GameFi 代币（如 Pixels、Gala 等）遭到社区审视；PlayDapp 股东 Wemade 股价当日下跌 7%。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类

**Key Management 失守 + Access Control 滥用**。并非代码逻辑 bug，而是特权账户（MINTER_ROLE 持有者）的私钥或社工控制被攻陷，使得合约自身的访问控制机制被"合法"地绕过。

### 3.2 受损合约

- **代币合约**：`0x3a983ef4d84abf69ea5b1459df8a9c7fa7ed6a73`（PLA Proxy, 2020-12-17 部署）
- **实现合约**：`0x85684ca72a1b49f3aa1a20e9c5f7faf6a534e1ce`（Implementation v2）
- **攻击者 mint 调用**：block 19188827（Feb-09-2024）与 19210690（Feb-12-2024）

### 3.3 关键代码片段

PLA 合约的 mint 授权模型（基于 OpenZeppelin `AccessControlUpgradeable`）简化如下：

```solidity
// PlayDapp Token (PLA) - ERC20Upgradeable with AccessControl
// 源: Etherscan verified 0x3a983ef4d84abf69ea5b1459df8a9c7fa7ed6a73

bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

function mint(address to, uint256 amount) public virtual {
    // ❶ 仅校验调用者是否持有 MINTER_ROLE
    require(
        hasRole(MINTER_ROLE, _msgSender()),
        "ERC20PresetMinterPauser: must have minter role to mint"
    );
    _mint(to, amount);
}

function grantRole(bytes32 role, address account)
    public virtual override onlyRole(getRoleAdmin(role))
{
    _grantRole(role, account);  // ❷ DEFAULT_ADMIN_ROLE 可任意扩展 MINTER_ROLE
}
```

**错在哪**：
- ❶ `mint()` 没有上限（cap）、没有时间锁、没有多签确认；一旦 `MINTER_ROLE` 泄露，攻击者可一次性铸出远超现有供应的代币。
- ❷ `DEFAULT_ADMIN_ROLE` 持有者可以无限制地 `grantRole(MINTER_ROLE, 任意地址)`，且该操作无 timelock。链上数据表明 2024-02-09 攻击前约 15 分钟，有一笔来自合法 admin 地址的 `grantRole` 交易将 MINTER_ROLE 授予了攻击者地址——这要么是管理员 EOA 私钥被盗，要么是该 EOA 本身就是攻击者/合谋者。

### 3.4 攻击步骤分解

1. **获取权限（tx 未公开具体入侵路径）**：攻击者通过私钥窃取 / 社工 / 内鬼，控制了持有 DEFAULT_ADMIN_ROLE 的账户（或直接获得已持有 MINTER_ROLE 的账户）。
2. **（可选）自授权**：如果入侵账户只有 admin 权限，先调用 `grantRole(MINTER_ROLE, attackerEOA)` 授予自身 mint 权限。
3. **第一笔 mint**（block 19188827）：`mint(0xb50721..., 200_000_000 * 1e18)` — 铸造 2 亿 PLA。
4. **市场试探 + 分散**：通过 Uniswap v3 和 CEX 存款小额试探流动性与风控响应，约 02-09 至 02-11 抛售 1.5 亿 PLA。
5. **第二笔 mint**（block 19210690）：`mint(attacker, 1_590_000_000 * 1e18)` — 铸造 15.9 亿 PLA。
6. **混币 + 跨 CEX 出金**：$6.3M 经 Tornado Cash，剩余通过 5+ 个中转地址存入 OKX、Gate、MEXC；PlayDapp 发起冻结请求。

### 3.5 为何审计未发现

PLA 合约代码本身是 OZ 标准模板，审计机构（Theori 2021）并不会将"管理员私钥被盗"列为代码层问题。**这类事件的根因永远在合约之外**：

- 缺少 **Mint cap**（硬上限），标准 ERC-20 审计建议但非必须。
- 缺少 **Timelock**（24–72h），这本是 DeFi 治理合约标配，却在 GameFi 代币被普遍忽略。
- Admin/Minter 私钥由单签 EOA 或弱多签（2/3 热钱包）持有，没有使用 Gnosis Safe + 硬件钱包 + 离线签名的"冷-温-热"分级架构。

## 4. 事后响应

- **项目方**：
  - 02-13 暂停 PLA 合约（`pause()`）。
  - 02-20 部署全新 PDA 合约 `0x0d7cca66b9e44f525b7d6fe5e8ae0b8c87d0b8b2`，引入：mint cap、MINTER_ROLE timelock、多签(Gnosis Safe 4/7)。
  - 按 02-09 00:00 UTC 快照兑换合法持有者，攻击者地址与已抛售部分不予兑换。
- **资产追回**：借助 Tether、Circle、Binance、OKX、Bybit 协作冻结约 $13.4M；剩余因混币未追回。
- **执法**：韩国 KNPA Cyber Bureau 介入调查；未公开是否确认为 Lazarus（慢雾与 rekt.news 均表示**归因未公开**，不同于 DMM Bitcoin 与 WazirX 有 FBI 正式声明）。
- **事后审计**：Theori + Hexens 于 2024-03 对 PDA v2 合约复审，报告公开。
- **行业连锁**：GameFi 项目方批量清理历史 MINTER_ROLE 授权；Immutable、Pixels 于 3 月主动披露迁移/加固。

## 5. 启发与教训

- **开发者**：
  - ERC-20 铸造权限必须配合 mint cap + Timelock + Multisig 三件套；任何无上限 mint 函数视为高危。
  - 避免 EOA 直接持有 MINTER_ROLE；应通过 Gnosis Safe 并接入硬件钱包。
  - `DEFAULT_ADMIN_ROLE` 的变更本身要触发告警（OpenZeppelin Defender / Forta）。
- **审计方**：将"权限私钥托管方式"纳入审计范围（而不仅仅是代码），要求项目方提供 key ceremony 文档。
- **用户**：观察代币合约是否有 mint cap 和 timelock 是评估 GameFi/L2 代币风险的 baseline；Etherscan "Write Contract" 页能直接看出 `mint()` 是否可被单地址调用。
- **协议**：事件响应必须有预案——包括多 CEX 冻结联系人列表、Tether/Circle 紧急邮件模板、快照+迁移脚本。

## 6. 参考资料

- rekt.news PlayDapp Rekt
- Halborn: Explained: The PlayDapp Hack (February 2024)
- SlowMist / MistTrack 追踪线索（Twitter @SlowMist_Team, 2024-02-10 / 2024-02-13）
- PeckShield Alert (@peckshield, 2024-02-09)
- PlayDapp 官方公告：<https://medium.com/playdapp-kr>（PDA 迁移说明, 2024-02-20）
- Etherscan PLA 合约：`0x3a983ef4d84abf69ea5b1459df8a9c7fa7ed6a73`

---

*Last verified: 2026-04-23*
