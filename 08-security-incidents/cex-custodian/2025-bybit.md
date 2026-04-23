---
title: Bybit 冷钱包 $1.46B 超级被盗事件（2025-02-21, ~$1.46B）
module: 08-security-incidents
category: CEX-hack | Wallet | Supply-Chain | Social-Eng | Key-Mgmt
date: 2025-02-21
loss_usd: 1460000000
chain: Ethereum
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.bybit.com/en/help-center/article/Incident-Update-Unauthorized-Activity-Involving-ETH-Cold-Wallet
  - https://slowmist.medium.com/slowmist-analysis-of-the-root-cause-of-bybit-hack-b8b8b6a4e0c3
  - https://www.ic3.gov/PSA/2025/PSA250226
  - https://rekt.news/bybit-rekt/
  - https://www.sygnia.co/blog/bybit-incident-investigation-update
  - https://blog.chainalysis.com/reports/bybit-hack-february-2025/
tx_hashes:
  - 0x46deef0f52e3a983b67abf4714448a41dd7ffd6d32d32da69d62081c68ad7882 (Ethereum, 恶意 Safe exec transaction，授权转出 ETH 冷钱包)
  - 攻击前恶意实现合约部署 tx：SlowMist/Sygnia 报告已披露，具体 hash 以官方报告为准
---

# Bybit 冷钱包 $1.46B 超级被盗事件

> **TL;DR**：2025-02-21，全球第二大中心化交易所 Bybit 的以太坊冷钱包被盗取约 401,347 ETH + mETH/stETH/cmETH 等共计约 14.6 亿美元，成为加密史上最大单笔被盗事件。攻击者并未直接窃取私钥，而是通过入侵 Safe {Wallet} 多签前端基础设施（开发者机器被植入恶意 JavaScript），向签名者的浏览器呈现伪造的交易 UI——界面显示"普通的资金归集"，但实际由签名者硬件钱包盲签的数据是一笔 `delegatecall` 到恶意合约，该合约直接改写 Safe 的 `masterCopy`（实现合约槽），从而控制整个多签钱包并执行转出。2025-02-26 美国 FBI 发布 PSA 正式将事件归因朝鲜 Lazarus Group "TraderTraitor" 子集群。

## 1. 事件背景

### 项目 / 主体

- **Bybit**：2018 年成立，总部原迪拜；按现货+衍生品总交易量长期位列全球前三大 CEX。事发时日均交易量约 $36B，托管资产估算 $20B+ 级别。
- **被攻击钱包**：Bybit 以太坊 ETH 多签冷钱包（Safe 合约，3/X 门限，具体 threshold 未完全披露），用于大额归集与对热钱包补仓。
- **签名基础设施**：Bybit CEO Ben Zhou 公开说明，Bybit 使用 Safe {Wallet}（原 Gnosis Safe）作为多签合约层，签名者使用 Ledger 硬件钱包线下物理确认。

### 时间轴

- **2025-02-21 14:13:35 UTC**（约）：Bybit 发起"从冷钱包向热钱包补仓"的常规多签交易。
- **14:16 UTC**：签名完成后交易上链，但交易内容并非预期的 ERC-20 转账，而是改写 Safe 实现合约。
- **几分钟内**：攻击者调用新实现合约，连续转出 401,347 ETH、8,000 mETH、90,376 stETH、15,000 cmETH 至约 50+ 中转地址。
- **14:44 UTC**：Bybit CEO Ben Zhou 在 Twitter / Telegram 直播确认冷钱包被盗。
- **T+1 h**：Bybit 向市场公开宣布：偿付能力充足，用户资金 1:1 持有；紧急从 Binance、Bitget、HTX、Mirana 等对手方拆入约 10 万 ETH 头寸兑付提现。
- **2025-02-22**：Safe 团队暂停前端 UI；SlowMist、Sygnia、Verichains、Mandiant 四支外部安全团队平行调查。
- **2025-02-26**：FBI 发布 PSA I-022625 正式归因朝鲜 Lazarus Group "TraderTraitor" 行动。
- **2025-03 起**：Chainalysis/Elliptic 链上协同冻结；最终约 $40–60M（占比 3%~4%）被交易所/桥冻结，大部分资金已通过 THORChain、eXch、Cross-chain bridge 洗入 BTC。

