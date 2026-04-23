---
title: Bitstamp 热钱包被盗 19,000 BTC（2015-01-04, ~$5M 按当时价）
module: 08-security-incidents
category: CEX-hack
date: 2015-01-04
loss_usd: 5000000
chain: [Bitcoin]
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://www.reuters.com/article/us-bitstamp-cyberattack-idUSKBN0KI09320150109
  - https://www.coindesk.com/markets/2015/07/01/bitstamp-hackers-tried-to-steal-19-million-in-phishing-attack/
  - https://arstechnica.com/information-technology/2015/07/leaked-bitstamp-incident-report-details-the-hack-that-took-down-the-bitcoin-exchange/
  - https://bitcoinmagazine.com/business/leaked-report-bitstamps-bitcoin-hack-not-sophisticated-affair
tx_hashes:
  - 攻击聚合地址 1L2JsXHPMYuAa9ugvHGLwkdstCPUDemNCf（社区分析，Bitstamp 官方未直接确认）
---

# Bitstamp 热钱包被盗 19,000 BTC

> **TL;DR**：2015 年 1 月 4 日，欧洲头部 BTC 交易所 Bitstamp 的热钱包被攻击者清空，损失约 18,866 BTC，按当时价约 500 万美元。根因为鱼叉式钓鱼（spear phishing）：攻击者在 2014 年 11 月至 12 月间连续向 Bitstamp 6 名关键员工发送定制化社工邮件（Skype、邮件、LinkedIn 附件），最终诱导系统管理员 Luka Kodric 在其工作电脑上打开含宏的 .doc 文件，部署后门控制热钱包签名机。该事件因泄露的内部事故报告成为 Web3 社工攻击的经典教学案例。

## 1. 事件背景

- **主体**：Bitstamp 成立于 2011 年，注册于斯洛文尼亚，后迁至英国、卢森堡；事件发生时是欧洲最大 BTC/美元交易所之一。
- **时间轴**：
  - 2014-11-04：攻击者开始向 Bitstamp 6 名员工发起社工接触（Skype、邮件、LinkedIn）。
  - 2014-12-11：系统管理员 Luka Kodric 在 Skype 上接受名为 "ihatebitcoin@hushmail.com" 发送的含宏 .doc 文件（伪装为 Ubuntu 开发者社区申请表）。
  - 2014-12-11 至 2015-01-04：攻击者横向移动，获取 Bitstamp 的 LastPass 密钥、服务器 SSH 凭证、热钱包 wallet.dat 与 2FA 种子。
  - 2015-01-04 02:17 UTC：首笔异常提现。
  - 2015-01-05：Bitstamp 暂停所有服务。
  - 2015-01-09：对外公告事件，交易所恢复。
  - 2015-07：内部事故报告（由斯洛文尼亚律所撰写）被匿名泄露给 Reddit，详细技术过程曝光。
- **发现过程**：热钱包出金自动告警触发；运维在 ~30 分钟内暂停服务。

## 2. 事件影响

- **直接损失**：18,866 BTC（约 $5.1M 按当时价 ~$270）。
- **受害方**：Bitstamp 自有资金承担全部损失，用户资金完整；但交易所服务中断 8 天。
- **连带影响**：
  - 推动交易所普遍采用"热钱包仅保留 1–5% 总资产"的行业惯例。
  - BitGo 多签方案在事件后被 Bitstamp 新架构采纳。
  - 社工攻击成为 CEX 安全培训核心内容。
- **资金去向**：被盗 BTC 流入多个混币服务，部分地址与后续朝鲜/东欧黑客活动关联，但归因未公开正式报告。

## 3. 技术根因

- **漏洞分类**：Social-Eng（主因） + Key-Mgmt + Endpoint-Security。
- **根因链**：
  1. 攻击者定向调研 6 名员工的公开信息（LinkedIn 职位、兴趣）。
  2. 对 Luka Kodric 使用"Punk Rock Club Ljubljana 会员申请表"为诱饵，因其公开简历中有相关兴趣。
  3. 含恶意 VBA 宏的 .doc 在其公司 Windows 机器上打开，宏下载后门（远程访问木马，具体家族未公开）。
  4. 后门窃取其 KeePass/LastPass 主密码、Bitstamp 内部 Wiki 凭证、SSH key passphrase。
  5. 攻击者登录热钱包签名服务器，导出 wallet.dat 与 2FA 种子文件 `bitstamp.gauth`。
  6. 2015-01-04 在完全控制签名凭证后，在 5 分钟内将热钱包 18,866 BTC 全部转出。
- **关键伪代码（签名机权限配置缺陷）**：

```bash
# 事故报告披露的 Bitstamp 当时签名机配置（简化还原）
# 缺陷：2FA 种子文件与 wallet.dat 存放在同一台机器，且被 root 进程读取
/opt/bitstamp/wallet/wallet.dat       # bitcoind 主钱包
/opt/bitstamp/wallet/bitstamp.gauth   # Google Authenticator HOTP 种子
# 一旦攻击者获得 root，2FA 等同虚设
```

- **攻击步骤与 tx**：
  1. 初始钓鱼（2014-11-04 起）。
  2. 植入后门（2014-12-11）。
  3. 横向移动（2014-12-11 至 2015-01-03）。
  4. 集中出金（2015-01-04 02:17–02:30 UTC）：主聚合地址 `1L2JsXHPMYuAa9ugvHGLwkdstCPUDemNCf`（社区归因）。
  5. 混币与跨交易所转移（持续数月）。
- **审计盲区**：Bitstamp 当时无员工终端 EDR；无钓鱼邮件沙箱；2FA 种子未隔离。

## 4. 事后响应

- **项目方**：
  - 2015-01-09 恢复服务，热钱包架构改造为 BitGo 多签。
  - 引入 HSM 托管 2FA 种子。
  - 员工强制安全培训与钓鱼演练。
- **资产追回**：未公开追回信息。
- **执法**：斯洛文尼亚、英国、美国 FBI 均展开调查；未公开嫌疑人起诉结果。
- **行业连锁**：Kraken、Coinbase、Bitfinex 等同行普遍加固 2FA 种子与签名机隔离。

## 5. 启发与教训

- **对开发者 / CEX**：
  - 签名机必须专机专用，不安装办公软件，不接收任何办公邮件。
  - 2FA 种子不可与 wallet.dat 同机；HOTP/TOTP 种子应放硬件 Token（YubiKey）。
  - 热钱包应设单笔 / 每小时提现上限 + 人工审核门槛。
- **对安全团队**：
  - 钓鱼演练必须定期、真实（含定制化诱饵）。
  - EDR + 邮件沙箱是基础设施。
- **对用户**：事件证明即使一线交易所也可能被社工击穿；大额资产应自托管。
- **对行业**：泄露的事故报告成为极少数公开的 CEX incident post-mortem，教学价值巨大。

## 6. 参考资料

- Bitstamp Incident Report（泄露版本）：Reddit r/Bitcoin 2015-07 存档
- Reuters：《Bitstamp says losses from hack less than $5M》2015-01-09
- CoinDesk：《Bitstamp Hackers Tried to Steal $19 Million in Phishing Attack》2015-07
- Ars Technica：《Leaked Bitstamp incident report details the hack》2015-07
- Bitcoin Magazine：《Leaked Report: Bitstamp's Bitcoin Hack Not Sophisticated Affair》

---

*Last verified: 2026-04-23*
