---
title: Munchables 朝鲜前员工后门案（2024-03-26, $62.5M, 全额归还）
module: 08-security-incidents
category: Social-Eng | Supply-Chain | Protocol-Bug
date: 2024-03-26
loss_usd: 62500000
chain: Blast
severity: Tier-3
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/munchables-rekt/
  - https://x.com/zachxbt/status/1772623546553307551
  - https://slowmist.medium.com/slowmist-brief-analysis-of-the-munchables-incident
  - https://x.com/peckshield/status/1772590996632838151
  - https://x.com/_munchables_/status/1772754626201182506
tx_hashes:
  - 0x9a7b1e12e7c3e7b0d6c4a8f1f4b9c5e9f0c7d1b3e8e2f5a1b6c3d0e7f4a9b2c5 (Blast, 攻击者 withdraw 17,413 ETH)
  - 攻击者/前雇员地址: 0x6E8836F050A315611208A5CD7e228701563D09c5
---

# Munchables 朝鲜前员工后门案

> **TL;DR**：2024-03-26，Blast L2 上 Munchables 被**前开发者**（ZachXBT 追踪其用 4 GitHub 账号应聘、疑似朝鲜 IT 劳动力）利用其在 LockManager UUPS 实现中预埋的后门盗走 17,413 ETH（≈ $62.5M）。5h 后在 Pacman 公开施压下**全额归还**。2024 年"开发者背调"议题标志案例。

## 1. 事件背景

- **项目**：Munchables，2024-02 Blast 上线 NFT 养成游戏，匿名创始人 @plotchy，积累 17,400+ ETH。
- **攻击者**（ZachXBT 调查）：同一人用 4 GitHub 账号（Werewolves0493、NelsonMurua913、Super1114、Bng420）应聘 5+ 加密项目，账号互相推荐、共用付款钱包，资金流与已知朝鲜 IT 劳动力集群重叠。
- **时间轴**：2023-10 入职 → 03-26 20:20 UTC 触发后门 → 03-27 01:30 Pacman 施压后全额归还。

## 2. 事件影响

临时损失 17,413 ETH ≈ $62.5M；实际损失 ≈ $0；Blast 不回滚，玩家约 12h 无法提现。Blast 当周第二起（前有 Super Sushi Samurai $4.8M）；OFAC 2024-Q4 DPRK 警示引用 ZachXBT 方法。

## 3. 技术根因

### 3.1 分类

**Supply-Chain 社工 + 未验证的可升级合约**。不是代码 bug，而是**部署流程信任模型**崩塌。

### 3.2 后门机理

```solidity
// Malicious LockManager v1 (伪代码, implementation 未 verify)
contract LockManagerV1 is UUPSUpgradeable, OwnableUpgradeable {
    mapping(address => uint256) public deposits;
    function initialize() public initializer { __Ownable_init(); } // ❶ owner=部署者
    // ❷ 后门: owner 可改写任意用户 deposit
    function setAssignedAmounts(address[] calldata u, uint256[] calldata a) external onlyOwner {
        for (uint i = 0; i < u.length; i++) deposits[u[i]] = a[i];
    }
    function withdraw(uint256 amt) external {
        require(deposits[msg.sender] >= amt);
        deposits[msg.sender] -= amt;
        payable(msg.sender).transfer(amt);
    }
}
```

**错在哪**：❶ 部署者即 owner，proxy admin "转交" 未对 **implementation source** 独立审阅；❷ `setAssignedAmounts` 等同后门——owner 给自己写 17,413 ether 后合法 `withdraw`；implementation **从未 verify**，社区无从审阅。

### 3.3 攻击步骤

2023-10 入职部署后门合约且未 verify → 潜伏 5 月，TVL 17,413 ETH → 03-26 20:20 `setAssignedAmounts([self], [17_413e18])` → `withdraw` → ZachXBT 公布 4 GitHub 聚类，Pacman 施压，5h 全额归还。

### 3.4 为何未发现

无第三方审计；implementation 未 verify；部署者未 KYC。

## 4. 事后响应

Munchables 重部署（PeckShield 审计），开发者 KYC。FBI 2024-05 发 DPRK IT 劳动力警示；OFAC 2024-10 制裁 DPRK 帮扶集群。ZachXBT 调查范式被行业复用。

## 5. 启发与教训

- 招聘合约开发者必须 KYC + 背调；禁止匿名开发者单独持部署 / owner 权限。
- UUPS implementation 必须 verify + 第三方审计；避免"初始化者即 owner"。

## 6. 参考资料

- rekt.news: Munchables Rekt
- ZachXBT Twitter 调查（2024-03-26）
- PeckShield Alert、SlowMist 事件解析
- Munchables 官方公告（2024-03-27）
- FBI: North Korean IT Workers Warning

---

*Last verified: 2026-04-23*
