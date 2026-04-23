---
title: ENS 域名系统：Registry / Resolver、Name Wrapper、L2 Name（ENSv2）
module: 04-dapp/social-dao
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://docs.ens.domains/
  - https://github.com/ensdomains/ens-contracts
  - https://eips.ethereum.org/EIPS/eip-137
  - https://eips.ethereum.org/EIPS/eip-3668
  - https://ens.domains/blog/post/ensv2
---

# ENS 域名系统：Registry、Resolver、Name Wrapper 与 L2 Namespaces（ENSv2）

> **TL;DR**：ENS（Ethereum Name Service）把人类可读的 `alice.eth` 映射到以太坊地址、内容哈希、社交资料等。核心架构遵循 EIP-137：Registry（所有权树）+ Resolver（记录插件）分离。2022 年 Name Wrapper 把 ENS 转为 ERC-1155 以支持子域 NFT 化与 Fuse 可控制权。2023 年 CCIP-Read（EIP-3668）让 ENS 支持 gas-free L2 / 链下域名解析。2024 年 ENS Labs 宣布 ENSv2 路线图：把 Registry 搬到 L2（自建 Namechain，基于 OP Stack），大幅降低续费 gas。本篇讲清 ENS 四个层次、namehash 算法、反向解析、ERC-1155 wrapped name 的 Fuse 位图、以及 L2 Name 的解析原理。

## 1. 背景与动机

区块链地址是 20 字节十六进制串，人类既难记又易出错。2017 年 Nick Johnson 在以太坊基金会启动 ENS 项目，目标是构建一个「可组合、不可审查的命名系统」。初版合约 2017-05 部署；2019 年完成 `.eth` 永久注册与荷兰拍释放；2021-11 ENS 发行 ENS 治理代币并空投给老用户，催生下一波采用浪潮（vitalik.eth 等名字被广泛使用）。

ENS 的设计参考了 DNS 但在两点根本不同：
1. **所有权基于智能合约而非集中 registrar**：`.eth` TLD 由智能合约管理，无人可审查/没收（除合约升级，由 DAO 治理）。
2. **记录可扩展**：一个名字不仅映射到 ETH 地址，还可保存 BTC 地址、IPFS 内容哈希、社交链接（EIP-634 text record）、邮件等。

主要演进里程碑：
- 2017-05：ENS 初版上线，TLD `.eth`。
- 2019-05：永久注册机制（.eth 7 字符以上免荷兰拍）。
- 2021-11：ENS 空投 & DAO 成立。
- 2022-03：Name Wrapper v1 上线（ERC-1155）。
- 2022-11：引入 CCIP-Read 用于 Optimism / Base / Linea 子域。
- 2023-10：Name Wrapper v2 与 safer Fuse 升级。
- 2024-04：ENSv2 路线图宣布，计划把 Registry 迁移到 L2 Namechain。
- 2025–2026：L2 原生 `.eth` 子域逐步普及，Coinbase `.base.eth`、Optimism `.op.eth` 等。

## 2. 核心原理

### 2.1 Namehash 算法与命名空间

ENS 用 `namehash` 把任何字符串名字递归哈希成 32 字节 `bytes32 node`：

```
namehash('') = 0x00...00
namehash(name) = keccak256(namehash(parent) || keccak256(label))
```

例：
- `eth` 的 node = `keccak256(0x00...00 || keccak256('eth'))` = `0x93cdeb70...`
- `vitalik.eth` 的 node = `keccak256(namehash('eth') || keccak256('vitalik'))`

Namehash 保证层级性：修改父 registry 的所有权可以影响全部子域。label 先 `nameprep` 规范化（UTS46 / EIP-137 附录）。所有 ENS 合约查询以 `bytes32 node` 为键。

### 2.2 Registry / Resolver 分离

