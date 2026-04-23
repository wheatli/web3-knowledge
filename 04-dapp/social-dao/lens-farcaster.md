---
title: Lens Protocol v2 与 Farcaster（Hubs + Frames）
module: 04-dapp/social-dao
priority: P1
status: DRAFT
word_count_target: 5000
last_verified: 2026-04-22
primary_sources:
  - https://docs.lens.xyz/
  - https://github.com/lens-protocol/core
  - https://docs.farcaster.xyz/
  - https://github.com/farcasterxyz/protocol
  - https://docs.farcaster.xyz/developers/frames/spec
---

# 去中心化社交：Lens Protocol v2 与 Farcaster（Hubs + Frames）

> **TL;DR**：Lens 与 Farcaster 是 2022 年以来两条最活跃的去中心化社交路线。Lens Protocol（Aave 团队）把社交图谱完全上链：Profile 是 NFT、Post/Comment/Mirror 是链上 Publication、Follow 也是 NFT。Lens v2（2023）引入 Actions、Collect Modules、Open Actions 与 Sponsored Gas。Farcaster（Dan Romero / Varun Srinivasan）采取「身份上链，内容链下」的混合模型：Optimism 上的 ID Registry + Key Registry 管理 fid 和签名密钥，内容由用户运行的 Hub 网络 gossip；2024 年 Frames 规范把 cast 升级为可交互 canvas，引爆 Farcaster 月活到 50 万+。本篇对比两套协议的数据模型、经济模型、身份系统、Hubs/Frames 规范与 SDK 实战。

## 1. 背景与动机

Web2 社交网络的核心问题：数据锁在平台方 → 用户无法迁移、平台算法黑箱、创作者经济被平台税收。加密社交协议的目标是实现 **可组合身份 + 可移植社交图谱 + 去中介化收益**。两条路线：
- **纯链上**（Lens）：所有社交对象都是链上资产，可被任意 DApp 读写。优点是无信任；缺点是成本高、内容上链困难。
- **混合式**（Farcaster）：链上放身份与稀缺权限，链下放内容。兼顾成本与去中心化。

Lens 2022 年 2 月上线 Polygon（为省 gas），v1 以 SocialGraph NFT + Collect NFT 模型运行；2023-06 发布 v2 版本并改进 Open Action 架构；2024 年 Lens Chain（基于 ZK Stack）推出，v3 正在规划。Farcaster 2021 年末由前 Coinbase 高管 Dan Romero 启动，2022 年春 Beta、2023 年逐步开放、2024-01 Frames 规范上线后迎来爆发式增长，2024-04 基金会推出 Warps 和 Warpcast（官方客户端）。

## 2. 核心原理

### 2.1 形式化定义：社交协议抽象

社交协议可抽象为 5 类对象：
- **Identity**：用户身份（地址绑定、名字）。
- **Profile**：持久化个人资料（bio、avatar）。
- **Publication / Cast**：内容本体（post、repost、reply）。
- **Social Graph**：follow / subscribe 关系。
- **Interaction**：点赞、收藏、打赏、评论。

Lens：所有 5 类对象都是链上状态。`Profile` = ERC-721；`Publication` = 链上 struct；`Follow` = ERC-721（一人 follow 自动铸一个 Follow NFT）；`Collect` = ERC-721（收藏/购买内容）。

Farcaster：Identity（fid）+ Key = 链上；Cast / Reaction / Link（follow）= 链下事件（Hub），签名验证通过后被 gossip。

### 2.2 Lens v2 核心合约

- **LensHub.sol**：主入口，Profile NFT 所有权 + publication ID 自增。
- **CollectNFT / FollowNFT**：每次首次 collect/follow 会部署或铸造一个 NFT。
- **Modules**：拔插式插件
  - **Follow Module**：限制条件（付费 follow / NFT gated）。
  - **Reference Module**：限制谁能评论/转发。
  - **Action Module (v2)**：任意链上 action（例：collect、tip、vote），替代 v1 的 Collect Module。
  - **Open Actions**：发表 publication 时指定多个 action，使 cast 成为可组合 dApp 入口。
