---
title: 委托权益证明（Delegated Proof of Stake, DPoS）
module: 01-infrastructure/consensus
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://github.com/EOSIO/eos
  - https://github.com/bnb-chain/bsc
  - https://developers.tron.network/
  - https://steemit.com/dpos/@dantheman/dpos-consensus-algorithm-this-missing-white-paper
---

# 委托权益证明（Delegated Proof of Stake, DPoS）

> **TL;DR**：DPoS 由 Daniel Larimer 于 2014 年为 BitShares 提出，核心是"代币持有者投票选出一组固定（通常 21–100）'超级节点'轮流出块"，把 PoS 的广泛验证者集合收窄为少数职业节点以换取极高吞吐。代表链是 EOS（21 BP）、Tron（27 SR）、BSC（21/41 PoSA 混合）。DPoS 提供亚秒终局、低能耗、高 TPS，但被批评"中心化、卡特尔化、贿选"。本文比较三条主流 DPoS 链、剖析其与经典 PoS/BFT 的关系，并讨论去中心化争议。

## 1. 背景与动机

2014 年 4 月 Daniel Larimer 在 BitShares 项目中首次实现 DPoS。他认为 PoW 能耗浪费、经典 PoS 验证者"搭便车"难以监管，提议通过 **链上投票选出小规模（N 个）出块节点** 实现秒级确认。2015 年 Larimer 将相同机制用于 Steem（去中心化社交），2018 年创立 EOS.IO，BP（Block Producer）数设为 21。

DPoS 在东亚市场被广泛采用：Tron（2018，Justin Sun 主导）、NEO（早期）、WAX、Hive（Steem 分叉）、Lisk。2020 年币安推出 BNB Smart Chain 采用 **PoSA**（Proof of Staked Authority）——21 个验证者由质押 BNB 的投票者选出，技术上属于 DPoS 变体，并加入 Parlia 共识（Clique-like + epoch rotation）。

核心问题：PoW/经典 PoS 的"开放验证者集合"带来高去中心化但也意味着高通信开销。BFT 消息复杂度 `O(n²)` 限制 n 不能太大。DPoS 把"谁是验证者"从"谁有币"切换到"谁被选上"，这是一种社会-经济混合 Sybil 抵抗。

## 2. 核心原理

### 2.1 形式化定义

DPoS 定义为三元组 `(V, E, R)`：

- **V（验证者集）**：固定大小 N（EOS N=21，Tron N=27，BSC N=21-41）。
- **E（选举函数）**：`E: Ballots × Epoch → V`。选举基于 stake-weighted approval voting。
- **R（轮换规则）**：确定性 round-robin 或 pseudo-random（从 V 中按 slot 顺序抽提议者）。

EOS 选举公式（`eos/libraries/chain/eosio_contract.cpp`）：每个账号可投 ≤ 30 票，stake-weighted 累加到候选人。每 126 秒（21 producer × 6 块 × 1 块/块时间）重新排序。

Finality：EOS 采用 **LIB（Last Irreversible Block）**——2/3 × 21 = 14 个 BP 确认后即终局，平均 ~180 秒。BSC 采用"2/3 验证者见证 + 2 epoch 稳定"。

### 2.2 关键数据结构（EOS）

```cpp
// eos/libraries/chain/include/eosio/chain/producer_schedule.hpp
struct producer_schedule_type {
    uint32_t version;
    vector<producer_key> producers;  // 21 个
};

struct producer_key {
    account_name producer_name;
    public_key_type block_signing_key;
};
```

### 2.3 子机制拆解

**子机制 1：选举**
链上 smart contract `eosio.system::voteproducer`（EOS）或 `system.vote`（Tron）。投票权 = stake × 锁定系数（EOS 有 stake age，近年投票权重高）。BSC 通过 `stakeHub` 合约（Parlia 升级后）委托 BNB，Top 21/41 按 voting power 成为 validator。

**子机制 2：出块轮换**
EOS：21 个 BP 按 schedule 依次出块，每人连续出 6 块（每 0.5s 一块，共 3s），然后切下一 BP。若当前 BP 跳过（miss），下一 BP 直接接替。Tron 每轮 27 × 1 = 27 块 × 3s = 81s。BSC 每 epoch = 200 块重洗 validator schedule。

**子机制 3：终局 LIB**
EOS LIB：当 ≥ 2/3（15/21）BP 对某块及其祖先打 pBFT 预提交签名，该块进入 irreversible。源码 `eos/libraries/chain/fork_database.cpp`。Tron 类似：19/27 SR 确认。

