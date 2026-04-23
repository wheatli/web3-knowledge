---
title: CoinDash ICO 官网收款地址被篡改 $7M 被盗（2017-07-17, ~$7M 按当时价）
module: 08-security-incidents
category: Social-Eng
date: 2017-07-17
loss_usd: 7000000
chain: [Ethereum]
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.coindesk.com/markets/2017/07/17/coindash-ico-hacked-crypto-investors-lose-millions
  - https://medium.com/@CoinDashDev/coindash-token-sale-hack-a7a4cd04ef04
  - https://www.ccn.com/coindash-ico-hack
  - https://cointelegraph.com/news/7-million-lost-in-coindash-ico-hack
tx_hashes:
  - 攻击者收款地址：0x6A164122D5Cf7C840D26e829B46dCc4ED6C0ae48
  - CoinDash 合法地址（被替换前）：0x725A2a32b6b6ecf4Dd0b51ed066b34727c0a2C79
---

# CoinDash ICO 官网前端被劫持，$7M 误转攻击者

> **TL;DR**：2017 年 7 月 17 日，以色列 DeFi/社交交易项目 CoinDash 进行 ICO 开售的 3 分钟内，其官网 `coindash.io` 上展示的 ETH 收款地址被攻击者篡改为 `0x6A16…ae48`，投资者按官网指引转账，合计约 43,000 ETH（约 $7M）落入攻击者地址。这是 Web3 第一起广受讨论的"前端 / DNS / 服务器劫持"ICO 社工事件，暴露了"合约再安全，前端被改一行字同样归零"的系统性风险。

## 1. 事件背景

- **主体**：CoinDash（后更名 Blox）由以色列团队创立，定位为社交交易平台，ICO 目标 $12M。
- **时间轴**：
  - 2017-07-17 12:00 UTC（规划开售时间）：ICO 按计划开启，官网展示 ERC-20 众筹地址。
  - 2017-07-17 12:00-12:03 UTC：攻击者在 ICO 开售瞬间，将官网上的 ETH 收款地址替换为自己的钱包 `0x6A16…ae48`。
  - 2017-07-17 12:03 UTC：CoinDash 发现异常，在官方 Twitter 与网站弹窗紧急叫停，并关闭 ICO。
  - 投资者累计向攻击者地址转入约 43,438 ETH（~$7M）。
- **发现过程**：社区在 Etherscan 上发现大额 ETH 流向非官方地址，Twitter 反馈在几分钟内汇聚。

## 2. 事件影响

- **直接损失**：约 43,438 ETH（~$7M 按 2017-07 ~$160 估）。
- **受害方**：ICO 早期投资者（大约 2,100 个地址）。
- **连带影响**：
  - 成为"前端安全"话题的标志性事件，推动后来的合约地址硬编码、ENS、PGP 签名公告等多种防护方案讨论。
  - 激化了对 ICO 模式"基础设施成熟度不足"的质疑。
- **资金去向**：
  - 2017-09：攻击者主动返还约 10,000 ETH 到 CoinDash 合法地址，留言声称"部分返还"。
  - 2018-02：再次返还约 20,000 ETH。
  - 2018-11：第三次返还约 6,578 ETH。
  - 累计返还约 36,578 ETH；剩余未返还。
- **归因**：归因未公开（无 FBI / Chainalysis 正式归因报告）。攻击者返还资金行为本身罕见。

## 3. 技术根因

- **漏洞分类**：Social-Eng / Front-end hijack；具体子类为**网页服务器或 CMS 入侵**（CoinDash 未明确披露是 DNS 层还是 Web 服务器层被入侵，事后报告更倾向于 Web 服务器被入侵改写静态页面）。
- **根因链**：
  1. CoinDash 网站托管在共享或第三方 CMS 环境（具体堆栈未公开），上线前安全加固不足。
  2. 攻击者在 ICO 开售前已取得对网站后端的写入权（可能通过未打补丁的 CMS 插件、管理员弱密码或 SSH 凭证泄漏）。
  3. ICO 开售瞬间，攻击者修改 HTML 中的收款地址字段，或替换 JS 中硬编码常量。
  4. 投资者看到的"官方地址"即攻击者地址，协议合约未被攻击（根本无合约被调用）。
- **没有涉及智能合约漏洞**：CoinDash 的 ERC-20 众筹合约本身代码未被触碰，问题 100% 在 Web2 层。
- **关键"代码"（Web2 层示意）**：

```html
<!-- coindash.io/ico 页面上的关键片段（概念还原，非原始源码） -->
<div class="eth-deposit">
  送 ETH 到：
  <span class="addr">
    <!-- 攻击者将此处原本的 0x725A… 替换为 0x6A16… -->
    0x6A164122D5Cf7C840D26e829B46dCc4ED6C0ae48
  </span>
</div>
```

- **攻击步骤**：
  1. 预先入侵 Web 服务器（时间点未公开，最晚在 ICO 开售前数小时）。
  2. 监控 ICO 开售流量；待第一批投资者开始转账后改写地址。
  3. 接收约 3 分钟的错误转账，随即 CoinDash 下线官网结束。
- **tx hash**：主要 tx 数千笔，汇入攻击者地址 `0x6A164122D5Cf7C840D26e829B46dCc4ED6C0ae48`（Etherscan 可查）。
- **为何未被发现**：ICO 时代的安全预算几乎完全投入合约审计，Web2 端渗透测试普遍缺失。

## 4. 事后响应

- **项目方**：
  - CoinDash 承诺按原计划向每一位受害投资者等额发放 CDT token，损失由项目方承担。
  - 更名为 Blox 后继续运营产品。
- **资产追回**：攻击者分 3 次主动返还 ~36,578 ETH，剩余 ~7,000 ETH 未追回。
- **执法**：未公开立案结果。
- **行业连锁**：
  - ICO 项目开始推行 "PGP 签名公告" 发布合约地址。
  - ENS 域名（Ethereum Name Service）被更多项目用作对外公告渠道。
  - 后来 Gnosis、Aragon 等大项目的 ICO 进一步把地址写进链上多重公告。

## 5. 启发与教训

- **对开发者**：
  - Web2 前端是"信任边界"的真正起点；ICO / 铸币页面必须：
    - Web 服务器启用 WAF、最小权限、SSH MFA、CMS 全面禁用。
    - 关键资产地址写进官方 Twitter、ENS、GitHub release 等多渠道冗余公告。
    - 页面使用 Subresource Integrity（SRI）与 CSP，保护静态资源与第三方脚本。
- **对审计方**：ICO 审计不能只看合约，必须覆盖 Web2 基础设施。
- **对用户**：
  - 永远交叉校验收款地址；通过项目方 Twitter 官号 + Telegram 同时确认。
  - 浏览器插件（MetaMask）的 ENS 解析或"已知地址白名单"是额外防线。
- **对协议 / 生态**：事件催生了 Token Sale Factory（例如 OpenZeppelin Gnosis Auction）这类去中心化众筹方案，把前端替换风险降到最低。

## 6. 参考资料

- CoinDesk：《CoinDash ICO Hacked: Crypto Investors Lose Millions》2017-07-17
- CoinDash 官方 Medium：《CoinDash Token Sale Hack》2017-07-17
- Cointelegraph：《$7 Million Lost in CoinDash ICO Hack》
- CCN：《CoinDash ICO Hack》
- 攻击者地址 Etherscan：`0x6A164122D5Cf7C840D26e829B46dCc4ED6C0ae48`
- SlowMist 2017 年度报告：ICO 前端安全章节
- rekt.news 回顾（简版）

---

*Last verified: 2026-04-23*
