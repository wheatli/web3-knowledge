---
title: Solana 账户模型（Account Model, PDA, Rent, System Program）
module: 03-smart-contract/solana
priority: P0
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://solana.com/docs/core/accounts
  - https://solana.com/docs/core/pda
  - https://solana.com/docs/core/fees#rent
  - https://github.com/anza-xyz/agave/tree/master/programs/system
  - https://github.com/anza-xyz/agave/blob/master/sdk/program/src/pubkey.rs
---

# Solana 账户模型

> **TL;DR**：Solana 把"状态"完全外置到**账户（Account）**——每个账户是以 32 字节 `Pubkey` 为键、由特定 Program 拥有的可变字节数组。与 EVM 把代码和存储捆在同一合约地址不同，Solana 严格分离：**程序账户**（executable=true，持字节码）、**数据账户**（持业务状态）、**PDA**（程序派生地址，无私钥，程序自治子账户）三类。账户空间按字节收取 **Rent**（租金，目前必须预存 2 年租金豁免，称 rent-exempt），由 **System Program**（`11111111111111111111111111111111`）统一负责创建、转账、分配 Owner。所有写入都必须 "声明账户 + 通过该账户 owner 的 Program 执行"——这是 Sealevel 并行执行和安全边界的数据层根基。

---

## 1. 背景与动机

以太坊 EVM 的账户模型早期（2015）把 **合约地址 = 代码 + storage trie 根** 绑定，这导致：

1. 状态查询必须遍历 MPT，不利于并行。
2. 同一合约地址的所有状态读写互相竞争，难以分片。
3. 代码升级极困难（CA 不可变），催生了 EIP-1967 proxy 等 workaround。

Anatoly 设计 Solana 时借鉴了 **操作系统文件系统 + Actor 模型**：

- "**Accounts are the files of Solana**"（Solana Cookbook 原话）。
- 每个账户等价于一个文件：`pubkey` 是 inode，`owner` 是所属进程（Program），`data` 是文件内容，`lamports` 是磁盘配额。
- Program 像操作系统内核模块：有代码但无自己的状态；它通过操作自己 owner 的数据账户来完成业务。

这样获得：

- **状态分片天然成立**：不同账户在物理上就是独立的 `AccountSharedData`，无需"按合约切 Merkle 分支"。
- **并行执行可行**：见 Sealevel。
- **代码升级简单**：program 账户可通过 BPF loader 替换字节码。

## 2. 核心原理

### 2.1 形式化定义

Solana 全局状态 `S` 可形式化为函数：

```
S : Pubkey → AccountSharedData
```

其中 `AccountSharedData = (lamports: u64, data: Vec<u8>, owner: Pubkey, executable: bool, rent_epoch: u64)`。

一笔 Tx 是状态转换 `S → S'`，满足以下 **不变式**：

- **I1（Owner 单写）**：`data` 或 `lamports` 的减少只能由 `account.owner == executing_program_id` 的 Program 执行（lamports 增加任意程序可做——这是转账的基础）。
- **I2（签名或 PDA 签名）**：对任一 "required signer" 账户的修改必须伴随其 Ed25519 签名；PDA 在 CPI 时以 `invoke_signed(seeds)` 方式由拥有程序"签名"。
- **I3（Rent 豁免）**：`lamports ≥ min_balance(data.len())`，否则账户可被回收（2026 年后强制，见 SIMD-0099）。
- **I4（可执行不可写）**：`executable = true` 的账户 data 只能由对应的 BPF loader 修改（部署/升级路径）。
- **I5（Size 不超限）**：`data.len() ≤ 10 MB`。

### 2.2 Account 数据结构

`sdk/src/account.rs`：

```rust
pub struct Account {
    pub lamports: u64,       // 余额（1 SOL = 10^9 lamports）
    pub data: Vec<u8>,       // 原始字节状态，最大 10 MB
    pub owner: Pubkey,       // 谁能写这账户（某个 Program）
    pub executable: bool,    // 是否是字节码账户
    pub rent_epoch: u64,     // 上次 rent 检查的 epoch
}
```