- **Registry（ENS Registry）**：一个合约，存 `nodes[node] = { owner, resolver, ttl }`。只做所有权与 resolver 指针的管理，不存具体记录。
- **Resolver**：实际返回记录的合约，实现 `addr(bytes32)` / `text(bytes32,string)` / `contenthash(bytes32)` 等接口。每个 node 可指定不同的 resolver，允许自定义实现。
- **Registrar**：控制某个父域的子域发放规则，如 `.eth` 的 `BaseRegistrarImplementation`（ERC-721，tokenId = `keccak256(label)`）。

分层逻辑让 ENS 可以与任何 DNS TLD 互操作：EIP-1062 引入 DNSSEC 验证，任何拥有 DNSSEC 签名的普通 DNS 域名（如 `alice.xyz`）也能申领为 ENS 使用，Resolver 可以沿用同一组接口。

### 2.3 Name Wrapper 与 Fuses

标准 `.eth` 注册返回 ERC-721（tokenId = labelhash）。但子域无法独立成为 NFT，也无法组合访问控制。Name Wrapper（2022 部署）把 name 包装为 ERC-1155 NFT 并引入 **Fuses（32 位权限位图）**：

关键 Fuse 位：
- `CANNOT_UNWRAP`：不可解包。
- `CANNOT_BURN_FUSES`：不可再烧新 fuse。
- `CANNOT_TRANSFER`：不可转移。
- `CANNOT_SET_RESOLVER`：不可改 resolver。
- `CANNOT_SET_TTL`：不可改 TTL。
- `CANNOT_CREATE_SUBDOMAIN`：不可再创子域。
- `PARENT_CANNOT_CONTROL`：父不能撤销（释放子域完整控制权）。
- `CAN_EXTEND_EXPIRY`：可延长到期。

Fuse 是 **单调烧录** 的：一旦 burn 不可撤销。这让父域可以 promise「我永远不能收回你这个子域」（例：`coolwallet.eth` 分发 `alice.coolwallet.eth`）。

### 2.4 CCIP-Read（EIP-3668）与 L2 Names

传统 ENS 所有记录必须上链，对小钱包而言 gas 昂贵。EIP-3668 定义「OffchainLookup」revert，让 resolver 返回一个 URL，客户端（viem/ethers）fetch 后再把结果 POST 回合约验证：

```
function resolve(bytes calldata name, bytes calldata data) public view returns (bytes memory) {
    revert OffchainLookup(address(this), urls, callData, callbackSig, extraData);
}
```

ENS 通过 CCIP-Read 实现 Optimism/Base/Linea 等 L2 子域：
1. 用户在 L2 上的「NameNode」合约设置记录。
2. L1 Resolver 在查询时 revert + URL。
3. Client 调用 URL（网关服务）取 L2 状态 proof（Merkle/State proof）。
4. Client 把 proof 回给 L1 callback，合约验证 state root。

这让 `alice.cb.id`（Coinbase `.base.eth` 的 subdomain）在 L2 上几乎免费注册，而在 L1 可被解析。

### 2.5 关键参数（截至 2026-04）

| 参数 | 值 | 可治理 |
| --- | --- | --- |
| `.eth` 最短长度 | 3 字符 | DAO |
| 5 字符名字年费 | $640 | DAO |
| 4 字符名字年费 | $160 | DAO |
| 3 字符名字年费 | $640（原 $640，曾调整） | DAO |
| 5+ 字符年费 | $5 | DAO |
| Grace Period | 90 天 | DAO |
| Premium 释放周期 | 21 天指数下降 | DAO |
| Controller | 多版本（`ETHRegistrarControllerV2/V3`） | DAO |

### 2.6 边界条件与失败模式

- **到期后 Premium 拍卖**：5 字符以上域名到期 90 天 grace 后进入 21 日荷兰拍，起价 $100M → 下降到 $0。未及时续费的持有人可能失去名字。
- **Unicode 同形异义**（homograph）：如 `vіtalik.eth`（西里尔 і）与 `vitalik.eth` 视觉一致，可能钓鱼。UI 方应警告。
- **Resolver 作恶**：用户设置了恶意 resolver，可返回伪造的 addr。建议使用 Public Resolver 或经过审计的合约。
- **Name Wrapper Fuse 锁死**：若 burn 了 `CANNOT_UNWRAP + CANNOT_TRANSFER`，名字终生不可转移。
- **CCIP-Read 网关中心化**：网关挂掉则 L2 name 无法解析；可通过多网关 + 白名单缓解。

