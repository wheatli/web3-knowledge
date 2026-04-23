---
title: UTXO 账本模型（Bitcoin & EUTXO）
module: 01-infrastructure/ledger-model
priority: P0
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://bitcoin.org/bitcoin.pdf
  - https://github.com/bitcoin/bitcoin
  - https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
  - https://docs.cardano.org/learn/eutxo-explainer
  - https://iohk.io/en/research/library/papers/the-extended-utxo-model/
  - https://developer.bitcoin.org/reference/transactions.html
---

# UTXO 账本模型（Bitcoin & EUTXO）

> **TL;DR**：UTXO（Unspent Transaction Output）把链状态表达为一组未花费输出的集合，每笔交易消费旧 UTXO、产生新 UTXO。与 Account 模型相比，UTXO 天然并行、隐私更好、状态无关、适合 ZK-friendly 设计，但编写合约更繁琐。Bitcoin、Litecoin、Zcash 使用经典 UTXO；Cardano 的 EUTXO 扩展了可验证 datum，使"合约级 UTXO"成为可能；Fuel、Sui 的 object 模型也来自 UTXO 家族。

## 1. 背景与动机

Satoshi 在 2008 年设计 Bitcoin 时选择 UTXO 有几个原因：
- 原始"支票 + 余额"心智模型：每笔 tx 的 input 引用之前的 output，相当于 endorsement；
- SPV / Merkle 证明友好：只验证路径而非全局状态；
- 并行性：无冲突的 tx 可并发验证；
- 隐私：地址可一次性使用，天然混淆余额。

但 UTXO 对 **状态共享的智能合约** 不友好：同一合约余额在不同 UTXO 碎片中，交易必须原子收集。以太坊 2014 选择 Account 模型正是为了合约便利性。Cardano 2020 的 EUTXO 证明 UTXO 也能支持富表达智能合约，只是开发思维不同。

Web3 场景：
- 公链：Bitcoin、Litecoin、Bitcoin Cash、Dogecoin。
- 隐私：Zcash、Monero（后者是 UTXO 变体 + ring）。
- 高吞吐：Cardano EUTXO、Fuel (VM-UTXO)、Sui (object-based, UTXO-like)。
- Rollup：Zcash、Namada、Aleo 使用 note-based（变体 UTXO）。

## 2. 核心原理

### 2.1 形式化定义

令 UTXO 集合为 $U \subseteq \mathrm{ID} \times \mathrm{Value} \times \mathrm{Script}$，其中：
- $\mathrm{ID} = (\mathrm{txid}, \mathrm{index})$ 唯一标识。
- $\mathrm{Value} \in \mathbb{N}$ satoshi。
- $\mathrm{Script}$ 是锁定脚本（Bitcoin Script, P2PKH, P2WSH, Taproot...）。

状态转移函数 $\delta: U \times T \to U$：
$$\delta(U, \tau) = (U \setminus \{\mathrm{inputs}(\tau)\}) \cup \mathrm{outputs}(\tau), \quad \text{iff } \mathrm{valid}(\tau, U).$$

$\mathrm{valid}$ 要求：
1. $\mathrm{inputs}(\tau) \subseteq U$（UTXO 存在）。
2. $\sum v_{\mathrm{in}} \ge \sum v_{\mathrm{out}}$（守恒，差为 fee）。
3. 每 input 的 unlock script 满足对应 output 的 lock script。
4. 无重复 input（防双花）。

**不变式**：总 supply $M + \sum v_{\mathrm{in}} - \sum v_{\mathrm{out}}$ 守恒（M = coinbase 增量）。

### 2.2 Bitcoin Script

Bitcoin Script 是栈式、图灵不完备语言，典型脚本：
- **P2PKH**：`OP_DUP OP_HASH160 <PubKHash> OP_EQUALVERIFY OP_CHECKSIG`。
- **P2WPKH (SegWit)**：witness 段承载签名，tx hash 可减 witness 实现 malleability 修复。
- **P2TR (Taproot, BIP-340/341/342)**：Schnorr + 可切换 Script tree。

这层设计保证 **结算安全** 而非可编程：Turing-incomplete + 无持久状态 → 不会挖矿死循环或状态 explosion。

### 2.3 EUTXO（Cardano 扩展）

EUTXO 把 UTXO 扩展为 $(\mathrm{Address}, \mathrm{Value}, \mathrm{Datum}, \mathrm{Script})$。
- **Datum**：附着在 UTXO 的任意数据（类型化）。
- **Redeemer**：花费时提供的参数。
- **Validator Script (Plutus)**：图灵完备，以 $(datum, redeemer, context)$ 为输入决定是否允许花费。

