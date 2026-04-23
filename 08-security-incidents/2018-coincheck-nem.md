---
title: Coincheck NEM(XEM) 热钱包被盗 $534M（2018-01-26）
module: 08-security-incidents
category: CEX-hack
date: 2018-01-26
loss_usd: 534000000
chain: [NEM]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://corporate.coincheck.com/2018/01/27/46.html
  - https://www.nikkei.com/article/DGXMZO26148290W8A120C1000000/
  - https://blog.nem.io/nem-foundation-official-statement-regarding-coincheck-hack
  - https://www.reuters.com/article/us-japan-cryptocurrency-coincheck-idUSKBN1FF29G
  - https://www.chainalysis.com/blog/lazarus-group-funds-exchanges/
tx_hashes:
  - 攻击者接收地址 NC4C6PSUW5CLTDT5SXAGJDQJGZNESKFK5MCN77OG（NEM 链）
  - 共 19 笔批量转账，从 Coincheck 热钱包地址 NCQ62Y…（热钱包地址，链上可查）
---

# Coincheck XEM 热钱包被盗 $534M

> **TL;DR**：2018 年 1 月 26 日，日本交易所 Coincheck 的 NEM（XEM）热钱包被盗 523,000,000 XEM，按当时价格约合 580 亿日元 / 5.34 亿美元，是当时加密行业有记录以来最大单一 CEX 被盗事件（超过 2014 年 MtGox 当时价）。根因为：Coincheck 将所有 XEM 资产放在单一热钱包，无多签、无冷钱包隔离、甚至未启用 NEM 协议原生支持的多签账户功能；叠加内部网络被植入恶意软件后私钥泄露。2019 年 Chainalysis 公开指认 Lazarus Group 是主要洗钱通道，朝鲜归因被 FBI 与 Chainalysis 交叉确认。

## 1. 事件背景

- **主体**：Coincheck Inc.（成立 2014，总部东京）为日本主流加密交易所之一，事件发生时 NEM 交易量排名亚洲前三。
- **时间轴**：
  - 2018-01-26 02:57 UTC（日本时间 11:57）：攻击者从 Coincheck NEM 热钱包批量提出 523M XEM，共 19 笔 tx，约 20 分钟完成。
  - 2018-01-26 11:25 UTC 起：Coincheck 监控系统发现异常，暂停 NEM 出入金；随后暂停 BTC、ETH 等大部分币种交易。
  - 2018-01-26 22:00 UTC：Coincheck 召开紧急新闻发布会，CEO 和田晃一良公开鞠躬道歉。
  - 2018-01-27：NEM 基金会宣布对受 tainted 的地址启用 "自动标记"（mosaic tag），链上任何交易所收到后可拒绝。
  - 2018-03-13：Coincheck 宣布用自有资金按 88.549 日元/XEM 赔付全部 26 万受害用户。
  - 2018-04：日本 Monex Group 收购 Coincheck（~36 亿日元）。
  - 2018-03-21：NEM 基金会宣布关闭链上追踪（理由：无人愿意出金，追踪已完成使命），部分 XEM 被认为已通过韩国 Yobit、加拿大 ShapeShift 等渠道出金。
  - 2019-01：日本警方宣布部分嫌疑人被识别。
  - 2020-03-11：日本警方宣布逮捕 2 名日本人，承认以 XEM 换 BTC 出金。
  - 2019-03 / 2022（Chainalysis）：将 Coincheck 事件归因于 Lazarus Group。
- **发现过程**：Coincheck 自动告警；NEM 社区同一分钟内也在链上捕获异常大额转账。

## 2. 事件影响

- **直接损失**：523,000,000 XEM，按 2018-01-26 ~$1.02 估约 $534M；按日元约 580 亿日元。
- **受害方**：约 26 万 Coincheck 用户持有的 NEM 余额。
- **连带影响**：
  - NEM 代币价格单日下跌 15%+，短期流动性受重创。
  - 日本金融厅（FSA）对全部登记交易所展开业务改善命令（业務改善命令），Coincheck 被重点整改。
  - 推动日本 FSA 2018 年下半年发布"交易所安全指针"，强制要求冷钱包比例 ≥95%、多签签名等。
- **资金去向**：
  - 约 40% 通过暗网、场外和韩国 Yobit 等渠道出金。
  - 2019-03 Chainalysis 报告指出大量资金经 Lazarus 常用混币链路清洗。
  - 约 60% 在 NEM 基金会的 tag 机制下无法在主流 CEX 出金，长期静置。

## 3. 技术根因

