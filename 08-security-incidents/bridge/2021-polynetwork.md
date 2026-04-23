---
title: Poly Network 跨链桥 keeper 替换攻击（2021-08-10, ~$611M）
module: 08-security-incidents
category: Bridge
date: 2021-08-10
loss_usd: 611000000
chain: [Ethereum, BSC, Polygon]
severity: Tier-1
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://slowmist.medium.com/the-analysis-and-q-a-of-poly-network-being-hacked-8fab4ecbc684
  - https://www.certik.com/resources/blog/2Z6lylDNn3gMqdLFQYFcOA-poly-network-hack-analysis-largest-crypto-hack
  - https://rekt.news/polynetwork-rekt/
  - https://medium.com/poly-network/the-final-statement-on-poly-network-incident-10b41a6202f3
  - https://blog.mudit.blog/post/2021/08/10/poly-network-hack
tx_hashes:
  - 0xad7a2c70c958fcd3effbf374d0acf3774a9257577625ae4c838e24b0de17602a (Ethereum, 核心 verifyHeaderAndExecuteTx 调用)
  - 0xb1f70464bd95b774c6ce60fc706eb5f9e35cb5f06e6cfe7c17dcda46ffd59581 (BSC)
  - 0xc697f495e9d1c2d6bf3e14d40c1de1a33a1a501aab90d3c25fe04e62d63ac0d7 (Polygon)
---

# Poly Network 跨链桥攻击

> **TL;DR**：2021-08-10，多链互操作协议 Poly Network 的 `EthCrossChainManager` 合约被利用——攻击者构造特殊的跨链消息，通过其 `verifyHeaderAndExecuteTx` → `_executeCrossChainTx` 路径，让网桥以自身权限调用 `EthCrossChainData.putCurEpochConPubKeyBytes`，把合法 keeper 公钥替换为攻击者公钥，从而以"合法 keeper" 身份签出提款消息，提走 Ethereum / BSC / Polygon 三条链上总计约 **$611M** 资产。这是当时加密史上单笔金额最大的被盗事件。攻击者在事件发生后与 Poly Network 官方沟通，最终**几乎全额归还**全部资产，整个事件呈现出史无前例的"白帽式结尾"。

## 1. 事件背景

- **项目**：Poly Network 是由 Neo、Ontology、Switcheo 三方发起的跨链协议，支持 Ethereum、BSC、Polygon、Neo、Ontology、Heco 等，事件前 TVL 约 $1B。
- **架构**（关键合约，以太坊侧）：
  - `EthCrossChainManager` (ECCM)：跨链消息路由器；接收来自源链的 header + proof，验证后执行目标链 contract call。
  - `EthCrossChainData` (ECCD)：存储 keeper 公钥、跨链元数据；**owner 是 ECCM**，而不是多签/治理。
  - `LockProxy`：资产锁 / 解锁合约；被 ECCM 通过跨链消息调用以完成用户跨链资产释放。
- **时间轴**：
  - 2021-08-10 09:00 UTC 前后：攻击者在 Ethereum、BSC、Polygon 三链上几乎同步提交构造好的 `verifyHeaderAndExecuteTx` 调用，把 keeper 公钥换成攻击者自己的。
  - 2021-08-10 10:00–12:00 UTC：攻击者以新的 "合法 keeper" 身份发起提款跨链消息，从 `LockProxy` 分批提走三链资产。
  - 2021-08-10 12:30 UTC：SlowMist、PeckShield、rekt 推特同步预警；Poly Network 官方发推确认。
  - 2021-08-11：Poly Network 连续发公开信请求攻击者归还；Tether 冻结约 $33M USDT；Mr. White Hat（攻击者自称）在链上 message field 中与 Poly 沟通。
  - 2021-08-12：攻击者开始归还；2021-08-13：除 USDT（冻结）外基本归还完毕。
  - 2021-08-17：Poly Network 与攻击者通过 multisig 完成全部资产归还；攻击者获得 $500K 的"bug bounty"并声明不会被起诉。
- **发现过程**：SlowMist / PeckShield 链上监控 + rekt 社区，几乎实时。

## 2. 事件影响

- **直接损失**（2021-08-10 快照，分三链）：
  - Ethereum：约 **$273M**（USDC ~$96M、WBTC ~$85M、renBTC、UNI、SHIB、DAI、ETH 等）。
  - BSC：约 **$253M**（BUSD、USDC、BTCB、ETH、BNB 等）。
  - Polygon：约 **$85M**（USDC）。
  - 合计 **~$611M**。