这样合约 **无全局状态**，每个 UTXO 自带上下文；tx 必须原子提供所需输入 + 合约输出。

### 2.4 Bitcoin 子机制拆解

- **Coinbase**：每块第一笔 tx，从无到有产生新 BTC + 收取 fee。
- **Mempool**：候选 tx 池，按 feerate 排序。
- **Merkle Tree**：tx 哈希组成 Merkle 根，写入 block header。
- **Script Execution**：P2SH 允许把复杂脚本折成 hash，花费时公开，节省链上空间。
- **SegWit**：witness 与 tx 分离，tx hash 不含 witness 解决 malleability，同时打开 Lightning。
- **Taproot**：Schnorr + Merkle Script Tree，多签以单签形式表达。

### 2.5 关键参数与常量

| 参数 | 值 | 源 |
| --- | --- | --- |
| 块大小 | 1 MB base + 3 MB witness | BIP-141 |
| 出块时间目标 | 600s | difficulty adjust |
| Halving 周期 | 210,000 blocks ≈ 4 yr | Bitcoin Core |
| Dust 阈值 | ~546 sat | `GetDustThreshold` |
| Coinbase 锁定期 | 100 blocks | `COINBASE_MATURITY` |
| Max script size | 10,000 bytes | `MAX_SCRIPT_SIZE` |

### 2.6 失败模式

- **51% 攻击**：算力多数可双花，历史上 ETC、BCH SV 多次。
- **Dust Attack**：向大量地址发极小 UTXO 做 de-anonymize。
- **Malleability**：SegWit 前 tx hash 可被第三方修改影响 channel。
- **OP_RETURN spam**：存储非支付数据膨胀 UTXO set。
- **Taproot 解释性错误**：witness version 未识别 → 节点依 softfork 策略反应不一。

```mermaid
flowchart LR
  U1[UTXO A: 2 BTC] --> TX{Tx}
  U2[UTXO B: 3 BTC] --> TX
  TX --> O1[UTXO C: 4 BTC recipient]
  TX --> O2[UTXO D: 0.9 BTC change]
  TX -.fee 0.1.-> Miner
```

```
UTXO lifecycle
  mined in block N as output
  → held in UTXO set
  → spent in block M as input
  → pruned from UTXO set
```

## 3. 架构剖析

### 3.1 分层视图（Bitcoin Core）

1. **P2P**：inv/getdata/headers/compact blocks。
2. **Validation**：tx/block 检查 → ConnectTip → 更新 chainstate。
3. **Chainstate (UTXO set)**：LevelDB 持久化，内存 ~5 GB (2026)。
4. **Mempool**：policy + min-relay-fee。
5. **Wallet**：BIP-32/39/44/84/86 HD。
6. **RPC / REST / ZMQ**：对外接口。

### 3.2 核心模块清单

| 模块 | 职责 | 依赖 | 路径 |
| --- | --- | --- | --- |
| Validation | tx/block verify | Script | `bitcoin/src/validation.cpp` |
| Script interpreter | 栈式执行 | Crypto | `bitcoin/src/script/interpreter.cpp` |
| UTXO DB | coin state | LevelDB | `bitcoin/src/coins.cpp` |
| Mempool | mempool policy | Validation | `bitcoin/src/txmempool.cpp` |
| Wallet | HD 钱包 | bdb/sqlite | `bitcoin/src/wallet/` |
| Net | P2P | libevent | `bitcoin/src/net_processing.cpp` |

### 3.3 数据流：一笔 Bitcoin tx 的生命周期

1. 钱包选币（coin selection, e.g. Branch-and-Bound）。
2. 构造 tx：inputs (txid:index, witness)、outputs (value, scriptPubKey)、locktime。
3. 签名：ECDSA/Schnorr 签 sighash preimage。
4. 广播：`sendrawtransaction` → 节点 mempool → peer INV。
5. 矿工打包：按 feerate 选 tx，组 block。
6. 出块：6 个 confirmations（约 60 分钟）视为终局。
7. 接收方钱包扫描 block → 更新余额。

### 3.4 参考实现

- **Bitcoin Core** C++（主导，95% 节点）。
- **btcd** Go。
- **bcoin** JS。
- **libbitcoin** C++（模块化）。
- **Electrum Server / Fulcrum**：为轻客户端提供索引。

### 3.5 扩展接口