对应运行时的 `AccountSharedData`（内存里共享 COW 副本）。Pubkey 的生成有两种：

- **普通账户**：`Pubkey` 为 Ed25519 公钥（有私钥持有者控制签名）。
- **PDA**：`Pubkey` 为 `find_program_address(seeds, program_id)` 的结果——**落在 Ed25519 曲线外**（off-curve），没有对应私钥，只能由程序通过 `invoke_signed(seeds, bump)` "代签名"。

### 2.3 PDA 推导算法

```rust
pub fn find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> (Pubkey, u8) {
    for bump in (0..=255).rev() {
        let mut hash_input = [seeds, &[&[bump]], program_id.as_ref(), b"ProgramDerivedAddress"].concat();
        let hash = Sha256::digest(&hash_input);
        let pubkey = Pubkey::new(&hash);
        if !pubkey.is_on_curve() { return (pubkey, bump); }
    }
    panic!("UnreachableBump");
}
```

- 从 `bump = 255` 递减，逐次 SHA256 `(seeds || bump || program_id || "ProgramDerivedAddress")`。
- 若哈希落在 Ed25519 曲线上（可能存在私钥）→ 丢弃；若在曲线外 → 返回（`pubkey`, `bump`）。
- "canonical bump" = 第一个成功的 `bump`。客户端与链上必须使用同一 `bump` 否则地址不匹配。

### 2.4 子机制拆解

**(1) 空间分配**：创建账户用 `SystemProgram::CreateAccount { lamports, space, owner }`。runtime 为新账户分配 `space` 字节、预付 `lamports`、设置 `owner`。必须由 payer 签名。

**(2) Rent / 豁免**：Solana 账户按字节计费。`Rent` sysvar：`lamports_per_byte_year = 3480`（固定），`exemption_threshold = 2.0`（2 年）。一个 165 字节 SPL Token Account 需预存 `(165+128) × 3480 × 2 ≈ 2,039,280 lamports ≈ 0.002 SOL`。2022 年后"交 rent 而非豁免"的账户已被废弃（feature `DisableFees`），现仅允许 rent-exempt。

**(3) System Program**：Native Program（`11111...`，无字节码），处理账户生命周期：`CreateAccount / Transfer / Allocate / Assign / CreateAccountWithSeed / TransferWithSeed / AdvanceNonceAccount / AuthorizeNonceAccount / UpgradeNonceAccount`。

**(4) Realloc**：BPF loader v3 允许程序在运行时调用 `AccountInfo::realloc(new_len, zero_init)` 扩展（或缩小）账户 `data` 长度，最多每次 `MAX_PERMITTED_DATA_INCREASE = 10 KB`，单账户总 `space` 不超 10 MB。

**(5) Close / Lamport 扫零**：把账户 `lamports` 减到 0 → 下一 epoch 自动 GC。约定做法：`to.lamports += from.lamports; from.lamports = 0; from.assign(&System); from.realloc(0)`。

**(6) Ownership 转移**：`Assign { owner: new_program }`。仅当前 owner（通常是 System）可调用；data 必须全零。

### 2.5 参数与常量

| 参数 | 值 | 说明 |
| --- | --- | --- |
| Account data 最大 | 10 MB | `MAX_PERMITTED_DATA_LENGTH` |
| 单次 realloc 增长 | 10,240 B | `MAX_PERMITTED_DATA_INCREASE` |
| Rent lamports/byte/year | 3,480 | 固定（非治理） |
| Rent 豁免年数 | 2.0 | `exemption_threshold` |
| 每账户 header overhead | 128 B | rent 计算时加入 |
| Pubkey 长度 | 32 B | Ed25519 |
| Lamport | 10⁻⁹ SOL | 最小单位 |
| PDA 最大 seed 数 | 16 | `MAX_SEEDS` |
| 单 seed 最大 | 32 B | `MAX_SEED_LEN` |

### 2.6 边界条件与失败模式