- **Gasless**：Lens 官方 Relayer 补贴 gas；Profile 持有者签名，Relayer 提交。

### 2.3 Farcaster Hubs + Registry

**On-chain（Optimism）**：
- `IdRegistry.sol`：`register()` 铸造 fid（fid 是 uint64 自增）。
- `KeyRegistry.sol`：在 fid 下注册 signer key（Ed25519）；Key type 区分 app（Warpcast、Supercast 等）。
- `StorageRegistry.sol`：用户按年购买 storage units，决定 Hub 可保存其多少条 cast（默认 1 unit = 5000 casts）。
- `Bundler.sol`：把 Register + Key + Storage 组合为一笔交易。

**Off-chain Hubs**：P2P 节点，保存消息集合（cast、reaction、link、userdata），通过 gossip 最终一致。Hub 之间通过 gRPC + libp2p 同步，状态通过 Merkle Trie Sync 对齐。任意用户可运行 Hub。

消息是 signed protobuf：
```
Message {
  data: MessageData {
    type, fid, timestamp, network, body
  },
  hash: keccak256(data),
  signature: ed25519(hash, signerKey),
  signer: pubkey
}
```

Hub 校验签名（确认 signer 在 fid 的 KeyRegistry 里且未 revoke），时间戳（10min 之内）后接受。

### 2.4 Farcaster Frames 规范

Frame 把一条 cast 升级为 server-rendered interactive UI。规范（2024-01）定义 meta tags：

```html
<meta property="fc:frame" content="vNext" />
<meta property="fc:frame:image" content="https://example.com/img.png" />
<meta property="fc:frame:button:1" content="Vote Yes" />
<meta property="fc:frame:button:2" content="Vote No" />
<meta property="fc:frame:post_url" content="https://example.com/action" />
```

当用户在 Warpcast 点击按钮，客户端发送 `FrameActionPayload`（含 fid、signed message、button index、input）到 `post_url`。server 返回新的 meta tag 刷新画面。Frames v2（2024-Q3）支持 wallet transaction（`fc:frame:button:action=tx`），直接在 cast 内发起链上交易。

Frame 是强烈采用激励：一个 cast 就可以是完整 NFT mint UI、游戏、swap、投票；无需下载 DApp。

### 2.5 关键参数（截至 2026-04）

| 参数 | Lens v2 | Farcaster |
| --- | --- | --- |
| L1 / L2 | Polygon PoS → Lens Chain | Optimism |
| Profile / Identity Registry | LensHub（NFT）| IdRegistry |
| 身份成本 | 早期空投 + 付费 claim handle | $3 /年 storage + gas |
| 内容位置 | 全链上 | Hub（链下） |
| 内容成本 | 若不走 Gasless 则 gas | storage unit |
| 原生 token | LENS（未正式发）  | 无（Warps 积分） |
| 主流 App | Hey.xyz、Orb、Phaver | Warpcast、Supercast |

### 2.6 边界条件与失败模式

- **Lens 发文成本**：即便 gasless，relay 成本由 Lens 公司承担，长期可持续性待证明。
- **Farcaster Hub 同步分叉**：不同 Hub 可能因消息丢弃阈值不同看到不同内容，需要 Merkle Trie Sync 收敛。
- **Signer Key 泄露**：若 app signer key 泄露，可冒充用户发 cast；用户在 KeyRegistry `removeFor` 吊销。
- **Frame 服务器宕机**：Frame 服务器无响应则 UI 空白，client 会退化为普通 cast；需要缓存 + 备用端点。
- **Frame 授权滥用**：恶意 Frame 在 tx action 中诱导签 approve，需要钱包 + client 强化预审。

## 3. 架构剖析

### 3.1 分层视图