### 发现过程

- 签名者设备上 Safe UI 显示的交易与实际 Ledger 盲签 payload 不一致——签名完成后即刻发现链上 `masterCopy` 被改写。
- ZachXBT 与 Arkham 几乎实时公开追踪资金流向。

## 2. 事件影响

### 直接损失（按 2025-02-21 价格）

| 资产 | 数量 | 近似 USD |
|---|---|---|
| ETH | ~401,347 | ~$1.12B |
| stETH | ~90,376 | ~$253M |
| mETH | ~8,000 | ~$23M |
| cmETH | ~15,000 | ~$43M |
| 合计 | — | ~$1.46B |

数据综合自 Arkham、SlowMist、Elliptic 公开看板；单项数量以官方公告为准。

### 受害方

- Bybit 公司自身承担——Ben Zhou 声明"用户资金不受影响"，Bybit 以企业自有储备 + 对手方拆借 + 短期信用贷款填平缺口。
- 行业信任：事件加剧了 CEX 依赖第三方多签前端的风险暴露讨论。

### 连带影响

- ETH 现货 24h 下跌约 4%，mETH 脱锚短暂跌至 0.88 倍。
- Safe {Wallet} 被迫升级：强制交易原始数据校验、离线签名流程改版、WalletConnect 层移除。
- 多个 CEX（OKX、Binance、Kraken）随后审查其冷钱包签名 UI 是否同样依赖 Safe 官方 app.safe.global 前端。
- 推动 Ledger 等硬件钱包厂商加速推广 "Clear Signing"（EIP-712/ERC-7730 结构化展示）。

### 资金去向

- 初期资金被拆分至 ~50 地址，再通过 THORChain 大规模转 BTC（单日 THORChain 交易量破历史纪录）。
- eXch 无 KYC 交易所拒绝冻结，成为主要出口之一，Elliptic 报告超 $75M 经该平台。
- Chainalysis 评估约 3–4% 被全行业冻结；大部分资金已进入 Lazarus "mixing-then-OTC" 管道，预计 6–12 个月消化。

## 3. 技术根因（代码级分析，本章为核心）

### 3.1 漏洞分类

**Supply-chain（供应链）+ Social-Eng（定向社工）+ Wallet-UI 欺骗（盲签）**——并非智能合约漏洞、也非私钥泄露，而是"多签签名前端被篡改 + 硬件钱包盲签 + delegatecall 改写实现合约"的组合。

### 3.2 根因链条（Sygnia / SlowMist / Mandiant 共识）

1. **开发者机器被入侵**：Safe {Wallet} 某 AWS 开发账号下的一台开发者工作站于 2025-02-17 左右被 Lazarus 植入恶意代码（常见向量：LinkedIn 招聘诱骗 → Zoom 伪客户端 → npm 依赖后门，具体入口以 Mandiant 报告为准）。
2. **前端静态资源替换**：攻击者将恶意 JavaScript 植入 Safe 官方 `app.safe.global` 的静态资源（S3 / CloudFront 缓存层），仅对特定目标地址（Bybit 冷钱包）触发，避免大面积暴露。
3. **UI 欺骗**：签名者打开 Safe UI 看到"转 ETH 到热钱包"的常规界面，但前端实际构造的 `SafeTx` payload 为 `operation=1 (delegatecall)` + `to=<恶意合约>` + `data=<改写 masterCopy 的 calldata>`。
4. **硬件钱包盲签**：Ledger 在未启用 Clear Signing 的情况下，展示给签名者的是 EIP-712 哈希（一串不可读十六进制）。三名签名者依据 UI 显示的"人类可读"部分逐一盲签通过。
5. **执行阶段**：Safe `execTransaction` 使用 `delegatecall` 进入攻击合约，在 Safe 自身存储空间内改写 slot 0（`masterCopy`），指向攻击者部署的"后门实现"。
6. **清空余额**：攻击者直接调用新实现合约的自定义函数 `sweep()`，将所有 ETH/ERC-20 转出。由于 `masterCopy` 已变更，原来的多签校验逻辑被跳过。