- JSON-RPC：`getblockcount`, `getrawtransaction`, `sendrawtransaction`。
- REST API。
- ZMQ：`rawblock`, `rawtx` topic。
- Lightning：LN 是 off-chain UTXO channel。
- Ordinals / Inscriptions：BRC-20 在 witness 内刻数据。

## 4. 关键代码 / 实现细节

`CheckTxInputs` 简化版（Bitcoin Core v27，`validation.cpp`）：

```cpp
bool CheckTxInputs(const CTransaction& tx, TxValidationState& state,
                   const CCoinsViewCache& inputs, int spendHeight,
                   CAmount& fee)
{
    if (!inputs.HaveInputs(tx)) return state.Invalid(REASON_MISSING_INPUTS);
    CAmount nValueIn = 0;
    for (const CTxIn& in : tx.vin) {
        const COutPoint& prev = in.prevout;
        const Coin& coin = inputs.AccessCoin(prev);
        assert(!coin.IsSpent());
        if (coin.IsCoinBase() && spendHeight - coin.nHeight < COINBASE_MATURITY)
            return state.Invalid(REASON_PREMATURE_COINBASE);
        nValueIn += coin.out.nValue;
        if (!MoneyRange(coin.out.nValue) || !MoneyRange(nValueIn))
            return state.Invalid(REASON_OVER_MAX);
    }
    CAmount nValueOut = tx.GetValueOut();
    if (nValueIn < nValueOut) return state.Invalid(REASON_NEGATIVE_FEE);
    fee = nValueIn - nValueOut;
    return true;
}
```

## 5. 演进与版本对比

| 升级 | 时间 | 改动 |
| --- | --- | --- |
| P2SH (BIP-16) | 2012 | 脚本哈希，复杂多签上链成本 ↓ |
| SegWit (BIP-141) | 2017 | witness 分离，修复 malleability |
| Taproot (BIP-340/341) | 2021 | Schnorr + TapScript |
| Cardano EUTXO | 2020 | datum + script 上链 |
| Fuel UTXO-VM | 2023 | 并行 tx 执行 |
| Bitcoin Covenants (BIP-345) | 待定 | 未来交易约束 |

## 6. 实战示例

```bash
# regtest 下创建并广播一笔 tx
bitcoin-cli -regtest generatetoaddress 101 "$(bitcoin-cli getnewaddress)"
TXID=$(bitcoin-cli -regtest sendtoaddress "$(bitcoin-cli getnewaddress)" 1.0)
bitcoin-cli -regtest getrawtransaction $TXID true | jq '.vin, .vout'
# 观察 inputs/outputs，确认 fee = vin.value - vout.value
```

## 7. 安全与已知攻击

- **Mt. Gox Malleability 2014**：SegWit 前 tx hash 可变，导致撤回误判。
- **Value Overflow 2010**：早期 Bitcoin 0.3 整型溢出造出 1840 亿 BTC，通过 soft fork + reorg 修复。
- **Dust Attack** against privacy wallets。
- **CVE-2018-17144**：重复 input 导致通胀，0.16.3 紧急修复。
- **P2SH collision 2019 theoretical**：SHA-1 冲突对 OP_CHECKMULTISIG 威胁（P2SH uses RIPEMD160(SHA256)，目前安全）。

## 8. 与同类方案对比

| 维度 | UTXO (Bitcoin) | EUTXO (Cardano) | Account (Ethereum) | Object (Sui) |
| --- | --- | --- | --- | --- |
| 并发 | 高 | 高 | 中（依赖 tx 序） | 高 |
| 合约表达 | 低 | 高 (Plutus) | 高 (EVM) | 高 (Move) |
| 隐私 | 中（地址复用差） | 中 | 低 | 中 |
| 状态大小 | UTXO set | UTXO + Script | Trie 大 | Object set |
| ZK-friendly | 高 | 中 | 低 | 中 |

## 9. 延伸阅读

- Nakamoto S., "Bitcoin: A Peer-to-Peer Electronic Cash System"，2008
- Chakravarty et al., "The Extended UTXO Model"，IOHK 2020
- Bitcoin Optech Topics: https://bitcoinops.org/en/topics/
- Bitcoin Core Dev Docs: https://bitcoincore.org/en/doc/

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| UTXO | Unspent Transaction Output | 未花费输出 |
| Coinbase | Coinbase | 区块首笔产币 tx |
| scriptPubKey | Locking Script | 接收方锁 |
| witness | Witness | SegWit 独立签名段 |
| Datum | Datum | EUTXO 合约数据 |

---

*Last verified: 2026-04-22*
