---
title: The DAO 重入攻击（2016-06-17, ~$60M 按当时价 / ~$150M 峰值估）
module: 08-security-incidents
category: DeFi
date: 2016-06-17
loss_usd: 60000000
chain: [Ethereum]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://blog.ethereum.org/2016/06/17/critical-update-re-dao-vulnerability
  - https://www.coindesk.com/learn/understanding-the-dao-attack
  - https://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/
  - https://blog.slock.it/the-history-of-the-dao-and-lessons-learned-d06740f8cfa5
  - https://github.com/slockit/DAO/blob/develop/DAO.sol
tx_hashes:
  - 首次攻击 tx：0x0ec3f2488a93839524add10ea229e773f6bc891b4eb4794c3337d4495263790b
  - 攻击者合约：0x304a554a310C7e546dfe434669C62820b7D83490
  - 子 DAO 地址（DarkDAO）：0x304a554a310C7e546dfe434669C62820b7D83490
---

# The DAO 重入攻击

> **TL;DR**：2016 年 6 月 17 日，基于以太坊的明星众筹项目 The DAO（募资总额达 ETH 总供应量 14%，~$150M）遭遇重入攻击，攻击者利用 `splitDAO` 函数中"先外部调用后状态更新"的违反 Checks-Effects-Interactions 模式，以递归调用在单笔交易内反复提取份额，共抽走约 3,641,694 ETH（~$60M 按当时价）。事件引发以太坊社区激烈分歧，最终 2016-07-20 硬分叉回滚攻击，新链沿用 ETH 名字，拒绝分叉的原链成为 ETC（Ethereum Classic）。这是 Web3 历史上第一次 Tier-1 级智能合约漏洞事件，也是"代码即法律"叙事被现实击碎的分水岭。

## 1. 事件背景

- **主体**：The DAO 是 Slock.it 团队于 2016-04-30 推出的去中心化自治组织，以 ERC-20 形式的 DAO 代币承载投票权，旨在以去中心化 VC 方式投资 DApp。众筹于 2016-05-28 结束，共募集 12,072,933 ETH（占当时 ETH 流通量 14%，约 $150M）。
- **时间轴**：
  - 2016-05-28：众筹结束。
  - 2016-06-05：Cornell 教授 Emin Gün Sirer 等在 "A Call for a Temporary Moratorium on The DAO" 博客中警示"递归调用攻击"风险；Slock.it 回应称已有修复计划。
  - 2016-06-09：Peter Vessenes 博客《More Ethereum Attacks: Race-To-Empty》公开描述 race condition。
  - 2016-06-17 03:34 UTC：攻击开始，攻击者部署 "DarkDAO" 子 DAO 合约（地址 `0x304a554a...3490`）。
  - 2016-06-17 03:34–10:00 UTC：递归调用持续发生 258 次 `splitDAO`，累计抽走 3,641,694 ETH。
  - 2016-06-17 13:00 UTC：Vitalik Buterin 在 ethereum.org blog 发布 "Critical Update Re: DAO Vulnerability"。
  - 2016-06-18：Robin Hood Group（白帽）以相同手法抢救部分 ETH，复制攻击合约以 "counterattack" 将剩余 DAO 资金转入受控 child DAO。
  - 2016-07-20：以太坊主网在区块 1,920,000 硬分叉，将 DarkDAO 与 WithdrawDAO 余额一次性迁移至 WithdrawContract，允许原 DAO token holders 1:100 赎回 ETH。
  - 2016-07-20：不接受回滚的节点维持原链，命名为 Ethereum Classic（ETC）。
- **发现过程**：攻击由 Griff Green（Slock.it）与社区成员在钱包地址异常增长时发现，几小时内即完成归因。

## 2. 事件影响

- **直接损失**：3,641,694 ETH（~$60M 按 2016-06-17 ~$20.5 估；若按 2021 峰值 ~$4,800 则名义缺口达 $17.5B）。
- **受害方**：The DAO 的 ~11,000 个投资地址；Slock.it 公司。
- **连带影响**：
  - 硬分叉永久分裂以太坊社区，诞生 ETC。
  - ETH 价格从 ~$20 跌至 ~$10（约 50% 回撤）后于数月内修复。
  - 重入攻击成为 Solidity 审计第一优先项；OpenZeppelin `ReentrancyGuard`、Checks-Effects-Interactions 模式成为编程范式。
  - 监管层开始关注 DAO 代币证券属性，美国 SEC 2017-07 发布 "DAO Report"，认定 DAO token 属证券。