**子机制 4：激励与惩罚**
EOS 通胀 5%/年，其中 1% 给 BP、4% 给 worker proposals。BP 奖励按出块率分配。Tron SR 每出块 32 TRX。BSC validators 获所有区块 gas fee（无通胀奖励，2023 后 BNB Chain Fusion）。惩罚：EOS 无 slashing；BSC Parlia 会跳过恶意签名的 validator 一轮（Clique-like "not authorized" 惩罚），严重时通过治理罢免。

**子机制 5：治理**
EOS：BP 通过 `eosio.system` 可修改核心参数（CPU/NET rate）。Tron SR 票选改参数。BSC 治理由 BNB 持有者投票触发硬分叉。

**子机制 6：账户资源模型**（EOS 独有）
与以太坊 Gas 市场不同，EOS 按账号 **staked CPU/NET/RAM** 分配每日免费资源。这是 DPoS 优化吞吐的代价——必须"租赁"算力，导致 2019-2021 EIDOS 空投滥用事件使 RAM 价格飙升。

### 2.4 参数对比

| 参数 | EOS | Tron | BSC (Parlia) |
| --- | --- | --- | --- |
| 验证者数 N | 21 BP + 备选 | 27 SR + 127 SR Partner | 21/41（Luban 升级 2023） |
| 出块时间 | 0.5 s | 3 s | 3 s（Lorentz 降至 1.5 s 计划） |
| 每人连出 | 6 块 / 3 s | 1 块 | 1 块 |
| Finality | ~180 s LIB | ~60 s | 2 × 21 块 ≈ 126 s（v1.1 后 fast finality） |
| Slashing | 无（社会罚） | 无 | 跳过轮次、治理罢免 |
| Token | EOS | TRX | BNB |
| 年通胀 | 5%（4% 治理基金） | ~3% 销毁 | 0（手续费模式） |
| 投票上限 | 30 | 30 | ~41 |

### 2.5 边界条件与失败模式

- **卡特尔**：2018 EOS 被曝光 BP 互相投票形成 cartel，被链上数据证实。"One whale vote buys all top 21"。
- **贿选**：2019 Huobi 节点被指控用交易所用户存款 stake 自己当 BP。
- **网络分区**：若 ≥ 8 BP 离线，EOS 直接停摆（满足 CP）。2018 主网上线两周内有 multi-hour 停摆记录。
- **治理紧急暂停**：EOS 曾有 BP 仲裁冻结账号（被视为违背区块链不可篡改精神），2019 后废除。

### 2.6 状态机

```mermaid
flowchart LR
    H[代币持有者] -- 投票 --> CT[eosio.system 合约]
    CT -- 累计 stake-weighted --> R[按票数排序]
    R --> TOP[Top 21 BP]
    TOP --> SCHED[轮换 schedule 126s]
    SCHED --> BP1[BP_1 出 6 块]
    BP1 --> BP2[BP_2 出 6 块]
    BP2 --> BPn[BP_21 出 6 块]
    BPn --> LIB[2/3 确认 → LIB]
    LIB --> Final[Irreversible]
```

## 3. 架构剖析

### 3.1 分层视图（EOS）

1. **Contract Layer**：WASM + C++ ABI，`eosio.system`、`eosio.token` 系统合约。
2. **Consensus Layer**：DPoS + pBFT finalizer。
3. **Network Layer**：`net_plugin` P2P。
4. **Chain Layer**：`fork_database` 处理分叉。
5. **RPC Layer**：HTTP + WebSocket。

### 3.2 核心模块清单

| 模块 | 职责 | 源码 | 可替换 |
| --- | --- | --- | --- |
| Chain Controller | 区块应用与 fork 处理 | `eos/libraries/chain/controller.cpp` | 低 |
| Fork Database | 分叉树 | `eos/libraries/chain/fork_database.cpp` | 低 |
| Producer Schedule | BP 轮换 | `eos/libraries/chain/producer_schedule.cpp` | 低 |
| Net Plugin | P2P | `eos/plugins/net_plugin/` | 中 |
| Producer Plugin | BP 签名出块 | `eos/plugins/producer_plugin/` | 中 |
| Chain Plugin RPC | JSON RPC | `eos/plugins/chain_plugin/` | 高 |
| System Contract | eosio.system | `eosio.contracts/contracts/eosio.system/` | 中（治理） |
| BSC Parlia | BSC 共识 | `bsc/consensus/parlia/parlia.go` | 低 |
| BSC System | Validator 集 | `bsc/core/systemcontracts/` | 中 |
| Tron Java-Tron | SR 节点 | `java-tron/framework/src/main/java/org/tron/core/consensus/` | 低 |

