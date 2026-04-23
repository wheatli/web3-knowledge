---
title: Parity Multisig 1.5 钱包 153,037 ETH 被盗（2017-07-19, ~$30M 按当时价）
module: 08-security-incidents
category: Wallet
date: 2017-07-19
loss_usd: 30000000
chain: [Ethereum]
severity: Tier-2
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.parity.io/the-multi-sig-hack-a-postmortem/
  - https://blog.zeppelin.solutions/on-the-parity-wallet-multisig-hack-405a8c12e8f7
  - https://hackingdistributed.com/2017/07/22/parity-wallet-hack/
  - https://etherscan.io/address/0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4
tx_hashes:
  - 攻击 tx：0xeef10fc5170f669b86c4cd0444882a96087221325f8bf2f55d6188633aa7be7c
  - 受攻击钱包（Edgeless Casino）：0x1DBa1131000664b884A1BA238464159892252D3a
  - 受攻击钱包（Swarm City）：0x7Dc4f41294697a7903C4027f6Ac528C5d14cd7eB
  - 受攻击钱包（æternity）：0x35A066F0F0dF9D4B968B0A065FA802489a5ef26a
---

# Parity Multisig 1.5 钱包漏洞

> **TL;DR**：2017 年 7 月 19 日，Parity Multisig Wallet v1.5 合约被匿名黑客（自称 "0xbaka"）利用一个关键的 `initWallet` 初始化函数可被任何外部账号重新调用的漏洞，接管了 3 个知名 ICO 项目的多签钱包（Edgeless、Swarm City、æternity）并提走共 153,037 ETH（约 3,000 万美元）。白帽组织 White Hat Group（WHG）发现后以同样手法抢救了剩余约 377,000 ETH 至受控地址，再逐一返还给合法 owner。漏洞本质为：在 Library contract 通过 `delegatecall` 被代理时，未对初始化函数设置一次性保护或访问控制。

## 1. 事件背景

- **主体**：
  - Parity Technologies（Gavin Wood 创办）：Parity Ethereum 客户端的开发方。
  - Parity Multisig Wallet：当时以太坊最常用的多签钱包合约之一，v1.5 自 2016 年末部署，广泛被 ICO 项目用于托管募资。
- **时间轴**：
  - 2017-07-19 12:33 UTC（区块 4041179 附近）：攻击者向 Edgeless Casino 多签钱包发送 tx，调用 `initWallet`，重置 owner。
  - 2017-07-19 随后几小时：攻击者依次攻陷 Swarm City、æternity 多签钱包，共卷走 153,037 ETH。
  - 2017-07-19 下午：White Hat Group（Dexaran、Jordi Baylina 等）使用同一漏洞将剩余高风险钱包内的 ~377,000 ETH 抢救至新钱包。
  - 2017-07-20：Parity 官方发布 post-mortem。
  - 2017-07-24：WHG 开始逐一将被救 ETH 返还给合法 owner。
- **发现过程**：Parity 社区成员发现知名 ICO 钱包被重置；链上归因在小时级内完成。

## 2. 事件影响

- **直接损失**：153,037 ETH（~$30M 按 ~$196 估）。
  - Edgeless Casino：26,793 ETH
  - Swarm City：44,055 ETH
  - æternity：82,189 ETH
- **受害方**：3 个 ICO 项目的金库资金；ICO 投资者间接损失。
- **连带影响**：
  - 引发智能合约钱包库架构争议，直接为 2017-11 Parity 2.0 library freeze 事件埋下伏笔（相同代码库的升级版再次出事，见 `2017-parity-freeze.md`）。
  - 推动 OpenZeppelin `Initializable` 模式与 EIP-1167（Minimal Proxy）等标准。
- **资金去向**：攻击者将 ETH 转入单一地址并长期静置；部分通过 ShapeShift 等渠道分散。归因未公开（无 FBI / Chainalysis 正式报告）。

## 3. 技术根因

- **漏洞分类**：Access-Control + Initialization + Proxy Pattern Misuse。
- **受损合约**：Parity Multisig v1.5，主库合约 `WalletLibrary` 部署于 `0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4`；用户钱包为轻量级 proxy，通过 `delegatecall` 调用库。
- **关键代码片段**（从 Parity 公开 post-mortem 与 Etherscan verified 源码还原，标注错在哪一行）：