- **资金去向**：
  - 大部分被迁入 WithdrawContract `0xbf4ed7b27f1d666546e30d74d50d173d20bca754`，投资者按 1 ETH = 100 DAO 比例赎回。
  - 未迁移的 ETC 链上 DAO 余额由黑客最终转出并部分在 Shapeshift 等服务出金。
  - 归因未公开（ETC 侧攻击者至今未被识别）。

## 3. 技术根因（核心章节）

- **漏洞分类**：Reentrancy（重入） + Checks-Effects-Interactions 反模式 + 调用控制权（call semantics）。
- **受损合约**：`DAO.sol` @ `slockit/DAO` GitHub 仓库，主网部署地址 `0xbb9bc244d798123fde783fcc1c72d3bb8c189413`。
- **关键代码片段**（简化自 `DAO.sol` 的 `splitDAO` 函数，标注错在哪一行）：

```solidity
// DAO.sol @ slockit/DAO (v1.0, 部署于 2016-04-30)
// 完整源码见 GitHub: slockit/DAO/blob/develop/DAO.sol

function splitDAO(
    uint _proposalID,
    address _newCurator
) noEther onlyTokenholders returns (bool _success) {

    Proposal p = proposals[_proposalID];
    // ... 检查提案有效性、签名 ...

    // (1) 创建新的 child DAO（本例即 DarkDAO）
    DAO newDAO = createNewDAO(_newCurator);

    // (2) 计算按比例应迁移的 ETH
    uint fundsToBeMoved =
        (balances[msg.sender] * p.splitData[0].splitBalance) /
        p.splitData[0].totalSupply;

    // ====== 问题起点 ======
    // (3) 这里向 newDAO 转账 ETH —— 会触发 newDAO 的 fallback 回调
    //     关键错误：在外部调用之后才清空余额
    if (newDAO.createTokenProxy.value(fundsToBeMoved)(msg.sender) == false)
        throw;

    // (4) *直到此时* 才扣减 msg.sender 的 DAO 余额
    //     —— 攻击者在 (3) 的回调中再次调用 splitDAO，
    //        此时 balances[msg.sender] 尚未清零，重复提款成立。
    Transfer(msg.sender, 0, balances[msg.sender]);
    withdrawRewardFor(msg.sender);
    totalSupply -= balances[msg.sender];
    balances[msg.sender] = 0;    // <-- 应该在 (3) 之前执行！
    paidOut[msg.sender] = 0;

    return true;
}
```

以及：

```solidity
// withdrawRewardFor 也存在类似递归问题
function withdrawRewardFor(address _account) noEther internal returns (bool _success) {
    if ((balanceOf(_account) * rewardAccount.accumulatedInput()) /
        totalSupply < paidOut[_account]) throw;

    uint reward =
        (balanceOf(_account) * rewardAccount.accumulatedInput()) /
        totalSupply - paidOut[_account];

    // 外部 call，攻击者 fallback 可再次进入
    if (!rewardAccount.payOut(_account, reward)) throw;

    paidOut[_account] += reward;   // <-- 状态更新晚于外部调用
    return true;
}
```

- **攻击步骤分解**：
  1. 攻击者部署 "DarkDAO" 合约，该合约的 fallback / default 函数重新调用 `The DAO.splitDAO` 指向同一提案。
  2. 攻击者在 The DAO 里以小量 DAO token 投票同意 DarkDAO 分裂提案（提案于 debating period 前创建）。
  3. 首次调用 `splitDAO(proposalId, darkDAO)`：
     - `fundsToBeMoved` 计算完毕。
     - 向 DarkDAO 转 ETH，触发 DarkDAO 的 fallback。
     - DarkDAO fallback 再次 `splitDAO(...)`，此时 `balances[attacker]` 尚未清零。
     - 递归 ~30 层后 EVM 栈或 gas 限制回落，开始退栈，每层均成功抽取 ETH。
  4. 每次完整 tx 可提取相当于余额约 30 倍的 ETH。
  5. 攻击共执行约 258 次类似 tx，历时 ~6 小时，累计 3.64M ETH。
