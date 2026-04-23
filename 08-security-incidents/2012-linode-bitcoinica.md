---
title: Linode 云主机入侵致 Bitcoinica/Slush Pool 等丢失 ~46k BTC（2012-03-01, ~$230k 按当时价）
module: 08-security-incidents
category: Supply-Chain
date: 2012-03-01
loss_usd: 230000
chain: [Bitcoin]
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://blog.linode.com/2012/03/02/security-incident-update/
  - https://bitcoinmagazine.com/technical/bitcoinica-hacked-a-timeline
  - https://bitcointalk.org/index.php?topic=66979.0
  - https://arstechnica.com/tech-policy/2012/03/bitcoins-worth-228000-stolen-from-customers-of-hacked-webhost/
tx_hashes:
  - Bitcoinica 被盗 43,000 BTC 聚合目的地址 1MAazCWMydsQB5ynYXqSGQDjNQMN3HFmEu（社区分析，待核实）
  - Slush Pool 被盗 tx hash 未公开
---

# Linode 云主机入侵致 Bitcoinica/Slush Pool 等丢失 ~46k BTC

> **TL;DR**：2012 年 3 月 1 日，云主机商 Linode 的客服面板遭社工入侵，攻击者获得 8 个特定托管 Bitcoin 相关服务的虚拟机的 root 权限，从 Bitcoinica（~43,000 BTC）、Slush Pool（~3,094 BTC）、Faucet 与 Gavin Andresen 个人钱包等 8 家受害方共盗走约 46,000 BTC。这是 Web3 历史上第一起明确定性为"云供应链（cloud supply-chain）"的大规模入侵：热钱包私钥以明文形式保存在 VPS 磁盘上，云服务商的客服访问权限成为单点风险。

## 1. 事件背景

- **主体**：
  - Linode：美国老牌 VPS 托管商，2012 年在加密社区被广泛使用，因其按小时计费、Xen 虚拟化、API 完备。
  - Bitcoinica：由 Zhou Tong（当时 17 岁）创立的 BTC 杠杆交易平台，2012 年初为全球领先的 BTC 保证金交易所之一。
  - Slush Pool（现为 Braiins Pool）：世界上最早的 BTC 矿池。
- **时间轴**：
  - 2012-02-28 深夜：攻击者通过 Linode 客服支持入口，以社会工程学手段获得管理员后台访问权。
  - 2012-03-01（UTC）：攻击者定向登录 Bitcoinica、Slush Pool 等 8 台 VPS，使用 root 权限直接读取本地钱包文件 `wallet.dat` 并签名转账。
  - 2012-03-02：Linode 在官方博客发布 Security Incident Update，确认客户面板被入侵。
  - 2012-03-02：Bitcoinica 公告损失 43,554 BTC，同时 Slush Pool、Faucet（由 Gavin Andresen 维护）、TradeFortress 的 BC3 服务等陆续披露损失。
- **发现过程**：Bitcoinica 运维发现钱包余额异常归零，随即联系 Linode。Linode 审计客服工单记录定位入侵。

## 2. 事件影响

- **直接损失**（按 2012-03-01 BTC ~$5 计）：
  - Bitcoinica：约 43,000 BTC（~$215k）
  - Slush Pool：约 3,094 BTC（~$15k）
  - Faucet / Gavin Andresen 个人：约 50 BTC
  - 其他 5 位 Linode 客户（身份部分未公开）：合计数百 BTC
  - 合计约 46,000 BTC（~$230k）
- **受害方**：Bitcoinica 用户（保证金账户）、Slush Pool 矿工的待结算余额、BC3 / Faucet 用户。
- **连带影响**：
  - Bitcoinica 此后再未恢复元气，2012-05 和 2012-07 又连续两次被黑（MtGox 账户被盗、LastPass 账号被盗），最终停运并进入破产程序。
  - 加密社区开始系统性讨论"不要把热钱包私钥放在 VPS 上"，推动了离线签名服务（如 Armory cold signing）的流行。
- **资金去向**：Bitcoinica 被盗 BTC 大部分流入混币服务与早期 CEX，部分地址被链上分析社区长期跟踪，但未公开最终归宿。

## 3. 技术根因