**Lens**：
1. L1 合约层（LensHub + Modules）。
2. Lens Protocol SDK / GraphQL API（由 Lens Labs 运行，准中心化 index）。
3. Clients（Hey / Orb / Phaver / Lenster）。
4. 外部 DApp 读 Lens GraphQL + 链上状态。

**Farcaster**：
1. OP Registry（Id/Key/Storage）。
2. Hubs P2P 网络。
3. 客户端 SDK（@farcaster/hub-web、auth-kit）。
4. Warpcast / Supercast / Jam 等 client。
5. Frames 生态（mint.fun / paragraph.xyz / Bracket.game）。

### 3.2 核心模块清单

| 模块 | 协议 | 职责 |
| --- | --- | --- |
| LensHub | Lens | Profile / Publication 状态 |
| ModuleGlobals | Lens | 注册可用 Module |
| FollowNFT / CollectNFT | Lens | 社交关系与收藏凭证 |
| Lens GraphQL API | Lens Labs | 链上 + 链下索引 |
| IdRegistry | Farcaster | fid 铸造 |
| KeyRegistry | Farcaster | signer key 管理 |
| StorageRegistry | Farcaster | 年费制 storage |
| Hub (Hubble / Neynar) | Farcaster | 消息存储 + gossip |
| Frame Server | Farcaster | 动态 UI 渲染 |
| Auth Kit | Farcaster | SIWE-like 登录 |

### 3.3 数据流 / 生命周期

**Lens：一条 post 从发布到展示**：
1. 用户在 Hey 前端撰写内容，SDK 上传 metadata 到 Arweave/IPFS。
2. 用户通过 Lens Gasless relay 签名 `post(PostParams)`。
3. Relay 支付 gas，调用 `LensHub.post`，生成 `(profileId, pubId)`，触发 `PostCreated` 事件。
4. Lens GraphQL indexer 监听事件并同步 metadata URL。
5. 其他用户在 feed 中通过 GraphQL query 拿到这条 post + media URL。

**Farcaster：一条 cast 从发布到跨 Hub 同步**：
1. 用户在 Warpcast 写内容，客户端用 signer key 签 `MessageData { type: CastAdd, body: CastAddBody }`。
2. 客户端提交给 Warpcast 的 Hub（或任意 Hub）。
3. Hub 校验 signer 在 KeyRegistry 中有效、storage 未超限、签名有效。
4. Hub 广播到其它 peers，所有 Hub 的状态 Merkle Trie 收敛。
5. 其它 client（Supercast）从自选 Hub 拉取 feed。

### 3.4 客户端多样性 / 参考实现

- **Lens**：lens-protocol/core（Solidity）、@lens-protocol/client（TS SDK）、Hey.xyz（开源 React 客户端）。
- **Farcaster**：farcasterxyz/hubble（Rust/TS Hub 实现）、@farcaster/hub-web（TS SDK）、Neynar（托管 Hub + SDK）、Warpcast（闭源）、Supercast、Paragraph。

Neynar 目前是 Farcaster 生态上最大的托管 Hub + API 提供商；对开发者简便但引入中心化风险。

### 3.5 扩展 / 互操作接口

- **Lens Open Actions**：publication 内置第三方 action 合约（例：Bonsai Quests、Decent Collect）。
- **Lens Profile Transfer**：ERC-721 可转移账户。
- **Farcaster Sign In With**：基于 fid + Ed25519 signer 的 SSO。
- **Farcaster Channels（2024）**：按主题分组的 cast 集合（~subreddit）。
- **Frames v2**：支持 `tx` action，Sponsored Txn；与 MetaMask / Rainbow 集成。
- **跨协议 graph**：Airstack / Pinata 等索引同时支持 Lens + Farcaster + ENS graph。

## 4. 关键代码 / 实现细节

Lens v2 `LensHub.post`（`lens-protocol/core@v2`，`contracts/LensHub.sol:~300`）：