- **关键 tx**：`0x0ec3f2488a93839524add10ea229e773f6bc891b4eb4794c3337d4495263790b`（首笔）；完整列表见 Etherscan 攻击者地址。
- **为何审计未发现**：
  - The DAO 上线前并未经历现代意义的完整安全审计；由社区成员提交 PR 并由 Slock.it 团队合并。
  - Vessenes 与 Sirer 6 月初即警告"race-to-empty"风险，Slock.it 公开回复称 withdrawRewardFor 的同类问题已"不可利用"，判断失误。
  - Solidity 0.3.x 时代尚无 ReentrancyGuard / Checks-Effects-Interactions 编程范式普及。

## 4. 事后响应

- **项目方 / 社区**：
  - 2016-06-17：Vitalik 博客呼吁暂停 DarkDAO 出金（该 DAO 代码要求 27 天 debating period 后才能提币，为响应争取时间）。
  - 2016-06-24：软分叉提案失败（因 DoS 漏洞被撤回）。
  - 2016-07-20：硬分叉在区块 1,920,000 执行，将 DarkDAO / ExtraBalance / WithdrawDAO 余额迁移到 WithdrawContract。
- **Robin Hood Group**：Alex Van de Sande、Griff Green 等在主网与 ETC 上均做了白帽抢救行动，部分 ETH 被 RHG 合约保护。
- **资产追回 / 赔付**：ETH 链上原投资者可按 1 DAO = 0.0001 ETH 比例在 WithdrawContract 提款；ETC 链上大部分 DAO 资金由原攻击者控制。
- **法律 / 执法**：
  - 美国 SEC 于 2017-07 发布 "Report of Investigation: The DAO"，明确 DAO token 属于证券，但对该案未提起诉讼。
  - ETC 链攻击者至今**归因未公开**（曾有社区指称 Toby Hoenisch，但本人否认，Chainalysis/FBI 无正式报告）。
- **行业连锁**：
  - OpenZeppelin 2016-2017 推出 `ReentrancyGuard` modifier；
  - Solidity 0.4.x 增加 `transfer()`/`send()` 默认 2300 gas 限制（虽非根治，但大幅缩小攻击面）。
  - Checks-Effects-Interactions 被正式写入 Solidity 官方文档。

## 5. 启发与教训

- **对开发者**：
  - **Checks-Effects-Interactions 模式**：先校验，再改状态，最后再做外部调用。
  - 对外部调用使用 `ReentrancyGuard` 或 mutex 锁。
  - 使用 `transfer()` 或显式 `call{value:...}("")` 并严格控制 gas。
  - 谨慎 fallback：合约接收 ETH 即执行逻辑，等同允许未知代码运行。
- **对审计方**：
  - 必须系统检查所有外部调用前后的状态一致性。
  - Slither 的 `reentrancy-eth` / `reentrancy-no-eth` 规则应作为默认检测项。
  - Fuzz / invariant testing：余额不变量 `sum(balances) == totalSupply` 需常态测试。
- **对用户 / 投资者**：
  - 未经审计的高 TVL 合约具备不可忽视的尾部风险。
  - 治理合约的时间锁（debating period）在关键时刻是救命绳。
- **对协议 / 生态**：
  - 该事件证明"不可篡改"在极端情况下可被社区共识推翻，但代价是链分裂。
  - 催生了 EIP-2929（Berlin, 2021）降低了一些 storage 访问 gas，但也让 2300 gas 的 `transfer()` 理论上可能失败，后 Solidity 官方建议改用 `call` + reentrancy guard。

## 6. 参考资料

- Vitalik Buterin：《Critical Update Re: DAO Vulnerability》ethereum.org blog, 2016-06-17
- Phil Daian：《Analysis of the DAO exploit》 hackingdistributed.com, 2016-06-18
- Emin Gün Sirer, Dino Mark, Vlad Zamfir：《A Call for a Temporary Moratorium on The DAO》2016-05-27
- Peter Vessenes：《More Ethereum Attacks: Race-To-Empty》2016-06-09
- SlowMist 历史回顾：《The DAO 事件技术复盘》（2019 年安全年报）
- rekt.news：《The DAO – Rekt》回顾帖
- U.S. SEC Report of Investigation No. 81207：The DAO Tokens（2017-07-25）
- GitHub 源码：`slockit/DAO` (commit 8b2e87f) `DAO.sol`
- Etherscan：攻击合约 `0x304a554a310C7e546dfe434669C62820b7D83490`

---

*Last verified: 2026-04-23*
