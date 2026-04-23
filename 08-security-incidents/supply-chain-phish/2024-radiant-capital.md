---
title: Radiant Capital 多签硬件钱包盲签案（2024-10-16, ~$50M）
module: 08-security-incidents
category: Wallet | Social-Eng | Supply-Chain
date: 2024-10-16
loss_usd: 50000000
chain: Ethereum, Arbitrum, BSC
severity: Tier-2
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/radiant-capital-rekt2/
  - https://medium.com/@RadiantCapital/radiant-capital-incident-update-e56d8c23829e
  - https://www.mandiant.com/resources/blog/applejeus-north-korea-finance
  - https://slowmist.medium.com/slowmist-analysis-of-the-radiant-capital-hack
  - https://x.com/peckshield/status/1846738766834749871
tx_hashes:
  - 0xa9fad53ee8a8c9b6e86dccbebfbf9cee7ea8c3a6bb37e9e7e63d6c10dcd9f1c7 (Arbitrum, 主攻击 transferOwnership / upgrade)
  - 0x3d7c0d2b5a67ec28c33f6d7f1f2a41abc74a9cd0a9b92b37c6c5b28a6e86d2b9 (BSC, 主攻击 transferOwnership / upgrade)
  - Radiant DAO Multisig: 0x111CEEee040739fD91D29C34C33E6B3E112F2177
---

# Radiant Capital 多签硬件钱包盲签案

> **TL;DR**：2024-10-16，跨链借贷协议 Radiant Capital（TVL ≈ $300M）被朝鲜黑客组织 UNC4736 / AppleJeus / Citrine Sleet 通过**社工 + macOS 后门 INLETDRIFT + 硬件钱包 UI 盲签**组合攻击，获得 3 名签名者的合法签名，在 Arbitrum 与 BSC 上执行 `transferOwnership` + UUPS `upgradeTo` 把 Radiant Pool 合约逻辑替换为恶意实现，随即以 owner 身份抽走用户对 lending pool 授予的 allowance，共损失约 $50M。Mandiant 高置信度归因朝鲜。事件证明即使硬件钱包 + 多签 + Tenderly 模拟齐备，签名 UI 层一旦被污染，安全边界即整体崩溃。

## 1. 事件背景

- **协议**：Radiant Capital，跨链借贷（LayerZero OFT），Arbitrum / BSC / ETH / Base。攻击前 TVL ≈ $300M。DAO 治理由 **3/11 Gnosis Safe** 管理 Pool owner（阈值被批评过低）。
- **时间轴**：2024-09-11 开发者收到伪装 PDF 的 ZIP，内含 macOS 后门 INLETDRIFT；9–10 月后门在多签名者工作站潜伏篡改 Safe UI；10-10 失败试探；**10-16 17:21 UTC** 3 签到位，tx 在 Arbitrum + BSC 同时提交，Pool owner 被劫持；12-06 Radiant 公开 Mandiant 归因。
- **发现**：PeckShield、慢雾、Radiant 签名者几乎同时。

## 2. 事件影响

- **直接损失**：约 $50M（rekt.news 记 $53M），覆盖 USDC、USDT、WBTC、ARB 等。
- **特点**：损失由**已对 lending pool approve 的普通用户**承担，非 DAO treasury。
- **代币**：RDNT 当日 -34%。
- **资金去向**：桥回 Ethereum → Tornado Cash → THORSwap，大部分未追回。

## 3. 技术根因（代码级）

### 3.1 分类

**Social Engineering + Supply Chain + Hardware Wallet Blind Signing**，与 WazirX（2024-07）、DMM Bitcoin（2024-05）同模式，Lazarus TraderTraitor 2024 三大案。

### 3.2 UUPS upgradeTo 盲签

```solidity
// UUPSUpgradeable (简化) — Radiant LendingPool
function upgradeTo(address newImplementation) external onlyProxy {
    _authorizeUpgrade(newImplementation);  // 仅 onlyOwner, 无 timelock
    _upgradeToAndCallUUPS(newImplementation, new bytes(0), false);
}
function _authorizeUpgrade(address) internal override onlyOwner {}
```

**错在哪**：
- owner 为 3/11 Safe，被钓鱼即可 `upgradeTo(malicious)` 一笔 tx 完成升级，**无 timelock 缓冲**。
- Safe 在 Ledger 上只能展示 `safeTxHash`（hash 十六进制），签名者不具备设备端解码 `upgradeTo(addr)` 的能力；UI 层被 INLETDRIFT 篡改后，真实 `newImplementation` 是恶意合约。
- 即使启用 Clear Signing，签名者看到"upgradeTo(0x...)"仍无法判断目标实现的恶意性，需要 off-chain Tenderly 模拟——而模拟页面同样被后门替换。

### 3.3 攻击步骤

1. 钓鱼（09-11）：`Penetration_Testing_Report.zip` 植入 INLETDRIFT 后门。
2. 潜伏：后门替换浏览器 DOM，篡改 Safe 审批页与 Tenderly 模拟结果。
3. 失败试探（10-10）：某签名者异常中止，攻击者迭代。
4. 执行（10-16 17:21 UTC）：3 签者在被污染的 UI 上审批"例行 admin 操作"，实际签名 `transferOwnership(attacker)`；tx 在 Arbitrum + BSC 同步提交。
5. 升级 & 抽款：`upgradeTo(maliciousImpl)`；新实现暴露 `drainAllowances` 类函数，批量拖走 approve 余额。
6. 洗钱：桥到 Ethereum → Tornado Cash。

### 3.4 为何未发现

Ledger 对自定义 Safe tx 只能展示 hash；Tenderly 模拟被后门替换 DOM；3/11 阈值过低；升级操作无 Timelock，Forta 无告警窗口。

## 4. 事后响应

- **Radiant**：阈值提至 7/11 + 48h Timelock；2024-12 发布事后报告。
- **Mandiant**：高置信度归因 **UNC4736 / AppleJeus / Citrine Sleet**（DPRK 侦察总局）。
- **执法**：FBI 2024-12 TraderTraitor PSA 提及。
- **行业**：Safe Foundation 推 "Transaction Simulator 2.0"；Ledger 发布 ERC-7730 Clear Signing metadata 标准。

## 5. 启发与教训

- UUPS `upgradeTo` 必须走 Timelock；禁止 Safe 直接升级生产合约。
- 签名者使用隔离专用设备（禁 Telegram / 邮件）；两台独立设备对比 raw calldata，不盲签。
- 审计威胁模型必须覆盖"签名者端被社工"。
- 高 TVL 协议阈值 ≥ 60%，签名者跨组织分布。

## 6. 参考资料

- rekt.news: Radiant Capital Rekt 2
- Radiant Capital Incident Update（Medium, 2024-12-06）
- Mandiant: AppleJeus / Citrine Sleet
- 慢雾 SlowMist 事件解析
- PeckShield Alert（2024-10-16）
- FBI IC3 I-122324-PSA

---

*Last verified: 2026-04-23*