```solidity
// lens-protocol/core/contracts/LensHub.sol (v2) — 简化
function post(Types.PostParams calldata postParams)
    external override whenPublishingEnabled returns (uint256)
{
    _validateCallerIsProfileOwnerOrDelegatedExecutor(postParams.profileId);
    return PublicationLogic.post({
        postParams: postParams,
        transactionExecutor: msg.sender,
        _publicationCounter: _profileCounter[postParams.profileId],
        _pubByIdByProfile: _pubByIdByProfile,
        _actionModuleWhitelistData: _actionModuleWhitelistData,
        _referenceModuleWhitelisted: _referenceModuleWhitelisted
    });
}
```

Farcaster Hub 消息校验（`farcasterxyz/hubble`，`apps/hubble/src/storage/engine/index.ts`）核心流程：

```ts
// Hubble 消息验证（简化）
async function mergeMessage(message: protobufs.Message): Promise<Result<number>> {
  // 1. 结构/字段校验
  const validated = validateMessage(message);
  if (validated.isErr()) return err(validated.error);
  // 2. 签名校验
  const signerValid = await verifyMessageHashSignature(message, message.signer);
  if (!signerValid.ok) return err("bad signature");
  // 3. 确认 signer 在 KeyRegistry 中 active
  const keyActive = await this.l1Client.isKeyActive(message.data.fid, message.signer);
  if (!keyActive) return err("signer revoked");
  // 4. 存储
  return this._storage.merge(message);
}
```

Frame server 处理 POST（Next.js 伪代码）：

```ts
// Farcaster Frame handler
export async function POST(req: Request) {
  const { untrustedData, trustedData } = await req.json();
  const valid = await verifyFrameMessage(trustedData.messageBytes);
  if (!valid) return new Response("invalid", { status: 400 });
  const fid = untrustedData.fid;
  const button = untrustedData.buttonIndex;
  // 记录投票
  await recordVote(fid, button);
  return new Response(`
    <meta property="fc:frame" content="vNext" />
    <meta property="fc:frame:image" content="${resultImg(fid)}" />
  `, { headers: { "content-type": "text/html" } });
}
```

## 5. 演进与版本对比

| 事件 | 时间 | 协议 | 影响 |
| --- | --- | --- | --- |
| Lens v1 | 2022-02 | Lens | Profile/Publication 上链 |
| Farcaster Beta | 2022 夏 | Farcaster | 邀请制 |
| Lens v2 | 2023-06 | Lens | Open Actions / Sponsored |
| Farcaster 开放 | 2023-10 | Farcaster | 公开注册 |
| Frames v1 规范 | 2024-01 | Farcaster | 爆发增长 |
| Farcaster 月活破 50 万 | 2024-Q1 | Farcaster | — |
| Frames v2（tx） | 2024-Q3 | Farcaster | 链上交互直接嵌入 |
| Lens Chain 上线 | 2024-Q4 | Lens | 自有 L2（ZK Stack） |
| Lens v3（规划） | 2026 | Lens | 模块化重构 |

## 6. 实战示例

**Farcaster：用 Neynar API 发一条 cast 并读 feed**

```ts
import axios from "axios";
const NEYNAR_KEY = process.env.NEYNAR_KEY!;

// 1. 发一条 cast（需要用户 signer UUID，参见 Neynar Managed Signer）
await axios.post(
  "https://api.neynar.com/v2/farcaster/cast",
  { signer_uuid: process.env.SIGNER_UUID, text: "hello from script" },
  { headers: { api_key: NEYNAR_KEY } }
);

// 2. 读取 /trending feed
const resp = await axios.get(
  "https://api.neynar.com/v2/farcaster/feed/trending?limit=5",
  { headers: { api_key: NEYNAR_KEY } }
);
console.log(resp.data.casts.map(c => c.text));
```

**Lens：用 SDK 创建 Publication**

```ts
import { LensClient, production } from "@lens-protocol/client";
const client = new LensClient({ environment: production });
const login = await client.authentication.login({ address: ADDR, signature: SIG });
await client.publication.postOnchain({
  contentURI: "ar://QmXyz",
  openActionModules: [
    { simpleCollectOpenAction: { collectLimit: 100 } }
  ]
});
```