- **非豁免账户创建**：runtime 直接报 `InsufficientFundsForRent`（2023 年后硬化）。
- **重复创建**：`CreateAccount` 目标账户已存在 → `AccountAlreadyInUse`。
- **Owner mismatch 写入**：程序试图写非本 owner 账户 → `ExternalAccountDataModified`。
- **未签名修改 lamports**：向外转账未签名 → `MissingRequiredSignature`。
- **PDA 冲突**：理论上 2⁻²⁵⁶ 概率两个不同 `(seeds, program_id)` 产生同一 pubkey；实践忽略。
- **Rent 周期租金恢复**：feature 开关 `disable_rent_collection` 激活前的旧账户若 rent 不足会被按 epoch 扣 lamports 直至清零。
- **10 MB 软限**：NFT 大图片等不适合放 data，需走链下或 state compression（Merkle tree 存叶哈希）。

### 2.7 账户模型 Mermaid

```mermaid
stateDiagram-v2
  [*] --> NonExistent
  NonExistent --> RentExempt : SystemProgram::CreateAccount
  RentExempt --> Writable : owner.process_instruction
  RentExempt --> Reassigned : SystemProgram::Assign
  Reassigned --> Writable
  Writable --> RentExempt : commit
  Writable --> Closed : lamports=0 & data.len()=0
  Closed --> [*]
```

### 2.8 ASCII 结构图

```
+----------------------- AccountsDB (mmap shards) ----------------------+
|  shard_00   shard_01  ...  shard_FF    (按 pubkey 前缀分片)           |
|     \          |              /                                       |
|      v         v             v                                        |
|   [ Pubkey_A ] [ Pubkey_B ] [ Pubkey_C ] ...                          |
|     |              |              |                                   |
|     v              v              v                                   |
|   Account(        Account(       Account(                             |
|    lamports,       lamports,      lamports,                           |
|    owner=Prog1,    owner=Sys,     owner=Prog2,                        |
|    data=[...],     data=[],       data=[...],                         |
|    exec=false)     exec=false)    exec=true)  <- program_binary       |
+-----------------------------------------------------------------------+
```

## 3. 架构剖析

### 3.1 分层视图

1. **AccountsDB 层**（`accounts-db/src/accounts_db.rs`）：按 pubkey 分片的 mmap 存储；append-only 写，version-based GC。
2. **Bank 层**（`runtime/src/bank.rs`）：当前 slot 的账户快照缓存，负责 rent 收集与状态提交。
3. **System Program**（`programs/system/src/system_processor.rs`）：账户生命周期逻辑。
4. **Program Runtime 层**：BPF 程序通过 `AccountInfo` 句柄读写。
5. **RPC / Geyser 接口层**：对外暴露账户查询与订阅。

### 3.2 模块表

| 模块 | 路径 | 职责 | 依赖 | 可替换性 |
| --- | --- | --- | --- | --- |
| AccountsDB | `accounts-db/src/accounts_db.rs` | mmap 持久化 + index | — | 低 |
| AccountsIndex | `accounts-db/src/accounts_index.rs` | pubkey→slot map | AccountsDB | 低 |
| SystemProgram | `programs/system/` | 创建/转账/assign | — | 低 |
| Rent sysvar | `sdk/program/src/rent.rs` | rent 常量 | — | 低（feature） |
| StakeProgram | `programs/stake/` | Vote/Stake 账户生命周期 | System | 低 |
| BPF Loader v3/v4 | `programs/bpf_loader/` | program 账户部署/升级 | AccountsDB | 低 |
| Geyser Plugin | `geyser-plugin-interface` | 账户变更流 | Bank commit | 高 |

### 3.3 账户生命周期数据流