- **受害方**：Poly Network 跨链池（LockProxy）；未直接触及终端用户钱包。
- **连带影响**：
  - 跨链桥从此被视为 Web3 头号高风险组件；业界开始系统性审视 keeper / relayer 架构、权限集中度。
  - 此事件启发 Ronin（2022-03, $624M）、Wormhole（2022-02, $326M）、Nomad（2022-08, $190M）等后续桥事件的媒体关注模式。
  - Tether 冻结行为再次引发"去中心化 vs 紧急冻结"的行业辩论。
- **资金去向**：攻击者未实际洗币；从第二天开始在 Poly Network 公开呼吁后分批原路归还至 Poly 提供的 multisig 地址；仅 ~$33M USDT 被 Tether 冻结，后恢复。

## 3. 技术根因（代码级分析）

### 3.1 关键合约与继承关系

- `EthCrossChainManager`（ECCM）合约地址：`0x838bf9E95CB12Dd76a54C9f9D2E3082EAF928270`（事件版本；事后升级）。
- `EthCrossChainData`（ECCD）合约地址：`0xcF2afe102057bA5c16f899271045a0A37fCb10f2`。
- ECCD `Ownable`，owner 被设置为 ECCM 合约地址——这一设计使 ECCM 可以作为"内部管理员"修改 keeper 公钥；而 ECCM 又允许任意合法跨链消息指定 `_toContract` 和 `_method`，这就构成了提权路径。
- 代码仓库：<https://github.com/polynetwork/eth-contracts>；关键文件 `contracts/core/cross_chain_manager/logic/EthCrossChainManager.sol` 与 `contracts/core/cross_chain_manager/data/EthCrossChainData.sol`。

### 3.2 漏洞函数与关键代码

**`EthCrossChainManager.verifyHeaderAndExecuteTx`**（简化）：

```solidity
function verifyHeaderAndExecuteTx(
    bytes memory proof,
    bytes memory rawHeader,
    bytes memory headerProof,
    bytes memory curRawHeader,
    bytes memory headerSig
) public whenNotPaused returns (bool) {
    ...
    // 1) 用当前 keeper 集合验证 header 签名
    require(_verifySigWithOrder(rawHeader, headerSig, keepers), "...");

    // 2) 把 proof 的 value 解为 ToMerkleValue（跨链调用描述）
    ToMerkleValue memory toMerkleValue = _deserializeMerkleValue(toMerkleValueBs);

    // 3) 执行目标合约调用
    require(
        _executeCrossChainTx(
            toMerkleValue.makeTxParam.toContract,        // ← 攻击者指定为 ECCD
            toMerkleValue.makeTxParam.method,            // ← 攻击者指定为 putCurEpochConPubKeyBytes 的 selector 对应方法
            toMerkleValue.makeTxParam.args,              // ← 攻击者的公钥
            toMerkleValue.makeTxParam.fromContract,
            toMerkleValue.fromChainID
        ),
        "execute cross chain tx failed!"
    );
    return true;
}
```

**`_executeCrossChainTx`**：

```solidity
function _executeCrossChainTx(
    bytes memory _toContract,
    bytes memory _method,
    bytes memory _args,
    bytes memory _fromContractAddr,
    uint64 _fromChainId
) internal returns (bool) {
    address toContract = Utils.bytesToAddress(_toContract);

    // 关键：构造 selector = keccak256(abi.encodePacked(_method, "(bytes,bytes,uint64)"))[:4]
    (bool success, bytes memory returnData) = toContract.call(
        abi.encodePacked(
            bytes4(keccak256(abi.encodePacked(_method, "(bytes,bytes,uint64)"))),
            abi.encode(_args, _fromContractAddr, _fromChainId)
        )
    );
    require(success == true, "EthCrossChain call business contract failed");
    ...
}
```

**`EthCrossChainData.putCurEpochConPubKeyBytes`**：

```solidity
function putCurEpochConPubKeyBytes(bytes memory curEpochPkBytes) public onlyOwner returns (bool) {
    ConKeepersPkBytes = curEpochPkBytes;
    return true;
}
```