## 3. 架构剖析

### 3.1 分层视图

1. **L0 Registry**：`ENSRegistry`（地址 `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`），根合约。
2. **L1 Registrar**：`.eth` `BaseRegistrarImplementation`（ERC-721）+ `ETHRegistrarController`（注册 UI 入口）。
3. **L2 Wrapper**：`NameWrapper`（ERC-1155）把 name NFT 化。
4. **L3 Resolver**：`PublicResolver`（默认）+ 自定义 resolver。
5. **L4 Client**：viem/ethers/ensjs 提供 resolve API。
6. **L5 应用**：MetaMask / Rainbow / Lens / Farcaster 的 name 显示。

### 3.2 核心模块清单

| 模块 | 职责 | 地址 | 可替换性 |
| --- | --- | --- | --- |
| ENSRegistry | 根树 | `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e` | 低（只能治理迁移）|
| BaseRegistrar (ERC-721) | `.eth` 所有权 | `0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85` | 中 |
| ETHRegistrarController | 注册入口 | 多版本 | 高 |
| NameWrapper | ERC-1155 包装 | `0xD4416b13d2b3a9aBae7AcD5D6C2BbDBE25686401` | 中 |
| PublicResolver | 默认记录合约 | `0x231b0Ee14048e9dCcD1d247744d114a4EB5E8E63` | 高 |
| ReverseRegistrar | 反向映射 addr→name | `0xa58E81fe9b61B5c3fE2AFD33CF304c454AbFc7Cb` | 低 |
| UniversalResolver | 客户端 batch 查询 | — | 高 |
| ENS DAO Governor | 协议治理 | — | 高 |

### 3.3 数据流 / 生命周期：注册 + 解析

**注册 `alice.eth`**：

1. 用户前端（app.ens.domains）调用 `Controller.commit(commitment)`，commitment = hash(label, owner, secret)。等待 60 秒（防 front-run）。
2. 用户调用 `register(label, owner, duration, secret, resolver, data, reverseRecord, ownerControlledFuses)`，支付 ETH（按 USD 价的年费 + premium）。
3. Controller 调用 BaseRegistrar 铸造 ERC-721，owner = user。若 `reverseRecord = true`，同时设置反向记录。
4. 若 Wrap = true，调用 NameWrapper 转成 ERC-1155 且 burn 初始 fuse。
5. 用户后续用 Resolver 设 addr / text / contenthash。

**解析 `alice.eth` → 地址**：

1. Client 计算 namehash。
2. 调用 Registry.resolver(node) 得 resolver 地址。
3. 调用 resolver.addr(node)（或 addr(node, 60)）返回地址。
4. 若 resolver revert OffchainLookup → Client 按 EIP-3668 走 gateway → callback。
5. 反向解析：`addr.reverse` namespace 下 `reverse(addr) = 'alice.eth'`，client 再正向解析确认一致（two-way check）。

### 3.4 客户端多样性 / 参考实现

- **ensjs v3**（TypeScript，ensdomains/ensjs-v3）：官方 SDK，基于 viem。
- **ethers.js provider.getResolver + resolveName**。
- **viem**：`normalize`、`getEnsAddress`、`getEnsText`、`getEnsAvatar` 等。
- **RainbowKit / wagmi**：直接调 viem。
- **ens-contracts**（Solidity）：合约集合 + 脚本。

### 3.5 扩展 / 互操作接口

- **EIP-634 Text Record**：任意 string key。常见 key：avatar、url、com.twitter、com.discord。
- **EIP-2304 Multichain Address**：coin type + 地址，支持 BTC / LTC / SOL 等。
- **EIP-181 Reverse Registrar**：addr→name。
- **EIP-5559 Off-chain data write deferral**：配合 CCIP-Read 写入。
- **ENSIP-10 Wildcard Resolution**：`*.alice.eth` 一个 resolver 接管全部子域。
- **DNSSEC Claim**：普通 TLD → ENS。