```mermaid
sequenceDiagram
  participant Payer
  participant SysProg as SystemProgram
  participant NewAcct
  participant MyProg as MyProgram
  participant AccDB as AccountsDB
  Payer->>SysProg: CreateAccount(lamports=x, space=n, owner=MyProg)
  SysProg->>NewAcct: allocate n bytes, set owner
  SysProg->>AccDB: commit (payer -x, new acct +x)
  Payer->>MyProg: call with [NewAcct]
  MyProg->>NewAcct: process_instruction (写 data)
  MyProg->>AccDB: commit
  Note over NewAcct: 后续 Tx 可引用其 pubkey
  Payer->>MyProg: close (transfer lamports out)
  MyProg->>NewAcct: realloc 0 & assign System
  MyProg->>AccDB: commit; GC eligible
```

### 3.4 参考实现与客户端差异

- **Agave** 用 RocksDB + mmap (`tiered_storage`) 混合。
- **Firedancer** 用自研 `fd_funk`（fork-aware DB），支持 ~百万 write/s。
- **Sig**（只读）引 lmdb 原型。

API 语义一致：所有客户端 Bank state hash 必须字节相同。

### 3.5 扩展接口

- **JSON-RPC**：`getAccountInfo(pubkey, {encoding, dataSlice, commitment})`、`getProgramAccounts(program, {filters: memcmp/dataSize})`、`getMultipleAccounts`、`getTokenAccountsByOwner`。
- **WebSocket**：`accountSubscribe` / `programSubscribe`。
- **Geyser Plugin**：`ReplicaAccountInfoV3` 流式；用于 Helius、Triton、Yellowstone gRPC。
- **gPA 限速**：`getProgramAccounts` 对大 program 代价高；生产索引器多依赖 Geyser。

## 4. 关键代码 / 实现细节

PDA 推导（`sdk/program/src/pubkey.rs`，v2.1）：

```rust
impl Pubkey {
    pub fn find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> (Pubkey, u8) {
        let mut bump_seed = [u8::MAX];
        for _ in 0..u8::MAX {
            let mut seeds_with_bump = seeds.to_vec();
            seeds_with_bump.push(&bump_seed);
            match Self::create_program_address(&seeds_with_bump, program_id) {
                Ok(pk) => return (pk, bump_seed[0]),
                Err(PubkeyError::InvalidSeeds) => { bump_seed[0] -= 1; }
                _ => panic!(),
            }
        }
        panic!("Unable to find viable bump seed");
    }
    pub fn create_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> Result<Pubkey> {
        let mut hasher = Sha256::new();
        for seed in seeds { hasher.update(seed); }
        hasher.update(program_id.as_ref());
        hasher.update(b"ProgramDerivedAddress");
        let bytes: [u8; 32] = hasher.finalize().into();
        if bytes_are_curve_point(&bytes) { Err(PubkeyError::InvalidSeeds) } else { Ok(Pubkey::new_from_array(bytes)) }
    }
}
```

System Program 核心（`programs/system/src/system_processor.rs`）：

```rust
SystemInstruction::CreateAccount { lamports, space, owner } => {
    let [from, to] = &accounts else { return Err(NotEnoughAccountKeys); };
    if !from.is_signer() { return Err(MissingRequiredSignature); }
    if to.data_len() != 0 { return Err(AccountAlreadyInUse); }
    if to.lamports() != 0 { return Err(AccountAlreadyInUse); }
    from.checked_sub_lamports(lamports)?;
    to.checked_add_lamports(lamports)?;
    to.realloc(space as usize, true)?;
    to.assign(&owner);
    Ok(())
}
```

## 5. 演进与版本对比

| 时间 | 事件 |
| --- | --- |
| 2020-03 | 账户模型 + System Program 主网 |
| 2020-07 | SPL Token Program（账户派生规范） |
| 2021 | Associated Token Account (ATA) 标准化 |
| 2022 | Realloc 支持（BPF loader v3） |
| 2023 | Rent 收集禁用（disable_rent_collection feature） |
| 2024 | Token-2022 扩展（加入 transfer fee、confidential transfer 等） |
| 2024 | State Compression（Merkle tree 压缩百万 NFT） |
| 2025 | ZK Compression（Light Protocol） |

## 6. 实战示例