- **漏洞分类**：Supply-Chain + Social-Eng + Key-Mgmt。
- **根因链条**：
  1. **Linode 客服面板的权限过大**：客服凭一个后台账号即可 reset 客户 VM 的 root 密码并通过 Lish（Linode Shell）进入终端。
  2. **社工攻击绕过客服验证**：攻击者冒充 Linode 员工/客户身份，诱导客服重置密码（具体剧本未公开）。
  3. **受害服务的钱包私钥以明文 `wallet.dat` 形式存放在 VPS 根文件系统**：无 HSM、无硬件签名、无异地热备。
  4. **无网络层防护**：VPS 无 IP allowlist，API 签名机直连公网。
- **受损模块**：
  - Linode 客服后台（Linode Manager + Lish），闭源，无公开修复 commit。
  - Bitcoinica 服务器端（PHP + bitcoind），代码曾短暂开源在 GitHub（`bitcoinica/bitcoinica`，当前仓库已删除，归档见社区镜像）。
- **关键伪代码**（2012 年 Bitcoinica 热钱包签名模式还原）：

```python
# 典型 2012 年交易所热钱包签名流程（简化）
import subprocess
def send_btc(to_addr, amount):
    # 缺陷：bitcoind RPC 无密码或弱密码，wallet.dat 明文存 VPS
    cmd = f"bitcoin-cli sendtoaddress {to_addr} {amount}"
    return subprocess.check_output(cmd, shell=True)
# 一旦 root 被取得，攻击者可以直接 bitcoin-cli dumpprivkey / sendtoaddress
```

- **攻击步骤**：
  1. 社工入侵 Linode 客服面板。
  2. 批量查询 Linode 客户库，定位与 Bitcoin/挖矿相关的 VM。
  3. 对目标 VM 重置 root 密码，通过 Lish 登录。
  4. `bitcoin-cli dumpprivkey` 或直接读取 `~/.bitcoin/wallet.dat` 后用自己的 bitcoind 加载签名。
  5. 将 BTC 转出至攻击者钱包；Bitcoinica 大额去向聚合到少数地址。
- **tx hash**：Bitcoinica 主聚合地址 `1MAazCWMydsQB5ynYXqSGQDjNQMN3HFmEu`（社区分析，待核实）；Slush Pool 未公开具体 tx。

## 4. 事后响应

- **Linode**：
  - 立即关停客服面板异常会话，强制所有员工改密。
  - 加强客服工单身份验证规则（但 2013-04 Linode 再次被黑，说明改进不彻底）。
  - 公开博客承诺"不会再发生"。
- **Bitcoinica**：Zhou Tong 与联合创始人承担部分赔付，后被 Intersango / Bitcoin Consultancy 收购；2012 年 5 月 MtGox 账户再次被盗 18,547 BTC；7 月 LastPass 被盗 40,000 BTC；最终进入诉讼清算。
- **Slush Pool**：由 Marek "Slush" Palatinus 从矿池运营利润补偿矿工，未让矿工承担损失；同时宣布永久不使用 Linode。
- **执法**：未公开刑事起诉结果；Bitcoinica 用户集体诉讼在 2012–2013 年在美国进行，部分赔付达成和解。

## 5. 启发与教训

- **对开发者**：
  - 热钱包私钥绝不可以明文存放在云主机文件系统；应使用 HSM、AWS KMS、YubiHSM 或离线签名机。
  - 热钱包应设每小时 / 单笔出金上限，超额需冷钱包二次签名。
- **对云服务商**：客服后台权限应最小化，敏感操作（如 root reset）必须多因素验证 + 客户端外带确认。
- **对用户**：选择托管服务时应关注供应商的安全事件历史与事件响应能力。
- **行业意义**：该事件是"云 + 加密"供应链风险的第一课，直接催生了后来多签冷钱包（BitGo 2013 成立）、HSM 托管（Bitfury Clarke）等基础设施。

## 6. 参考资料

- Linode 官方公告：`blog.linode.com/2012/03/02/security-incident-update/`
- Bitcoin Magazine：《Bitcoinica Hacked: A Timeline》
- Ars Technica：《Bitcoins worth $228,000 stolen from customers of hacked webhost》
- bitcointalk 主题：Bitcoinica 官方事件串（主题 66979）
- Gavin Andresen 个人邮件列表披露（bitcoin-dev 2012-03）
- 2012-05 后续事件：Bitcoinica MtGox Account 被盗（bitcointalk 主题 81045）

---

*Last verified: 2026-04-23*
