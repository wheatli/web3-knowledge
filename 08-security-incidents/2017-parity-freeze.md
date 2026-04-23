---
title: Parity Multisig 2.0 库合约被 devops199 自杀导致 513,774 ETH 永久冻结（2017-11-06, ~$150M 按当时价）
module: 08-security-incidents
category: Wallet
date: 2017-11-06
loss_usd: 150000000
chain: [Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.parity.io/a-postmortem-on-the-parity-multi-sig-library-self-destruct/
  - https://medium.com/chain-cloud-company-blog/parity-multisig-hack-again-b46771eaa838
  - https://github.com/paritytech/parity/issues/6995
  - https://etherscan.io/address/0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4
  - https://hackingdistributed.com/2017/11/08/parity-wallet-again/
tx_hashes:
  - devops199 初始化 library：0x05f71e1b2cb4f03e547739db15d080fd30c989eda04d37ce6264c5686e0722c9
  - devops199 调用 kill() 自杀：0x47f7cff7a5e671884629c93b368cb18f58a993f4b19c2a53a8662e3f1482f690
  - 受害钱包之一（Polkadot 众筹 Web3 Foundation）：0x3bfc20f0b9afcace800d73d2191166ff16540258
---

# Parity Multisig 2.0 库合约自杀冻结 513k ETH

> **TL;DR**：2017 年 11 月 6 日，一名自称 `devops199` 的 GitHub 用户在尝试学习 Solidity 过程中，对 Parity Multisig 2.0 版本的 library 合约（`0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4`）调用了 `initWallet` 将自己设为 owner，随后调用 `kill()` 执行 `selfdestruct`，导致所有依赖该 library 的多签钱包（约 587 个，含 Polkadot 主网众筹合约）在一夜之间无法执行任何 `delegatecall` 到已被销毁的库，共计 513,774.16 ETH 永久冻结。按当时 ~$291 估计约 $150M，按 2021 峰值估算缺口名义上超 $2.5B。至今该资金未被恢复；Parity 曾推 EIP-999 尝试通过硬分叉恢复，被以太坊社区投票否决。

## 1. 事件背景

- **主体**：
  - Parity Multisig 2.0：在 2017-07-19 v1.5 事件（见 `2017-parity-wallet-hack.md`）后 Parity 推出的修复版本，代码结构仍为"轻量 Wallet proxy + 共享 WalletLibrary"。
  - Polkadot / Web3 Foundation：Parity 为 Polkadot 进行的 ICO 多签钱包使用此架构，持有众筹 ETH。
- **时间轴**：
  - 2017-07-20：Parity 发布 1.5 事件修复 + 新库 2.0，新库地址 `0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4`。
  - 2017-11-06 15:33 UTC：devops199 向库合约本身（而非 proxy）发送 tx 调用 `initWallet([devops199], 1, ...)`，把自己设为该 library 合约上的 sole owner。
  - 2017-11-06 15:45 UTC：devops199 调用 library 合约的 `kill()` 函数，selfdestruct 触发，合约代码被从链上清除。
  - 2017-11-06 晚：社区开始发现多个 Parity 多签钱包转账失败，几小时内定位到问题。
  - 2017-11-07：Parity 官方博客公告事件。
  - 2017-12：Parity 发起社区讨论是否硬分叉恢复。
  - 2018-04：EIP-999（Jutta Steiner 提交）社区投票 ~55% 反对，不予激活。
- **发现过程**：多签钱包 owner 尝试交易被 EVM 在 `CALLCODE`/`DELEGATECALL` 到空地址时 revert；hackingdistributed 与多个 ICO 项目几乎同时发推求助。

## 2. 事件影响

- **直接损失**：513,774.16 ETH 永久冻结。按 2017-11-06 ~$291 估算约 $150M；按 2018 初高点 $1,400 估算缺口约 $720M；按 2021 峰值 $4,800 估算约 $2.46B。
- **受害方**：
  - Web3 Foundation / Polkadot ICO 钱包：306,276 ETH（最大单笔受害）。
  - Musiconomi、Æternity、Iconomi 等项目多签。
  - 587 个钱包共 ~573 名 owner 用户。
- **连带影响**：
  - 激起 EIP-999（"Restore Contract Code at 0x863DF…"）社区投票，最终被以太坊社区以"不可篡改"原则否决。
  - Web3 Foundation 被迫后续用私募轮补齐 Polkadot 开发资金。
  - `selfdestruct` opcode 的安全争议导致最终 EIP-6049（2023 弃用建议）与 EIP-4758（2022 提案将 selfdestruct 退化）陆续提出。
- **资金去向**：ETH 停在原 proxy 钱包的 storage 中，链上可见余额但无任何合法签名能将其取出。

## 3. 技术根因

- **漏洞分类**：Access-Control + selfdestruct + Library/Proxy Pattern Misuse（1.5 版本漏洞的同源变种）。
- **受损合约**：Parity Multisig 2.0 WalletLibrary，部署地址 `0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4`。
- **关键代码片段**（从 Parity issue #6995 与 Etherscan 归档还原）：

```solidity
// WalletLibrary 2.0 (修复 1.5 之后的新版本)
contract WalletLibrary is WalletEvents {

    // 修复点：增加 only_uninitialized 防止被重复初始化
    modifier only_uninitialized { if (m_numOwners > 0) throw; _; }

    // ====== 真正的错误在这里 ======
    // 修复了"重复初始化"，但没修复"直接对 library 合约调用"
    // 于是攻击者对 library 本身首次调用 initWallet，
    // 因 library 的 storage 中 m_numOwners 未设置，只要一次 init 就获得 owner 权限
    function initWallet(address[] _owners, uint _required, uint _daylimit)
        only_uninitialized {
        initDaylimit(_daylimit);
        initMultiowned(_owners, _required);
    }

    // kill 函数存在且只需 owner 就能调用
    function kill(address _to) onlymanyowners(sha3(msg.data)) external {
        suicide(_to);    // = selfdestruct(_to)
    }
}
```

关键的设计假设是："library 合约不会被直接调用，用户只会通过 proxy 的 delegatecall 调用它"。但这个假设**没有在合约中被强制**：

- Library 合约是一个普通合约，任何账号都可以向它发 tx。
- 一旦对 library 自身调用 `initWallet`，library 的 storage（不是 proxy 的！）中 `m_numOwners` 从 0 改为 N，owners 被设置为调用者。
- Library 合约此时有了 owner，且含 `kill()`，owner 可调用 `selfdestruct` 将 library 字节码清零。
- 所有 proxy 合约仍然保留指向该地址的 `_walletLibrary`，但 `delegatecall` 到已被销毁的合约会：在 EVM 中被当作"对无代码账户的 call"，除了转移 ETH 外无任何副作用，导致所有除 fallback 外的函数全部失效。
- Proxy 钱包中的 ETH 余额仍在 proxy storage，但没有任何函数能再执行提款。

- **攻击（事故）步骤**：
  1. tx `0x05f71e...22c9`：devops199 调用 `WalletLibrary.initWallet([devops199], 1, 0)`，把自己设为 library 的 sole owner。
  2. tx `0x47f7cf...2f690`：devops199 调用 `WalletLibrary.kill(devops199)`，合约 selfdestruct，字节码被清除并将 library 地址上原有 ETH（若有）发给 devops199。
  3. 所有 proxy 钱包此后调用任何函数（除 fallback 收款外）都会 revert。
- **为何审计未发现**：
  - v1.5 事件后 Parity 只增加 `only_uninitialized` 防重入式初始化，但未禁止"直接调用 library 合约"。
  - 没有在 library 构造时执行一次性自我初始化来封锁 owner 字段（即 OpenZeppelin 后续的 `_disableInitializers()` 模式）。
  - 审计方与 Parity 都对"library 本身作为独立部署合约"的攻击面认识不足。

## 4. 事后响应

- **项目方**：
  - 2017-11-07 发布 post-mortem，确认 ETH 永久冻结。
  - Parity 提出 EIP-156 / EIP-999 等多种恢复方案；EIP-999 主张通过硬分叉恢复该 library 地址的字节码。
  - 2018-04 社区 carbonvote 投票 ~55% 反对，最终方案被撤回。
- **资产追回**：无。
- **执法**：devops199 自称无恶意，且行为发生在被遗弃的 library 合约上，非盗窃，无人起诉；部分社区认为此账户实则蓄意，但无 FBI / Chainalysis 正式归因报告。
- **行业连锁**：
  - OpenZeppelin `Initializable` 正式添加 `_disableInitializers()`，所有 implementation 合约部署时必须自我 lock。
  - EIP-4758（2022）、EIP-6049（2023）讨论弃用 `SELFDESTRUCT`，最终在 Cancun 升级（2024-03）通过 EIP-6780 将 `SELFDESTRUCT` 语义改为"仅转账，不清除代码"，历史上该语义在 DeFi 与 L2 proxy 中带来的"不可恢复冻结"风险从此闭环。

## 5. 启发与教训

- **对开发者**：
  - **所有 implementation / library 合约在部署后必须主动 lock**：构造函数里调用 `_disableInitializers()` 或显式 `initialize(address(0), ...)`。
  - 不要在 library 里写 `selfdestruct`（或任何可以改变其字节码状态的逻辑）。
  - Cancun 升级后 `SELFDESTRUCT` 不再清除代码，但历史合约的风险仍需治理。
- **对审计方**：
  - 审计 proxy/library 架构必须显式验证 implementation 的直接可调用面。
  - Slither `uninitialized-state` / `suicidal` 规则应强制启用。
- **对协议治理**：
  - 该事件 + EIP-999 社区否决，塑造了以太坊"不可篡改"的社会共识。
  - Polkadot 团队的财务损失直接影响了其 2018 年融资节奏，反面证明项目方金库不应 100% 托管在任何单一智能合约。
- **对用户**：钱包合约选择应有充分历史（至少 1 年实战 + 多次审计 + Bug Bounty），Gnosis Safe 后续成为事实标准。

## 6. 参考资料

- Parity 官方 post-mortem：《A Postmortem on the Parity Multi-Sig Library Self-Destruct》2017-11-15
- Parity GitHub Issue #6995：devops199 事件技术讨论
- Phil Daian：《Parity Wallet, Again》2017-11-08 hackingdistributed.com
- Medium (Chain Cloud)：《Parity MultiSig Hack (again)》2017-11
- EIP-999 讨论：ethereum/EIPs#999
- Etherscan verified：WalletLibrary `0x863df6bfa4469f3ead0be8f9f2aae51c91a907b4`（字节码已为空）
- OpenZeppelin blog：《Initializable and The Parity Hack》系列
- SlowMist 2017 年度报告：Parity 2.0 章节

---

*Last verified: 2026-04-23*