### 3.3 关键技术点

**Safe 合约架构**（简化，原仓库：`safe-global/safe-smart-account`）：

```solidity
// Safe 以 Proxy 形式部署，用户资金存于 Proxy
// slot 0 存储 masterCopy（实现合约地址）
// execTransaction 支持 operation=0(CALL) 或 operation=1(DELEGATECALL)
contract SafeProxy {
    address internal singleton;  // slot 0 == masterCopy
    fallback() external payable {
        // delegatecall 到 singleton
    }
}

// execTransaction 的 operation=1 时，对攻击者 to 合约发起 delegatecall
// 此时攻击合约 "在 Safe 的存储上下文运行"，可以任意改写 slot 0
```

**攻击合约逻辑**（根据 SlowMist/Verichains 事后反汇编披露，伪代码）：

```solidity
// 攻击合约（被 Safe 以 delegatecall 调入）
contract Evil {
    function doIt() external {
        // 直接写 slot 0：等效于 singleton = newImpl
        assembly {
            sstore(0, 0x<攻击者新实现合约地址>)
        }
    }
}
```

一旦 `singleton` 被改写，后续所有对 Safe Proxy 的调用都路由到攻击者实现合约，owners / threshold 等校验被绕过。

### 3.4 攻击步骤分解

1. **T-4 天**：Lazarus 入侵 Safe 开发者 AWS 凭证，修改前端 JS（tx 未公开）。
2. **T 时刻 14:13 UTC**：Bybit 三位签名者各自在官方 UI 发起/签名"归集"交易，硬件钱包展示哈希。
3. **14:16 UTC**：`execTransaction` 上链，攻击者 `Evil.doIt()` 被 delegatecall，Safe 的 singleton 被替换。（tx: `0x46deef0f52e3a983b67abf4714448a41dd7ffd6d32d32da69d62081c68ad7882`）
4. **14:16–14:30 UTC**：攻击者调用被替换实现合约的提取函数，ETH/mETH/stETH/cmETH 被分批转至 51 个一级地址。
5. **T+1 h 起**：资金经 THORChain、Chainflip、eXch 开始跨链混淆。

### 3.5 为何审计 / 测试未发现

- **非合约漏洞**：Safe 合约经历多年审计与形式化验证，被信任程度极高；漏洞发生在"开发者机器 → 前端构建产物"的信任链条。
- **Clear Signing 缺失**：Ledger 默认硬件钱包对 Safe `execTransaction` 的展示是 EIP-712 哈希，不展开操作类型（call/delegatecall）、目标地址、data calldata；签名者无从辨别。
- **流程盲点**：行业长期把"硬件钱包线下签名 + 多签"视为双重防御，但当 UI 被篡改、签名者无法解析原始字节时，实质上只剩"多人盲签"。

## 4. 事后响应

### Bybit

- Ben Zhou 直播 1 小时答记者问；公开储备证明 & 拆入资金来源（Binance、Bitget、HTX、Mirana、Galaxy、DWF、MEXC、Antalpha 等）。
- 发起 Lazarus Bounty 网站（lazarusbounty.com），对协助冻结资金的白帽/交易所支付 10% 赏金。
- 2 周内完成补仓，用户提现始终保持开放。

### Safe 团队