```ts
import { Connection, Keypair, SystemProgram, Transaction, LAMPORTS_PER_SOL } from "@solana/web3.js";

const c = new Connection("https://api.devnet.solana.com");
const payer = Keypair.generate();
await c.requestAirdrop(payer.publicKey, 2 * LAMPORTS_PER_SOL);

const newAcct = Keypair.generate();
const space = 100;
const rent = await c.getMinimumBalanceForRentExemption(space);

const tx = new Transaction().add(SystemProgram.createAccount({
  fromPubkey: payer.publicKey,
  newAccountPubkey: newAcct.publicKey,
  lamports: rent,
  space,
  programId: SystemProgram.programId, // 归属 System，后续可 assign 给自定义 Program
}));
await c.sendTransaction(tx, [payer, newAcct]);
```

推导 PDA：

```ts
import { PublicKey } from "@solana/web3.js";
const programId = new PublicKey("Token...");
const [pda, bump] = PublicKey.findProgramAddressSync(
  [Buffer.from("vault"), payer.publicKey.toBuffer()],
  programId,
);
console.log(pda.toBase58(), bump);
```

## 7. 安全与已知攻击

- **Owner 检查缺失**：程序未校验 `account.owner == program_id` → 攻击者伪造同 layout 账户读入（"伪账户注入"）。修复：**Account 反序列化前必须校验 owner**。Anchor 自动做。
- **Signer 检查缺失**：未校验 `account.is_signer` → 他人可冒充权限（经典 Anchor CVE-2022）。
- **PDA bump 非 canonical**：接受任意 bump 让攻击者用非 canonical bump 重入子状态（CVE-2022 在 Sollet、Solend 现过）。修复：始终 `bump = find_program_address(...).1` 并 on-chain 校验。
- **Close 不清零**：关闭账户只 transfer lamports 不 `realloc(0)` + `assign(System)` → 数据残留、 race-condition 再 deposit 可被重新激活。修复标准：三步都做。
- **Rent 漏算**：扩容 `realloc(+n)` 未同步 transfer 对应 rent → runtime 报错；但若用 feature 禁用 rent 后未来可能复活。
- **Large account DoS**：申请 10 MB 空账户占据磁盘。对冲：创建费正比于空间。

## 8. 与同类方案对比

| 维度 | Solana Account | EVM Contract Storage | Move Resource | Sui Object |
| --- | --- | --- | --- | --- |
| 状态持有方 | 账户（可多） | 合约地址唯一 | 全局 `address::type` | 对象（有独立 ID） |
| 代码/状态分离 | 是 | 否 | 模块分离、资源外置 | 是 |
| 并行基础 | 预声明账户 | 无 | 类型 + 地址双键 | 对象所有权 |
| 外部引用 | Pubkey 直接 | 合约 + storage key | 类型路径 | ObjectID |

## 9. 延伸阅读

- Solana Docs / Accounts：<https://solana.com/docs/core/accounts>
- Solana Cookbook — PDAs：<https://solana.com/developers/cookbook/accounts/pda>
- Neodyme Audit 博客（PDA bump 攻击）：<https://neodyme.io/blog/solana_common_pitfalls>
- Anchor Constraints：<https://www.anchor-lang.com/docs/the-accounts-struct>
- Helius "Account Model Deep Dive"：<https://www.helius.dev/blog>
- 登链社区：<https://learnblockchain.cn/tags/Solana>

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| 账户 | Account | Solana 基础状态单元（lamports + data + owner） |
| 程序账户 | Program Account | `executable = true` 的账户，持 BPF 字节码 |
| 数据账户 | Data Account | 业务状态账户 |
| 程序派生地址 | Program Derived Address (PDA) | off-curve 无私钥地址，程序子状态容器 |
| 租金豁免 | Rent Exempt | 预存 ≥ 2 年 rent，账户永不清零 |
| Lamport | Lamport | 10⁻⁹ SOL，最小单位 |
| 系统程序 | System Program | 内置 native program，管账户生命周期 |
| 关联代币账户 | Associated Token Account (ATA) | 约定 PDA 形式的 SPL Token 账户 |

---

*Last verified: 2026-04-22*
