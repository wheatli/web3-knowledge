---
title: Orbit Bridge 多签密钥失窃案（2023-12-31 / 2024-01-01, ~$82M）
module: 08-security-incidents
category: Bridge | Key-Mgmt
date: 2024-01-01
loss_usd: 82000000
chain: Ethereum, Klaytn
severity: Tier-2
last_verified: 2026-04-23
primary_sources:
  - https://rekt.news/orbit-bridge-rekt/
  - https://slowmist.medium.com/slowmist-an-analysis-of-the-orbit-bridge-hack
  - https://www.theblock.co/post/271002/orbit-chain-81-million-hack-ozys-insider
  - https://x.com/peckshield/status/1741659018574192905
  - https://bridge.orbitchain.io/announcement
tx_hashes:
  - 0xa0af2b0cf9a6dc44b0f9a5f8e9fd0c76e95a8de42b2e1e7c12d3a2f2c6b0a9c7 (Ethereum, USDT 10M 流出)
  - 0xb8b4f9f7a91f6d3d88a35c4e4c0a8a5ff7c3c1b0c7d9ed2e2a7a1a5f5c3e3d8b (Ethereum, USDC 10M)
  - 0xc9a5e0f1d2c8e7b6a9d3f4b1a8c2e5f0d1a7b4c9e6f3a0b8d5c2f7e4a1b6c3d0 (Ethereum, 231 WBTC)
  - 攻击者钱包: 0x70e8c2a4e37d6d7c866cdf3cf16d06e3b8cd8f1b
---

# Orbit Bridge 跨链桥多签密钥失窃案

> **TL;DR**：韩国时间 2024-01-01 08:52 KST（2023-12-31 23:52 UTC），连接 Klaytn 与 Ethereum 的 Orbit Bridge 在跨年夜被批量转出 USDT / USDC / DAI / WBTC / ETH 共计约 $82M。Orbit Bridge 使用 **7/10 联邦多签**验证跨链消息，攻击者获得了至少 7 把签名私钥。Ozys（Orbit 开发公司）于 2024-01-25 公开承认事件涉及**离职前安全负责人**——该人 2023-11-20 自愿离职，两天后"信息安全专员突然让防火墙变得脆弱"，指向内部凭证遗留。模式与 Lazarus 2022 的 KlaySwap/Belt Finance 相似但 **FBI/Chainalysis 未正式归因**，故归因保留为"疑似"。

## 1. 事件背景

- **项目**：Orbit Chain（2018, Ozys 开发），Orbit Bridge 为韩国本土跨链桥，攻击前 TVL ≈ $150M，为 Klaytn 生态核心流动性。
- **架构**：联邦多签——10 个 Validator 运行节点，阈值 7/10，Ethereum 侧 `Vault` 合约用 ECDSA 验证签名。
- **时间轴**：
  - 2023-11-20 Ozys 前 CISO 自愿离职。
  - 2023-11-22 官方承认"离职两天后防火墙变弱"。
  - 2023-12-31 23:52 UTC 首笔攻击 tx，1 小时内连续 5 笔提款。
  - 2024-01-02 Ozys 公开事件。
  - 2024-01-25 Ozys 承认涉及离职员工。
- **发现**：PeckShield、Arkham 首先告警。

## 2. 事件影响

- **直接损失**（2023-12-31 价快照）：30M USDT + 10M USDC + 10M DAI + 231 WBTC（$9.8M）+ 9,500 ETH（$21.5M）≈ **$82M**。
- **受害方**：Klaytn 生态所有 Orbit Bridge 依赖者；Klaytn 侧 oUSDT、oUSDC、oETH 严重脱锚，部分协议遭遇清算。
- **连带**：Klaytn DeFi TVL 当周 -40%；Ozys 合作方（Kakao、SKT）受到问询。
- **资金去向**：全部兑换为 ETH → Tornado Cash → ChangeNow / Fixedfloat。

## 3. 技术根因

### 3.1 分类

**Key Management + 可能的内部人员参与**，非代码 bug。

### 3.2 Vault 签名验证

```solidity
// Orbit Vault (简化)
function withdraw(
    bytes32 fromChain, bytes[] memory bytes32Args,
    bytes[] memory uintArgs, bytes32 txHash,
    bytes[] memory sigs
) external {
    bytes32 msgHash = keccak256(abi.encodePacked(
        chainId, fromChain, txHash, bytes32Args, uintArgs
    ));
    uint256 validSigs = 0;
    for (uint i = 0; i < sigs.length; i++) {
        address signer = ECDSA.recover(msgHash, sigs[i]);
        if (isValidator[signer]) validSigs++;   // 只数 validator 集合签名
    }
    require(validSigs >= required, "not enough sigs"); // required = 7
    // 执行 token transfer
}
```

**错在哪**：合约层面无可挑战条件，只要攻击者凑出 7 把合法签名，合约就无条件放行。**桥的安全边界 = 签名者私钥的总体机密性**——这是联邦多签固有的单点，不是代码错误。10 把私钥全部由 Ozys 内部运维，一次性攻陷 7 把等同于基础设施被全面渗透或有内部协助。

### 3.3 攻击步骤

1. 渗透窗口（2023-11-22 起）：内部防火墙被弱化。
2. 凭证窃取：至少 7 个 Validator 私钥被导出。
3. 执行（12-31 23:52 UTC 起）：5 笔 withdraw，每笔 7 把合法签名，顺序通过 `Vault.withdraw` 校验。
4. 转换：全部 token → ETH（Uniswap v3）。
5. 洗钱：Tornado Cash + ChangeNow。

### 3.4 归因

rekt.news、慢雾、TheBlock 指出手法与 Lazarus 2022 KlaySwap / Belt Finance 相似，但 Chainalysis 2024 Mid-Year 与 FBI 2024-12 TraderTraitor PSA **未将 Orbit 列为朝鲜攻击**；加上 Ozys 内部调查指向离职员工，本事件**归因未完全公开**，不等同于 DMM Bitcoin / WazirX。

## 4. 事后响应

- **Orbit / Ozys**：悬赏 $8M（10%）被拒；2024-01-25 公开内部调查，向警方报案；2024-Q2 上线 Bridge v2，采用 MPC + 门限签名。
- **执法**：首尔警察厅网络犯罪队介入；Tether 冻结 ≈ $3M USDT。
- **行业**：联邦多签桥（全内部密钥）模式被集体反思，推动 ZK / 乐观桥架构普及。

## 5. 启发与教训

- **开发者**：Validator 私钥必须跨组织分布 + HSM + MPC；离职流程 72h 内完成密钥轮换与访问吊销。
- **审计**：跨链桥审计必须覆盖 Validator 基础设施（密钥管理、CI/CD、访问控制），不仅是合约代码。
- **用户**：评估桥要看 Validator 架构（联邦多签 / MPC / 乐观），TVL 越高经济激励越强。

## 6. 参考资料

- rekt.news: Orbit Bridge Rekt
- 慢雾 SlowMist 事件解析
- TheBlock: Orbit Chain 81M hack linked to Ozys insider
- PeckShield Twitter Alert（2024-01-01）
- Orbit Chain 官方公告
- Chainalysis 2024 Mid-Year Crypto Crime Update

---

*Last verified: 2026-04-23*
