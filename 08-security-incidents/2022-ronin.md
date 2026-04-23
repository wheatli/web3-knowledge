---
title: Ronin Bridge 跨链桥被盗事件（2022-03-23, ~$625M）
module: 08-security-incidents
category: Bridge
date: 2022-03-23
loss_usd: 625000000
chain: [Ronin, Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://roninblockchain.substack.com/p/community-alert-ronin-validators
  - https://www.chainalysis.com/blog/lazarus-group-axie-infinity-ronin-bridge-dprk/
  - https://www.fbi.gov/news/press-releases/fbi-statement-on-attribution-of-malicious-cyber-activity-posed-by-the-democratic-peoples-republic-of-korea
  - https://rekt.news/ronin-rekt/
  - https://thehackernews.com/2022/07/axie-infinity-hack-how-fake-job-offer.html
tx_hashes:
  - 0xc28fad5e8d5e0ce6a2eaf67b6687be5d58113e16be590824d6cfa1a94467d0b7（Ethereum 取出 173,600 ETH + USDC 汇总引用）
  - Ronin 侧 validator 签名交易（官方 post-mortem 引用，具体 hash 部分未公开）
---

# Ronin Bridge 跨链桥被盗事件

> **TL;DR**：2022-03-23，连接 Axie Infinity 游戏链 Ronin 与 Ethereum 的官方桥被盗 173,600 ETH 与 25.5M USDC，按当日价约合 6.25 亿美元，是彼时史上最大 DeFi/桥攻击。根因不是智能合约漏洞，而是桥的 5/9 PoA 多签中 4 个由 Sky Mavis 员工持有、1 个由 Axie DAO 托管给 Sky Mavis "临时签名"的权限未被回收；攻击者通过 LinkedIn 假面试社工窃得 Sky Mavis 4 把 validator 私钥后，再借"Axie DAO 代签"遗留白名单凑够第 5 签。FBI 与 Chainalysis 于 2022-04-14 正式归因朝鲜 Lazarus Group。

## 1. 事件背景

### 1.1 Ronin 与 Axie Infinity

Ronin 是 Sky Mavis 为链游 Axie Infinity 打造的 EVM 兼容侧链，采用 Proof-of-Authority（PoA）共识，上线时仅 9 个 validator。Ronin Bridge 是其与 Ethereum 主网之间的资产跨链桥：用户把 ETH、USDC、AXS、SLP 从 Ethereum 存入，合约在 Ronin 侧释放等值包装代币，反向提现需要至少 5/9 validator 共同签名。

Axie Infinity 在 2021 年 Q3 顶峰时 DAU 超过 270 万，2022 年初 Ronin Bridge 托管的资产规模超过 10 亿美元，是 Play-to-Earn 赛道的标志性项目。Sky Mavis 自营 4 个 validator，Axie DAO、Creator Fund、Animoca Brands、Nansen 等外部机构各运行若干 validator 以保持去中心化表象。

### 1.2 时间轴

- **2021-11**：Axie DAO 白名单 Sky Mavis 代为签发交易以应对用户压力，该"临时"授权此后未被显式回收。
- **2022-03-23 UTC 约 11:45**：攻击者在 Ronin Bridge 发起两笔大额提款：173,600 WETH 与 25.5M USDC。
- **2022-03-23 至 03-29**：6 天内无人察觉——Ronin 的监控告警被攻击者发起的小额正常提款流量掩盖，且 Sky Mavis 未对 validator 私钥进行独立活性审计。
- **2022-03-29**：一位用户发现无法从桥提现 5,000 ETH，报告至 Sky Mavis；团队交叉比对多签 validator 日志才发现两笔大额提款未在内部审批系统留痕。
- **2022-03-29 UTC 晚**：Sky Mavis 官方 Substack 发布《Community Alert: Ronin Validators Compromised》，暂停桥、AXS/SLP/RON 代币价在 24h 内下挫 20%+。
- **2022-04-06**：Binance 首批注资 1.5 亿美元，Sky Mavis / Animoca / a16z 追加 1.5 亿美元共 3 亿融资，用于用户赔付。
- **2022-04-14**：美国财政部 OFAC 把收款地址 `0x098B716B8Aaf21512996dC57EB0615e2383E2f96` 列入制裁名单；FBI 同日正式归因 Lazarus Group。
- **2022-06-28**：Ronin Bridge 恢复，validator 数量扩充到 11 并引入断路器。

### 1.3 发现过程

按 Sky Mavis 官方 post-mortem 叙述，发现纯属偶然：一名用户在客服反馈"5k ETH 提不出来"，Sky Mavis DevOps 调查时查 Ronin 节点的 deposit/withdraw 事件日志，发现两笔 6 天前的巨额提款对应的 validator 签名集合包含 Axie DAO gas-free RPC 节点——该节点自 2021-12 起理论上已不应代签任何交易。这种"被动发现"本身就是事后复盘的第一条教训。

## 2. 事件影响

### 2.1 直接损失

- **173,600 ETH**，按 2022-03-23 均价 $2,916 计约 $506M；
- **25.5M USDC**；
- 合计按当日价 **约 $625M**（有文档引用 $620M 或 $624M，差异在 ETH 价格快照口径）。

两笔交易分别从 Ronin Bridge Ethereum 合约 `0x8407dc57739bCDA7AA53Ca6F12F82F9d51C2F21E`（Ronin Bridge v2）提出，攻击者接收地址为 `0x098B716B8Aaf21512996dC57EB0615e2383E2f96`。

### 2.2 受害方

Ronin Bridge 以 1:1 托管模型运作，合约侧被抽空意味着：所有持有 WETH-on-Ronin、USDC-on-Ronin 的用户名义上变成"无准备金的 IOU"。Sky Mavis 承诺全额赔付，通过新一轮融资和自有金库弥补缺口，两年内逐步解决。

### 2.3 连带影响

- AXS 代币价从事件前 $68 跌至 4 月底 $45，SLP、RON 同步下挫；Axie 游戏内经济因链下美元流动性紧张出现通缩。
- 桥赛道集体承压：2022 年 2 月 Wormhole、3 月 Ronin、8 月 Nomad、10 月 BNB Cross-Chain 四大事件总损失超 15 亿美元，多链叙事信任度大幅降低。
- 引发业界对 PoA / 小集合多签桥模型的全面反思，ZK-bridge、optimistic-bridge 加速立项。

### 2.4 资金去向

攻击者 6 天分步混洗：先把 USDC 换成 ETH，再通过 Tornado Cash 分批洗净。Chainalysis 与 Elliptic 追踪显示，截至 2022-09，至少 87% 资金进入 Tornado Cash，剩余部分通过受制裁的 Blender.io（后继者 Sinbad.io）与部分亚洲 OTC 渠道。2022-04-14 OFAC 制裁后，Circle 冻结其路径上约 550 万 USDC；小额在 Huobi（当时名）被冻结。

## 3. 技术根因（核心）

### 3.1 漏洞分类

**密钥管理 + 治理 / 白名单残留 + 社会工程**。三类叠加，非智能合约漏洞。

### 3.2 受损模块

- Ronin validator 集合：9 个 PoA validator，其中 4 个由 Sky Mavis 运营。
- `RoninBridgeV2` 以太坊侧合约：`0x8407dc57739bCDA7AA53Ca6F12F82F9d51C2F21E`。
- `MainchainValidator` / `BridgeTracking` 在 Ronin 侧；提款流程要求收集 validator 对 `receiptHash` 的 ECDSA 签名，合约内 `verifyThreshold = 5`。

### 3.3 关键逻辑（简化伪代码，展示"为何 5/9 签名足以让攻击者得逞"）

```solidity
// 简化示意 —— 基于 Ronin Bridge v2 withdrawFor 逻辑
// 受损合约：0x8407dc57739bCDA7AA53Ca6F12F82F9d51C2F21E
function withdrawFor(
    Receipt memory receipt,
    Signature[] memory sigs   // 由链下 validator 签出，广播到以太坊
) external {
    bytes32 h = hashReceipt(receipt);
    uint256 weight = 0;
    address last = address(0);
    for (uint i = 0; i < sigs.length; i++) {
        address signer = ecrecover(h, sigs[i].v, sigs[i].r, sigs[i].s);
        require(signer > last, "sig order");   // 防重复
        last = signer;
        if (isValidator[signer]) {
            weight += validatorWeight[signer]; // 每个 validator 权重 1
        }
    }
    require(weight >= THRESHOLD, "not enough sigs"); // THRESHOLD = 5
    _releaseToken(receipt); // 放款到 receipt.recipient
}
```

"错"不在这段代码本身，而在于：
1. 攻击者持有 **4 把 Sky Mavis validator 私钥**（经 LinkedIn 假 offer 定向社工入侵 Sky Mavis 员工机器）；
2. Axie DAO 曾在 2021-11 授权 Sky Mavis "代签"以缓解用户活动洪峰——即 Sky Mavis 的 RPC 节点可以直接用 Axie DAO validator 的身份出签，且该白名单未被时间锁 / 代签额度 / 自动失效策略限制；
3. 2022-03-23，攻击者在攻陷的 Sky Mavis 基础设施上触发一次"合法"代签，把 Axie DAO 的第 5 签也补齐，最终向以太坊合约提交 5 个有效签名。

### 3.4 攻击步骤分解

1. **初始立足**：至少 2021 年底起，Lazarus 在 LinkedIn 针对 Sky Mavis 资深工程师投放假招聘。一名员工下载 PDF "offer"，内含恶意宏 / dropper。Sky Mavis post-mortem 与 The Block 报道确认该员工机器被植入 RAT。
2. **横向移动**：利用该员工 VPN / SSH 权限跳入内部 RPC / validator 节点子网，获取 4 把 Sky Mavis validator 的签名密钥（热钱包形式存放于运营节点）。
3. **发现 Axie DAO 白名单**：侦察期间识别 Sky Mavis RPC 节点仍配置了 Axie DAO validator 代签权限（2021-11 为用户活动临时加上）。
4. **构造提款 receipt**：伪造两张巨额 withdraw receipt，使用 4 把 Sky Mavis 密钥 + 1 次 Axie DAO 代签调用。
5. **上链执行**：向以太坊 `RoninBridgeV2.withdrawFor` 递交签名集合，合约验证 5 签通过，放款 173,600 ETH + 25.5M USDC 至 `0x098B716B8...`。
6. **洗币**：经 6 天无察觉后开始通过 Tornado Cash 混入 100 ETH 切片。

### 3.5 为何审计未发现

Ronin Bridge v2 合约本身由 Verichains 审计并无明显漏洞。审计范围是**合约**而非**运营架构**：审计方通常不审计 validator 私钥存放位置、白名单是否及时回收、人员访问边界。Sky Mavis 在 2021-11 授权 Axie DAO 白名单时也未召开安全评审。监控层面，桥没有"单笔 > 10k ETH 自动熔断"等断路器逻辑，这也是事后最易打补丁的一项。

## 4. 事后响应

### 4.1 项目方动作

- **2022-03-29**：暂停桥、暂停 Ronin-Katana DEX 跨链操作。
- **2022-04-06**：Binance 领投 1.5 亿美元，Sky Mavis / Animoca / a16z / Paradigm 共同追加 1.5 亿美元，总共 **$300M** 融资用于用户赔付。
- **2022-04-07**：Sky Mavis 发布详细 post-mortem（Substack），披露 LinkedIn 假面试链路。
- **2022-06-28**：桥重启，validator 扩至 11 家（新增 Dapper Labs、Community Gaming 等），threshold 提到 8/11，引入每日提款上限与熔断，向主网加装 circuit breaker。
- 长期：validator 私钥迁入 HSM，密钥轮换 SLA、渗透测试季度化。

### 4.2 资产追回

截至 2023-09，Sky Mavis 全额恢复用户 ETH/USDC 余额；追回率链上不高（<5%），大部分损失由融资与自有资金承担。OFAC 制裁配合 Tether/Circle 冻结了个位数百万美元稳定币。

### 4.3 法律 / 执法

- **2022-04-14**：OFAC 把攻击者地址列入 SDN 名单；FBI 官方声明归因 Lazarus Group（朝鲜国家背景）；
- Chainalysis 与 Elliptic 独立发布追踪报告确认资金流向 Lazarus 控制簇；
- 2022-08：美国财政部制裁 Tornado Cash 协议，Ronin 资金是直接触发因素之一；
- 2023-11：美国 SEC / DOJ 对 Lazarus 相关洗钱中介启动起诉，部分链下 OTC 被封。

### 4.4 复查与审计升级

Verichains、OpenZeppelin 参与重启后的架构审计；Sky Mavis 引入 Halborn 做持续渗透测试。行业层面，Chainlink CCIP、Axelar、LayerZero 等后续跨链项目把"validator 密钥 → HSM + MPC"与"断路器"作为默认最佳实践。

## 5. 启发与教训

### 5.1 对开发者

- **威胁模型必须含"运营链"**：智能合约只是冰山一角。多签桥的真实攻击面是 validator 私钥存放、签名 RPC 节点、代签白名单。
- **最小权限 + 时间锁**：任何"临时代签"白名单必须有强制过期日期或 timelock，默认回收。
- **断路器（circuit breaker）**：单笔、单日大额提款自动冻结等待人工审批；Ronin 重启后加装。
- **密钥多样化**：4 个 validator 用同一员工机器托管等于 1 把钥匙。

### 5.2 对审计方

- 审计合约 ≠ 审计系统。跨链桥项目应有单独的 **operational security audit**（运营安全审计），覆盖 validator 架构、密钥管理、监控告警。
- 建议 threshold 值与 validator 实际独立性同步校验：4/9 在一家公司 = 有效 1 签。

### 5.3 对用户

- 跨链资产集中在单桥是系统性风险；大额用户应分布于多个桥或保留在源链。
- 检查桥的 validator 集合独立性与断路器配置，不要只看 TVL。

### 5.4 对协议

- Timelock 治理与独立 validator 不是摆设，是活命要件；
- 员工安全培训纳入 on-chain 安全预算，LinkedIn / Telegram / Discord 假招聘是朝鲜组织标准模板；
- 监控必须覆盖"未经内部审批系统的 validator 签名"。

## 6. 参考资料

- Sky Mavis 官方 post-mortem：https://roninblockchain.substack.com/p/community-alert-ronin-validators
- Chainalysis 归因：https://www.chainalysis.com/blog/lazarus-group-axie-infinity-ronin-bridge-dprk/
- FBI 声明：https://www.fbi.gov/news/press-releases/fbi-statement-on-attribution-of-malicious-cyber-activity-posed-by-the-democratic-peoples-republic-of-korea
- rekt.news：https://rekt.news/ronin-rekt/
- The Hacker News LinkedIn 链路报道：https://thehackernews.com/2022/07/axie-infinity-hack-how-fake-job-offer.html
- Elliptic 资金追踪：https://www.elliptic.co/blog/540-million-stolen-from-the-ronin-blockchain
- Ethereum 攻击者地址（OFAC 制裁）：`0x098B716B8Aaf21512996dC57EB0615e2383E2f96`
- Ronin Bridge v2 合约：`0x8407dc57739bCDA7AA53Ca6F12F82F9d51C2F21E`

---

*Last verified: 2026-04-23*