```solidity
// WalletLibrary.sol（Parity Multisig v1.5, 简化）
contract WalletLibrary {

    // 问题函数：initMultiowned 用作初始化 owner 列表
    function initMultiowned(address[] _owners, uint _required) {
        // ====== 错在这里 ======
        // 没有 "onlyUninitialized" / "onlyOwner" / "once" 的守卫
        // 既可以被 proxy 构造时的 delegatecall 调用（合法），
        // 也可以被任何外部账号在任意时刻调用（非法但合约未禁止）
        m_numOwners = _owners.length + 1;
        m_owners[1] = uint(msg.sender);
        m_ownerIndex[uint(msg.sender)] = 1;
        for (uint i = 0; i < _owners.length; ++i) {
            m_owners[2 + i] = uint(_owners[i]);
            m_ownerIndex[uint(_owners[i])] = 2 + i;
        }
        m_required = _required;
    }

    // 外层 initWallet 同样缺保护
    function initWallet(address[] _owners, uint _required, uint _daylimit) {
        initDaylimit(_daylimit);
        initMultiowned(_owners, _required);
    }
}
```

用户的 proxy 合约（`Wallet.sol`）通过 `delegatecall` 将所有调用转发到库：

```solidity
// 用户的多签钱包 proxy（简化）
contract Wallet {
    address constant _walletLibrary = 0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4;
    function() payable {
        // 任何未匹配的 selector 都会 delegatecall 到库
        if (msg.value > 0)
            Deposit(msg.sender, msg.value);
        else if (msg.data.length > 0)
            _walletLibrary.delegatecall(msg.data);
    }
}
```

由于 `initWallet` 没有访问控制，攻击者只需：

1. 构造一笔调用 Wallet 合约的 tx，calldata 为 `initWallet(newOwners, 1, ...)`。
2. Wallet proxy 通过 fallback + delegatecall 将调用转发到 WalletLibrary。
3. WalletLibrary 在 proxy 的 storage context 下覆盖 owner 列表。
4. 攻击者现在是唯一 owner，可以发 `execute()` 将 ETH 转出。

- **攻击步骤**：
  1. tx `0xeef10f...be7c`：对 Edgeless Casino 钱包调用 `initWallet`，重置 owner 为攻击者地址。
  2. 紧接一笔 tx 调用 `execute(attackerAddr, 26793 ether, ...)` 将全部 ETH 转出。
  3. 对 Swarm City、æternity 重复上述 2 步。
- **为何审计未发现**：Parity 内部 review 对"proxy delegatecall"的初始化语义理解不足；当时 EIP-1167 / OpenZeppelin Initializable 尚未成为标准。

## 4. 事后响应

- **项目方**：
  - 2017-07-20 发布 post-mortem，上线修复版本 v1.5.2（为 `initWallet` 增加 `only_uninitialized` 修饰符）。
  - 建议所有 v1.5 用户迁移到新钱包（埋下 2017-11 事件伏笔）。
- **WHG 白帽行动**：在攻击发生当日，用同一漏洞将约 377,000 ETH 抢救到白帽控制地址，随后逐一返还。
- **执法**：未公开追责结果；攻击者地址长期静置。
- **行业连锁**：
  - OpenZeppelin 推出 `Initializable.sol`，将 `initializer` modifier 标准化。
  - 审计方把"代理合约初始化函数访问控制"列为最高优先级检查项。

## 5. 启发与教训

- **对开发者**：
  - 所有用于代理初始化的函数必须使用 `initializer` 一次性保护（OpenZeppelin 模式）。
  - Library / Implementation 合约需在 deploy 时自行 `_disableInitializers()`，防止被直接调用。
  - `delegatecall` 语义：调用者的 storage 被修改，调用者必须完全信任被调用代码。
- **对审计方**：
  - Slither 的 `unprotected-upgrade` 规则应被广泛启用。
  - 在 proxy 架构中必须验证 initializer 的访问控制与 once-only 语义。
- **对用户 / 项目方**：ICO 金库托管必须使用经过充分审计的多签方案（如 Gnosis Safe），而非新兴或自研实现。

## 6. 参考资料

- Parity 官方 post-mortem：《The Multi-Sig Hack: A Postmortem》 2017-07-20
- OpenZeppelin blog：《On the Parity Wallet Multisig Hack》2017-07-19
- Phil Daian：《The Parity Wallet Hack Explained》2017-07-22 hackingdistributed.com
- Etherscan verified：WalletLibrary `0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4`
- SlowMist 2017 年报：Parity v1.5 漏洞章节
- rekt.news：Parity 回顾

---

*Last verified: 2026-04-23*