预期：Farcaster 脚本输出最近 5 条热门 cast 文本；Lens 调用后 Hey feed 里可见新 post 并含 collect 按钮。

## 7. 安全与已知攻击

1. **Lens Collect Phishing**：恶意 Collect Module 在 collect 时索取过多 ERC-20 approval；钱包需要明确展示 allowance。
2. **Farcaster Signer Leakage**：2024-02 某 client 的 signer 请求被钓鱼站点截获，受影响用户通过 KeyRegistry `removeFor` revoke。
3. **Frame Spoofing**：Frame message 中 `untrustedData` 可伪造；开发者必须以 `trustedData` 的签名验证后为准（`verifyFrameMessage`）。
4. **Storage DoS**：Farcaster 存储有限，spam bot 可通过 storage rental 消耗 Hub 资源；Hub 可设置速率限制。
5. **Lens Profile NFT Theft**：若 profile NFT 被盗，社交图谱被接管。Lens 支持 Profile Guardian（冷钱包守护）。
6. **Open Action 恶意逻辑**：任意外部合约被注入 publication，可能 re-enter；LensHub 已引入 action whitelist。

## 8. 与同类方案对比

| 维度 | Lens v2 | Farcaster | Nostr | Friend.tech |
| --- | --- | --- | --- | --- |
| 身份 | ERC-721 Profile | fid + Ed25519 keys | pubkey | 钱包地址 + shares |
| 内容存储 | 链上 + IPFS/Arweave | Hub（链下） | Relay（链下） | 链下 |
| 抗审查 | 强（链上） | 中（Hub 运营者） | 强（relay）| 弱（集中） |
| 可组合性 | 高（Open Actions） | 中（Frames） | 低 | 低 |
| Gas 成本 | 中（Lens Chain 后↓） | 低（年费）| ≈ 0 | 中 |
| Client 多样性 | Hey / Orb / Phaver | Warpcast / Supercast / Jam | damus / primal | 官方 |
| 代表用户 | 早期空投 | Web3 核心圈 | 中本聪文化 | 加密交易圈 |

## 9. 延伸阅读

- **官方**：docs.lens.xyz、docs.farcaster.xyz。
- **GitHub**：lens-protocol/core、farcasterxyz/protocol、farcasterxyz/hubble、neynarxyz/farcaster-api。
- **博客**：Lens Mirror（@lensprotocol）、Farcaster Blog (merkle manufactory)、Dan Romero Twitter、Varun Srinivasan notes、Paragraph 系列 essays。
- **Frames 深度**：Jesse Pollak《Frames v2》、Ben Roy《A Taxonomy of Farcaster Frames》。
- **数据**：warpcast.com/~/dev、lensfrens.xyz、OpenRank、Airstack dashboards。

## 10. 术语表

| 术语 | 英文 | 释义 |
| --- | --- | --- |
| Profile | Lens Profile NFT | 社交身份 NFT |
| Publication | Lens Publication | 内容结构（post/comment/mirror） |
| Collect | Collect | 收藏/购买内容（铸造 NFT） |
| Follow NFT | Follow NFT | 关注即铸 NFT |
| Open Action | Open Action | 任意可插拔链上行为 |
| Gasless | Sponsored Tx | 由 relayer 代付 gas |
| fid | Farcaster ID | Farcaster 身份编号 |
| Hub | Farcaster Hub | 去中心化消息节点 |
| Signer Key | Signer Key | Ed25519 签名密钥 |
| Storage Unit | Storage Unit | 用户存储额度 |
| Cast | Cast | Farcaster 的 post |
| Frame | Frame | 可交互 cast 界面 |
| Warps | Warps | Warpcast 内激励积分 |
| SIWE | Sign In With Ethereum | 登录协议 |

---

*Last verified: 2026-04-22*