**缺陷核心**：
1. ECCD.owner == ECCM：任何能让 ECCM 发起对 ECCD 的 call，就能绕过 onlyOwner。
2. `_executeCrossChainTx` 对 `_toContract` 与 `_method` 没有任何白名单：攻击者在跨链消息里可指定 `toContract = ECCD` 并且 `_method = "f1121318093"`（一段精心构造的字符串），使得 `keccak256("f1121318093(bytes,bytes,uint64)")` 的前 4 字节**恰好等于** `putCurEpochConPubKeyBytes(bytes)` 的 selector `0x41973cd9`（攻击者通过暴力枚举得到）。
3. 因此 ECCM 以自己的身份 call 了 ECCD.`putCurEpochConPubKeyBytes`，把 `ConKeepersPkBytes` 替换为攻击者控制的公钥。

### 3.3 攻击步骤

1. **离线 selector 暴力搜索**：寻找一个字符串 `M` 使得 `keccak256(M + "(bytes,bytes,uint64)")[:4] == 0x41973cd9`（`putCurEpochConPubKeyBytes(bytes)` 的 selector）。攻击者最终选用了 `f1121318093`。
2. **构造跨链消息**：
   - `toContract` = ECCD 地址
   - `method` = "f1121318093"
   - `args` = ABI 编码的攻击者公钥
3. **伪造合法 header**：由于攻击者彼时是链下 relayer 的自由构造方（Poly 的 keeper 集合对应的 header 签名是可由其他链产生并中继过来），攻击者提交了一个由合法旧 keeper 签名的 header，其中 Merkle 证明指向上述 toMerkleValue；Poly Network 对"跨链消息可被任意构造并提交"的防护仅依赖 header 签名，而攻击者可能利用了跨链消息的签名漏洞（也有分析指出 Poly 的 keeper 私钥对应已被泄露，但 SlowMist / Mudit.Blog 多数分析倾向"消息构造"路径，非私钥泄露）。
4. **发送 verifyHeaderAndExecuteTx**：ECCM 验过签名 → 执行 `_executeCrossChainTx` → 调用 `ECCD.putCurEpochConPubKeyBytes` → keeper 公钥被替换。关键 tx `0xad7a2c70c958fcd3effbf374d0acf3774a9257577625ae4c838e24b0de17602a`。
5. **以新 keeper 身份发起提款跨链消息**：此时攻击者就是"唯一合法 keeper"，可以签任意跨链消息。依次调用 `verifyHeaderAndExecuteTx` 让 ECCM 调用 `LockProxy.unlock` 把资产放给攻击者地址。
6. **三链同步操作**：Ethereum / BSC / Polygon 上 ECCM 架构一致，攻击者在几小时内完成三链清仓。关键 tx `0xb1f70464bd95b774c6ce60fc706eb5f9e35cb5f06e6cfe7c17dcda46ffd59581`（BSC）、`0xc697f495e9d1c2d6bf3e14d40c1de1a33a1a501aab90d3c25fe04e62d63ac0d7`（Polygon）。

### 3.4 为何审计未发现

- Poly Network 在事件前有一份由 NCC Group 完成的审计，但该审计聚焦于加密原语与跨链消息签名流程，未充分检查 ECCM → ECCD 的"owner 传染"问题。
- `_executeCrossChainTx` 的"method 名字可自由指定，selector 可被暴力碰撞"在当时的审计 checklist 中并非常规项；本案之后，"任何跨链桥消息的 toContract + method 必须有白名单"成为行业共识。
- Mudit Gupta（Polygon）在 2021-08-10 同日博客中最早公开给出根因定位，被业内引为经典分析。

## 4. 事后响应

- **项目方**：
  - Poly Network 2021-08-10 发公开信请求归还，2021-08-11 将攻击者称为 "Mr. White Hat"，邀请其成为首席安全顾问。
  - 2021-08-12–17 陆续与攻击者协商多签；最终成立 3-of-4 multisig（Poly + 攻击者双持有）逐笔赎回被盗资产。
  - 全部资产于 08-23 前归还（USDT 部分因 Tether 冻结经过解锁）；Poly Network 宣布给予攻击者 $500K 赏金。
- **合约修复**：升级 ECCM / ECCD，加入：
  - `toContract` 白名单，只允许调用指定的 `LockProxy`。
  - 删除 `putCurEpochConPubKeyBytes` 可被跨链消息触达的路径，改为多签治理。
  - ECCD 的 owner 从 ECCM 改为独立的 Timelock Multisig。