## 4. 关键代码 / 实现细节

Registry 的 `setSubnodeOwner`（`ENS.sol`，`ensdomains/ens-contracts`）：

```solidity
// ensdomains/ens-contracts/contracts/registry/ENS.sol:~120
function setSubnodeOwner(bytes32 node, bytes32 label, address owner) external authorised(node) {
    bytes32 subnode = keccak256(abi.encodePacked(node, label));
    records[subnode].owner = owner;
    emit NewOwner(node, label, owner);
}
```

PublicResolver 的 `addr(bytes32,uint256)`（`PublicResolver.sol`）：

```solidity
function setAddr(bytes32 node, uint256 coinType, bytes memory a) public authorised(node) {
    _addresses[recordVersions[node]][node][coinType] = a;
    emit AddressChanged(node, coinType, a);
    if (coinType == COIN_TYPE_ETH) {
        emit AddrChanged(node, bytesToAddress(a));
    }
}

function addr(bytes32 node, uint256 coinType) public view returns (bytes memory) {
    return _addresses[recordVersions[node]][node][coinType];
}
```

Name Wrapper 的 `wrap`（简化）：

```solidity
// ensdomains/ens-contracts/contracts/wrapper/NameWrapper.sol:~450
function wrap(bytes calldata name, address owner, address resolver) public {
    (bytes32 parentNode, bytes32 node, bytes32 labelhash) = _decode(name);
    // 转移 ERC-721 所有权到 NameWrapper，然后铸造 ERC-1155
    registrar.reclaim(uint256(labelhash), address(this));
    registrar.transferFrom(msg.sender, address(this), uint256(labelhash));
    _wrap(node, name, owner, 0 /*fuses*/, 0 /*expiry*/);
    if (resolver != address(0)) {
        ens.setResolver(node, resolver);
    }
}
```

## 5. 演进与版本对比

| 版本 | 时间 | 关键变化 |
| --- | --- | --- |
| ENS v1 | 2017-05 | Vickrey 拍卖注册 |
| 永久注册升级 | 2019-05 | `.eth` 年费制 + 7 字符免拍 |
| ERC-721 Registrar | 2020 | `.eth` 名字变 NFT |
| ENS DAO & $ENS | 2021-11 | 治理代币空投 |
| Name Wrapper v1 | 2022-03 | ERC-1155 + Fuse |
| CCIP-Read Resolver | 2022 | L2 names / gas-free |
| Name Wrapper v2 | 2023-10 | 修复若干 Fuse 组合漏洞 |
| ENSv2 宣布 | 2024-04 | L2 Namechain 路线 |
| Namechain Testnet | 2025 | OP Stack + ENS 专用 |
| ENSv2 主网 | 2026（预期） | 注册/续费迁 L2 |

## 6. 实战示例

用 viem 完成「解析 vitalik.eth + 设置 text record」：

```ts
import { createPublicClient, createWalletClient, http, normalize } from "viem";
import { mainnet } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const client = createPublicClient({ chain: mainnet, transport: http() });

// 1. 解析
const addr = await client.getEnsAddress({ name: normalize("vitalik.eth") });
console.log("vitalik.eth =>", addr);

// 2. 反向解析
const name = await client.getEnsName({ address: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045" });
console.log(name); // "vitalik.eth"

// 3. 设置自有名字的 text record（需拥有 name）
const wallet = createWalletClient({
  account: privateKeyToAccount(process.env.PK as `0x${string}`),
  chain: mainnet, transport: http()
});
const resolverAbi = [{ type: "function", name: "setText", inputs: [
  { type: "bytes32" }, { type: "string" }, { type: "string" }
]}] as const;

const resolver = "0x231b0Ee14048e9dCcD1d247744d114a4EB5E8E63"; // Public Resolver
const node = "0x..."; // namehash('alice.eth')
await wallet.writeContract({
  address: resolver, abi: resolverAbi, functionName: "setText",
  args: [node, "com.twitter", "alice"]
});
```