### 3.3 端到端数据流（BSC）

1. **T+0**：用户签名 → bsc mempool。
2. **T+0-3s**：当前 validator（按 `parlia.Prepare`）封装块。
3. **T+3s**：广播块至其他 20 个 validator。
4. **T+3-126s**：其他 14/21 validator 在其自己的 turn 出块时隐式确认（通过签名 header）。
5. **T+126s**：2/3 确认完成 → fast finality 生效（BEP-126, [bsc-1.1.18](https://github.com/bnb-chain/bsc/releases/tag/v1.1.18)）。

### 3.4 客户端多样性

- **EOS**：`nodeos`（C++）为 ~99% BP 使用；`eos-vm-oc` 加速。其他实现 Mandel 已在 Antelope 联盟分叉。
- **Tron**：`java-tron` ~99% SR 使用；`rust-tron` 实验性。
- **BSC**：`bsc`（go-ethereum fork）几乎 100%；`bsc-erigon` 为归档节点。

### 3.5 接口

- EOS History API、cleos CLI、State History Plugin（dfuse、Hyperion）。
- BSC EVM JSON-RPC 兼容 Ethereum 所有方法。
- Tron TronGrid HTTP API、wallet-cli。

## 4. 关键代码：BSC Parlia

```go
// bsc/consensus/parlia/parlia.go  (v1.4.x)
func (p *Parlia) Seal(chain consensus.ChainHeaderReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error {
    header := block.Header()
    val, signFn := p.val, p.signFn
    number := header.Number.Uint64()
    snap, err := p.snapshot(chain, number-1, header.ParentHash, nil)
    if err != nil { return err }
    if _, authorized := snap.Validators[val]; !authorized {
        return errors.New("unauthorized validator")
    }
    for seen, recent := range snap.Recents {
        if recent == val && number-seen < uint64(len(snap.Validators)/2+1) {
            return errors.New("signed recently, must wait for others")
        }
    }
    delay := time.Until(time.Unix(int64(header.Time), 0))
    if header.Difficulty.Cmp(diffNoTurn) == 0 {
        wiggle := time.Duration(len(snap.Validators)/2+1) * wiggleTime
        delay += time.Duration(rand.Int63n(int64(wiggle)))
    }
    sighash, err := signFn(accounts.Account{Address: val}, accounts.MimetypeParlia, ParliaRLP(header, p.chainConfig.ChainID))
    if err != nil { return err }
    copy(header.Extra[len(header.Extra)-extraSeal:], sighash)
    go func() {
        select {
        case <-time.After(delay):
        case <-stop: return
        }
        results <- block.WithSeal(header)
    }()
    return nil
}
```

## 5. 演进时间线

| 年份 | 事件 |
| --- | --- |
| 2014-04 | BitShares DPoS 上线 |
| 2016-03 | Steem |
| 2018-06 | EOS 主网（21 BP） |
| 2018-07 | Tron 主网（27 SR） |
| 2019-01 | BlockOne 与 BPs 治理争议 |
| 2020-04 | Steem 硬分叉为 Hive（DPoS 治理危机） |
| 2020-09 | BSC 主网上线（21 validator） |
| 2022-10-07 | BSC 跨链桥被盗 $570M（[BNB Chain postmortem](https://www.bnbchain.org/en/blog/bnb-chain-ecosystem-update)）|
| 2023 | EOS 社区分叉为 Antelope + EOS Network Foundation |
| 2023-06 | BSC Luban 升级，fast finality + 41 validator |
| 2024 | BSC BNB Fusion（BC→BSC 合并，Parlia v3） |
| 2025 | BSC Lorentz 升级，1.5s 出块计划 |

## 6. 实战示例

```bash
# EOS：查询当前 21 BP
cleos -u https://eos.api.eosnation.io get producers --limit 21 --reverse

# BSC：查询当前 validator set
curl -X POST https://bsc-dataseed.binance.org/ -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_call","params":[{"to":"0x0000000000000000000000000000000000001000","data":"0xb7ab4db5"},"latest"],"id":1}'

# Tron：查询 27 SR
curl https://api.trongrid.io/wallet/listwitnesses
```

## 7. 安全与已知攻击

- **EOS vote-buying 2018-06**：Huobi 交易所被曝光在 EOS 快照前用用户 EOS 自投 BP，获"Top 21 每月 ~$500K 通胀收益"。引发"DPoS = 市值选举"批评。
- **Steem 治理危机 2020-03**：Justin Sun 收购 Steemit Inc. 后动用其存在交易所（Binance/Huobi/Poloniex）的 STEEM 投票接管 BP。社区硬分叉为 Hive。
- **BSC Cross-Chain Bridge 2022-10-07 被盗 $570M**：Token Hub 合约漏洞被利用，攻击者伪造 IAVL proof 铸造 2M BNB。BSC validator 紧急社会协调停机修补。此事件说明"小型验证者集合 = 社会治理反应快"（好处）和"中心化（坏处）"两面性。
- **Ronin 桥 2022-03-23 被盗 $625M**：虽非严格 DPoS 但 9 validator 中 5 被 Lazarus 社工入侵，超 BFT 阈值。本质教训：DPoS-like 模型下节点数过少，社工攻击性价比高。
- **Tron 停机**：2022-05、2023-03 各有一次数小时停机，官方归因"网络问题"。

## 8. 与同类方案对比

| 维度 | DPoS (EOS) | 经典 PoS (Ethereum) | Tendermint | PoA |
| --- | --- | --- | --- | --- |
| 验证者数 | 21 | ~1M | 通常 150 | 许可白名单 |
| 选举方式 | 代币投票 | 32 ETH 门槛（无投票） | stake + 治理 | 链下合规 |
| Finality | ~180s LIB | ~12.8 min 经济 | 即时 | 即时（许可） |
| 去中心化 | 低-中 | 高 | 中 | 极低 |
| 吞吐 (TPS) | 4000+（宣传） / ~1000 实测 | 15-30（L1） / 10k+（L2） | 1000-10000 | 10000+ |
| 激励 | 通胀 5% | 通胀 + MEV + tip | 通胀 | 出块费 |
| 故障模式 | 卡特尔、贿选、停摆 | Inactivity leak | 停摆等 2/3 | 中心化失败 |

**去中心化争议**：DPoS 采样空间小，天生易被卡特尔化。Vitalik 在 [A Proof of Stake Design Philosophy](https://vitalik.eth.limo/) 中批评 DPoS"把权力集中到 21 个节点上，却假装是去中心化"。支持方（如 Larimer）则指出"代币持有者本来就是实质控制者，公开选举比隐式算力好"。

## 9. 延伸阅读

- **Tier 1**：
  - [EOSIO repo](https://github.com/EOSIO/eos)
  - [bnb-chain/bsc](https://github.com/bnb-chain/bsc)
  - [java-tron](https://github.com/tronprotocol/java-tron)
  - [Steemit DPoS missing whitepaper](https://steemit.com/dpos/@dantheman/dpos-consensus-algorithm-this-missing-white-paper)
- **Tier 2/3**：
  - Messari BSC 年度报告
  - a16z [On-chain voting](https://a16zcrypto.com/)
  - Vitalik [On Collusion](https://vitalik.eth.limo/general/2019/04/03/collusion.html)
  - learnblockchain.cn DPoS 源码解析
- **BEP**：
  - [BEP-126 Fast Finality](https://github.com/bnb-chain/BEPs/blob/master/BEP126.md)
  - [BEP-153 BNB Staking](https://github.com/bnb-chain/BEPs/blob/master/BEP153.md)
  - [BEP-294 BC Fusion](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP294.md)

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 区块生产者 | Block Producer (BP) | EOS 的 21 个出块节点 |
| 超级节点 | Super Representative (SR) | Tron 的 27 个出块节点 |
| PoSA | Proof of Staked Authority | BSC 混合 DPoS+PoA |
| LIB | Last Irreversible Block | EOS 的终局指针 |
| Schedule | Producer Schedule | BP 轮换表 |
| Wiggle | Wiggle Time | Parlia 非 in-turn 的随机延迟 |
| 委托 | Delegation | 将投票权委托给候选人 |
| 卡特尔 | Cartel | 验证者互相投票形成垄断 |
| 贿选 | Vote Buying | 收购代币以获选举权 |

---

*Last verified: 2026-04-22*