- **法律 / 执法**：Poly 未提起诉讼；SlowMist 公开声明已通过邮箱 + IP + 交易所 KYC 锁定攻击者身份，迫使其归还。Chainalysis 未在本案中做国家级归因，**归因未公开**（攻击者自述是"白帽个人")。
- **行业连锁**：跨链桥审计范式重构——Halborn、Trail of Bits、OpenZeppelin 陆续发布"跨链桥 threat model"指南；但 2022 年 Ronin / Wormhole / Nomad 再次出事，说明架构级问题未被彻底解决。

## 5. 启发与教训

- **开发者 / 协议设计（核心）**：
  - **最小权限**：跨链消息执行器（ECCM 级别的合约）绝不能对关键治理合约（ECCD 级别）拥有 owner 权限；应彻底解耦"消息路由"与"治理"。
  - **目标合约与方法白名单**：跨链桥的 `_executeCrossChainTx` 必须对 `(toContract, method)` 有严格 allowlist；否则 selector 碰撞是可预期风险。
  - **selector 碰撞防御**：永远不要用字符串 `method name` 动态拼接 selector；如果必须，要做 allowlist + length 验证；更好的做法是直接 encode 完整的 `(selector, args)`。
  - **Keeper / Validator 权限变更**：任何能改写验证者集合的函数必须：1) 被 Timelock 保护；2) 有社会层多签；3) 脱离跨链消息路径。
- **审计方**：
  - 跨链桥审计必须用**完整 threat model**：从源链中继 → 链下 relayer → 目标链执行，每一跳的可信假设都要明确。
  - Fuzzing：对 `_executeCrossChainTx` 随机生成 method 字符串，检验能否命中敏感 selector（此后 Halborn 有专用规则库）。
  - 不变量：`onlyOwner` 函数被调用时，`msg.sender` 是否属于受控集合？若 owner 是合约，必须穿透到该合约可被外部如何触达。
- **用户**：跨链桥类协议 TVL 越大，风险越集中；大额跨链应分批、使用多个独立桥或等待 timelock。
- **行业**：
  - 本事件奠定了"非合约语境下的白帽谈判"样板；但也暴露出"仅靠道德劝返"在未来大规模事件中并不可靠（Ronin、Nomad 均未归还或仅部分归还）。
  - 跨链互操作性方案此后分化为：乐观型（Optimism Bridge）、ZK 型（Succinct、zkBridge）、多 Relayer 去中心化（Axelar、Wormhole V2）、原生 IBC（Cosmos），每条路径都把 Poly Network 事件作为设计基准反例。

## 6. 参考资料

- SlowMist 分析：<https://slowmist.medium.com/the-analysis-and-q-a-of-poly-network-being-hacked-8fab4ecbc684>
- CertiK 分析：<https://www.certik.com/resources/blog/2Z6lylDNn3gMqdLFQYFcOA-poly-network-hack-analysis-largest-crypto-hack>
- rekt.news：<https://rekt.news/polynetwork-rekt/>
- Poly Network 官方最终声明：<https://medium.com/poly-network/the-final-statement-on-poly-network-incident-10b41a6202f3>
- Mudit Gupta 根因分析（最早公开）：<https://blog.mudit.blog/post/2021/08/10/poly-network-hack>
- Poly Network 合约仓库：<https://github.com/polynetwork/eth-contracts>
- 关键 tx（Ethereum）：<https://etherscan.io/tx/0xad7a2c70c958fcd3effbf374d0acf3774a9257577625ae4c838e24b0de17602a>
- 关键 tx（BSC）：<https://bscscan.com/tx/0xb1f70464bd95b774c6ce60fc706eb5f9e35cb5f06e6cfe7c17dcda46ffd59581>
- 关键 tx（Polygon）：<https://polygonscan.com/tx/0xc697f495e9d1c2d6bf3e14d40c1de1a33a1a501aab90d3c25fe04e62d63ac0d7>
- ECCM 合约：<https://etherscan.io/address/0x838bf9E95CB12Dd76a54C9f9D2E3082EAF928270>
- ECCD 合约：<https://etherscan.io/address/0xcF2afe102057bA5c16f899271045a0A37fCb10f2>

---

*Last verified: 2026-04-23*