- 紧急暂停 app.safe.global 前端；迁移至新 CI/CD + 多重签名构建产物公证。
- 上线强制 "交易原始 payload 展示" 与 "第二设备二次校验" 功能。
- 与 Ledger、Trezor 合作推进 Safe 交易 Clear Signing 标准。

### 执法与归因

- **2025-02-26 FBI PSA I-022625**：明确将事件归因 DPRK "TraderTraitor"（Lazarus 子集群），公开 51 个攻击者地址并呼吁 VASP 协同封堵。
- Chainalysis、Elliptic、TRM Labs 联合发布归因与链上追踪报告。

### 第三方独立调查

- Sygnia（Bybit 雇用）报告证实 Bybit 自身基础设施未被入侵，根因在 Safe 前端基础设施。
- Verichains / SlowMist 对攻击合约字节码做了完整逆向，还原 delegatecall → sstore 攻击路径。

### 行业连锁

- 多家交易所（OKX、Binance、Kraken）审查并放弃使用公开 Safe UI 做冷钱包大额签名，转向自建前端 + 专有设备。
- 硬件钱包厂商加速推广 ERC-7730 / Clear Signing 元数据。

## 5. 启发与教训

### 对开发者 / 协议方

- **多签前端必须自建或版本锁定**：公共 Safe UI 虽权威，但前端篡改风险属于单点失败；重资产场景应部署私有前端或使用完全静态的开源构建。
- **离线交易构造 + 原始 calldata diff**：大额签名前，签名者用独立机器重新构造 `SafeTx` 并逐字节对比。
- **引入 delegatecall 白名单**：Safe 可配置 Guard 合约拒绝任何 `operation=1` 或限制其 `to` 地址。

### 对审计 / 红队

- 从"合约审计"扩展到"签名工作流审计"：把从开发者键盘到签名链上的每一跳都纳入威胁建模。
- 供应链：npm / Docker / CDN 构建产物的可重放性（SLSA、sigstore）。

### 对用户 / 签名者

- 硬件钱包必须启用 Clear Signing。任何情况下对无法解析的 EIP-712 哈希要求二次核对。
- 警惕社工入口：LinkedIn 远程面试、虚假 Zoom/Meet 客户端、npm 包是 Lazarus 2023-2025 年主要社工向量。

### 对交易所治理

- CEX 大额冷钱包签名应设 timelock（≥ 30 min），给监控团队介入时间。
- 实时链上监控（Forta、Hexagate、Hypernative）对 `masterCopy` 写入事件设置 P0 告警。
- 偿付能力证明（PoR）常态化，非危机时期即须发布 Merkle 证明。

## 6. 参考资料

- **SlowMist**：The Root Cause of Bybit Hack（2025-02 系列推文 + Medium 深度稿）
- **Sygnia**：Bybit Incident Investigation Update（Bybit 雇用的一线取证方）
- **FBI IC3**：PSA I-022625 "North Korea Responsible for $1.5 Billion Bybit Hack"
- **Chainalysis**：Bybit Hack February 2025 资金流向报告
- **Elliptic / TRM Labs**：Bybit hack laundering 追踪系列
- **rekt.news**：Bybit Rekt
- **Verichains**：Bybit Exploit Technical Analysis（恶意合约反汇编）
- **Ben Zhou (CEO)**：2025-02-21 紧急直播 + Post-mortem 官方网页

---

**本条目源于 SlowMist / Sygnia / FBI / Verichains 2025-02~03 公开分析。部分细节（完整恶意实现合约字节码、Safe 开发者入侵时间线）后续可能随 Mandiant / Sygnia 最终报告更新，请重审。**

未公开 / 待核实字段：
- Safe 开发者机器初始入侵具体 tx / payload
- 攻击者部署"恶意实现合约"的完整地址（多个源给出不同占位）
- Safe 多签的具体 threshold（Bybit 未完整披露）

*Last verified: 2026-04-23*