- **漏洞分类**：Key-Mgmt + Architecture（冷热分离缺失）+ Protocol-Multisig-Unused + Endpoint-Malware。
- **关键事实**：
  1. **XEM 全部存储在单一热钱包**：与 BTC、ETH 使用多签+离线方案不同，Coincheck 对 XEM 缺少冷钱包方案，"因为 XEM 开发资源较紧张"（官方听证会陈述）。
  2. **未使用 NEM 原生多签**：NEM 协议自 Catapult 升级前即原生支持 n-of-m multisig 账户（NEM Multisig Contract），Coincheck 未启用。
  3. **私钥以明文存储**：热钱包签名机上 wallet 导入状态下运行，私钥可由 root 账户读取。
  4. **内部网络被投放恶意邮件附件**：事后日本警方调查指出，攻击者在 2017 年末通过邮件植入远控木马；员工点击后失守，攻击者数周横向移动至 XEM 签名机。
- **受损模块**：Coincheck 内部钱包服务（闭源）。
- **伪代码还原（NEM 节点签名流程的典型脆弱点）**：

```javascript
// nem-sdk 典型热钱包签名流程（简化）
const nem = require('nem-sdk').default;

function sendXEM(privateKey, toAddr, amountMicroXem) {
    const common = nem.model.objects.create("common")("", privateKey);
    // 缺陷：privateKey 从本地明文文件读出，无 HSM / remote signer
    const transferTx = nem.model.objects.create("transferTransaction")(
        toAddr, amountMicroXem / 1e6, "stolen"
    );
    const entity = nem.model.transactions.prepare("transferTransaction")(
        common, transferTx, nem.model.network.data.mainnet.id
    );
    return nem.model.transactions.send(common, entity, endpoint);
}
```

- **攻击步骤**：
  1. 2017 末：钓鱼邮件投递至 Coincheck 员工邮箱，植入远控。
  2. 数周横向移动：获取 XEM 签名机 SSH 凭证。
  3. 2018-01-26 上午：登录签名机，读取热钱包私钥。
  4. 使用自制脚本在 20 分钟内构造 19 笔批量 transferTransaction，签名广播。
  5. 523M XEM 到达攻击者地址 `NC4C6P…77OG`。
- **tx hash**：NEM 链上可查的攻击者地址及 19 笔入金 tx；完整列表见 NEM 浏览器 explorer.nemtool.com。
- **为何审计未发现**：日本 FSA 当时对交易所安全审计标准粗糙；Coincheck 在事件时仍处于"注册审查中"状态（尚未正式注册 FSA）。

## 4. 事后响应

- **项目方**：
  - 2018-01-28 宣布用自有资金按 88.549 JPY/XEM 以日元方式赔付全部 26 万用户，金额约 460 亿日元。
  - 2018-03-12 开始分批赔付完成。
  - 2018-04 被 Monex Group 收购，管理层大换血。
- **NEM 基金会**：
  - 启动链上 "mosaic tag" 追踪，对 tainted XEM 打上标签，呼吁 CEX 拒收。
  - 2018-03-21 宣布结束主动追踪，称"主要追踪目标已达到"。
- **执法**：
  - 2018–2020 日本警察厅通过地址监控识别 2 名日本公民为 "二次接收者"，2020-03 逮捕；罪名为组织犯罪处罚法下的"洗钱"。
  - Chainalysis 报告《Lazarus Group》归因为朝鲜 Lazarus。
- **行业连锁**：
  - 日本 FSA 对所有交易所发出业务改善命令；"冷钱包比例 ≥95%"成为法规要求。
  - 全球 CEX 对单链资产独立多签方案的重视度大幅提升。

## 5. 启发与教训

- **对开发者 / CEX**：
  - 每条链都要单独设计冷热钱包方案，不能因开发资源紧张而"本链只有热钱包"。
  - 对协议原生支持的安全特性（multisig、timelock）要主动启用，不应依赖自研风控替代。
  - 私钥绝不能以明文落盘；至少用 AWS KMS / YubiHSM / MPC。
- **对审计方**：合规审计必须逐链、逐资产核对冷热比例、多签启用情况。
- **对用户**：交易所存款应分散；热钱包公开金额是重要风险指标。
- **对监管**：日本 FSA 的 2018 年整改是全球最早的"交易所安全强监管"样板，效果显著：此后日本本土无同规模事件。

## 6. 参考资料

- Coincheck 官方公告：`corporate.coincheck.com/2018/01/27/46.html`
- 日本经济新闻：《コインチェック、580 億円不正流出》2018-01-26
- NEM 基金会声明：《Official Statement Regarding Coincheck Hack》2018-01-27
- Reuters：《Japan's Coincheck cryptocurrency exchange hacked》2018-01-26
- Chainalysis 报告：《Lazarus Group Funds Have Moved to Exchanges》2019-03 及《Crypto Crime Report 2022》
- 日本金融厅 2018-03 公开的《業務改善命令》文本
- SlowMist 年度报告：Coincheck 章节
- rekt.news：Coincheck 回顾

---

*Last verified: 2026-04-23*