预期：前两步打印 Vitalik 的地址与名字；第三步交易成功后，`client.getEnsText({ name: 'alice.eth', key: 'com.twitter' })` 返回 `"alice"`。

## 7. 安全与已知攻击

1. **Name Wrapper Fuse Bug（2022-11）**：某 Fuse 组合下子域所有权可能被父域强制回收。审计报告 ensdomains/ens-contracts#1xxx 公开，修复后升级到 v2。
2. **Homograph 钓鱼**：`vitalіk.eth`（西里尔 і）被广泛检测；钱包 UI 建议显示 Unicode 名字警告。
3. **Expired Name 抢注**：一些 5 字符热门名字过期后被抢注并挂高价，原持有者只能在 21 日 premium 窗口内高价赎回。
4. **Resolver Phishing**：钱包前端若使用第三方 Resolver（如未审计 Rainbow 合约），恶意合约可返回任意地址。
5. **反向解析不对称**：只设 reverse 未设 forward，钱包显示 `alice.eth` 但实际不是 Alice 的地址，需 two-way check 保护。
6. **ENS DAO 治理攻击面**：DAO 控制 TLD 合约升级，若被恶意提案通过可重定向所有 `.eth`。Timelock + 社区监督是主要防御。

## 8. 与同类方案对比

| 维度 | ENS | Unstoppable Domains | SpaceID | Lens Handles | Farcaster fname |
| --- | --- | --- | --- | --- | --- |
| 所有权 | 链上 NFT | ERC-721 | ERC-721（多链） | ERC-721 | 链下 + fid |
| 续费 | 年费 | 永久（一次付款） | 年费 | 免 | 免 |
| 标准兼容 | EIP-137 | 自定义 | 自定义 | 自定义 | Farcaster protocol |
| 反向解析 | 支持 | 支持 | 支持 | 支持 | 支持 |
| 钱包集成 | 普及 | 中 | 中 | Lens 内 | Farcaster 内 |
| DNS 兼容 | DNSSEC claim | 部分 | 部分 | 否 | 否 |
| L2 支持 | CCIP-Read / ENSv2 | 多链 | BNB/Arb | Polygon | OP |

ENS 的优势在于 **标准化 + 钱包兼容性 + DAO 治理**；Unstoppable Domains 以「永久」抓住用户但缺乏 L1 钱包原生支持。SpaceID 等后来者在 BNB 链上占主导。

## 9. 延伸阅读

- **官方**：docs.ens.domains、https://ens.domains/blog/、ensdomains/ens-contracts GitHub。
- **EIP / ENSIP**：EIP-137、EIP-181、EIP-634、EIP-2304、EIP-3668、EIP-5559、ENSIP-9/10/11。
- **博客 / 深度**：a16z《Naming the Internet》、Paradigm《The State of ENS》、Nick Johnson Twitter。
- **视频**：ETHGlobal ENS 研讨会、DevCon ENS 专场。
- **数据**：dune.com/makoto/ens、v3.ens.domains/analytics。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| ENS | Ethereum Name Service | 以太坊域名系统 |
| Namehash | Namehash | 递归哈希命名空间 |
| Registry | ENS Registry | 根所有权合约 |
| Resolver | Resolver | 记录查询合约 |
| Registrar | Registrar | 发放子域的注册器 |
| Name Wrapper | Name Wrapper | 将名字转 ERC-1155 并引入 Fuse |
| Fuse | Fuse | 单调 burn 的权限位 |
| CCIP-Read | EIP-3668 | 链下读取协议 |
| Reverse Record | Reverse Record | addr → name 映射 |
| Controller | Controller | 注册流程入口合约 |
| Namechain | Namechain | ENSv2 L2（基于 OP Stack） |
| Text Record | Text Record | 键值对记录 |
| Contenthash | Contenthash | IPFS/IPNS 内容指针 |

---

*Last verified: 2026-04-22*
