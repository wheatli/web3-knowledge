---
title: Mt.Gox 首次数据库泄露与价格闪崩（2011-06-19, ~$8.75M 按当时价）
module: 08-security-incidents
category: CEX-hack
date: 2011-06-19
loss_usd: 8750000
chain: [Bitcoin]
severity: Tier-3
status: DRAFT
last_verified: 2026-04-23
primary_sources:
  - https://en.bitcoin.it/wiki/MyBitcoin
  - https://www.wired.com/2011/06/bitcoin-flaw/
  - https://bitcointalk.org/index.php?topic=19139.0
  - https://arstechnica.com/tech-policy/2011/06/bitcoin-prices-plummet-on-hacked-exchange/
tx_hashes:
  - 攻击链上交易 hash 未公开（2011 年 Mt.Gox 内部账本交易未上链广播）
---

# Mt.Gox 首次数据库泄露与价格闪崩

> **TL;DR**：2011 年 6 月 19 日，当时承担全球约 70% BTC 交易量的 Mt.Gox 交易所遭入侵，攻击者通过一个被拖库的管理员账户接管高额权限账户，在站内将大量 BTC 以 1 美分的价格倾销，导致 BTC 短时间从 ~17.5 美元闪崩至 0.01 美元；同时 60,000+ 用户的邮箱、用户名、MD5(unsalted) 密码哈希被公开泄露。事件暴露了早期 CEX 的弱密码学、无冷热钱包隔离、无风控熔断的系统性问题，是加密行业最早的 Tier-3 级大规模安全事件之一。

## 1. 事件背景

- **主体**：Mt.Gox（Magic: The Gathering Online Exchange），由 Jed McCaleb 于 2010 年创立，2011 年 3 月出售给法籍 Mark Karpelès 运营的 Tibanne 公司。事件发生时 Mt.Gox 日均处理 BTC 交易量占全球约 70%，是早期 Bitcoin 生态的事实中心化入口。
- **时间轴**：
  - 2011-06-13：用户 allinvain 报告丢失 25,000 BTC（当时价 ~$500k），被视作事件前兆（见 bitcointalk 主题 16457）。一般认为这是独立的客户端盗窃事件，但与 Mt.Gox 数据库泄露在时间上相邻。
  - 2011-06-19（UTC）：Mt.Gox 审计员账户（具有只读高额余额的管理员账号）被入侵，攻击者使用该账号在交易所内部创建了大量 BTC 卖单。
  - 2011-06-20：Mark Karpelès 在 bitcointalk 发帖确认交易所被黑，冻结交易并回滚当日异常成交。
  - 2011-06-26：部分用户数据库（约 61,016 条记录）在互联网公开流传。
- **发现过程**：社区用户在价格行情上首先察觉到 BTC 从 ~17.5 美元瞬间跌至 0.01 美元；官方约 1 小时后发公告暂停交易。

## 2. 事件影响

- **直接损失**：
  - 被盗 BTC 约 2,000（按当时 ~$17.5 估算 ~$35k）至非官方估计的 25,000（~$437k）之间，具体官方未披露（未公开）。
  - 数据库泄露：61,016 个账号的 email + username + 密码哈希（MD5，部分 unsalted，部分加 salt 但规则简单）被公开。
- **受害方**：所有 Mt.Gox 用户（尤其密码在其他站点复用者）；被迫低价出货的做市商。
- **连带影响**：BTC 全球现货价格短时间闪崩至 0.01 美元（仅限 Mt.Gox 内部成交），其他交易所波动到 ~$10。Mt.Gox 回滚了部分成交，但"闪崩图"成为早期加密市场教材级案例。数据库泄露导致大量用户在其他早期服务（MyBitcoin、TradeHill 等）遭撞库，间接催生了后续数月内的多起账户入侵。
- **资金去向**：部分被盗 BTC 实际被 Mt.Gox 内部冻结（因操作在交易所内部账本完成，未上链广播）；链外提现部分去向未公开。

## 3. 技术根因

- **漏洞分类**：权限管理 + 弱密码学 + 无风控熔断（Key-Mgmt / Social-Eng 混合）。
- **关键技术缺陷**：
  1. **管理员账户凭据被拖库**：审计员（auditor）账号具有查看全站余额的权限，其密码哈希被攻击者从早期数据库获取并破解。
  2. **密码哈希弱算法**：Mt.Gox 早期部分账户使用 MD5 unsalted，后期改为 MD5+salt，但 salt 规则被分析后仍可批量破解。
  3. **无交易速率/价格异常熔断**：单账户可以挂出数万 BTC 以 0.01 美元的卖单而不触发任何限制。
  4. **无冷热钱包隔离**：Mt.Gox 在 2011 年尚未建立系统性的冷热钱包比例策略，后续 2014 年崩溃调查进一步证实其内部资金管理长期混乱。
- **受损模块**：Mt.Gox 自研 PHP/Python 交易后端（代码未开源，归档见 Wayback Machine 及泄露的 `www.mtgox.com` 截图）。
- **关键"伪代码"还原**（基于公开讨论，非原始代码）：

```python
# 简化还原 Mt.Gox 2011-06 的下单路径
def place_order(user, side, amount_btc, price_usd):
    # 缺陷 1：无单笔/单日数量上限
    # 缺陷 2：无基于 VWAP 的价格偏离熔断
    # 缺陷 3：auditor 账号居然有交易权
    if user.balance_btc >= amount_btc:
        orderbook.insert(side, amount_btc, price_usd)
        # 内部账本直接撮合，不产生链上 tx
```

- **攻击步骤**：
  1. 攻击者从早期 Mt.Gox 用户库（可能来自 2011 年更早的一次测试库泄露）获取 auditor 账户哈希。
  2. 离线破解 MD5 哈希获得明文密码。
  3. 登录 auditor 账户，自行构造卖单以 0.01 美元挂出海量 BTC。
  4. 使用多个账户吸收低价订单并尝试提现 BTC（大部分被 Mt.Gox 事后冻结）。
- **tx hash**：因交易发生在 Mt.Gox 内部账本，未上链广播，**tx 未公开**；仅事后提现部分可能上链，hash 未被官方披露。
- **审计盲区**：Mt.Gox 当时并无第三方安全审计，也无外部渗透测试记录。

## 4. 事后响应

- **项目方**：
  - 暂停交易 7 天（2011-06-20 至 2011-06-26）。
  - 强制所有用户重置密码；升级密码存储为更强的算法（但具体升级到 bcrypt/scrypt 未公开）。
  - 回滚 2011-06-19 异常成交，按事发前中间价恢复账本。
- **资产追回**：由于大部分异常成交被回滚，公司层面自述损失有限（几千 BTC 级别），但这一数据后来在 2014 破产清算中被质疑与长期账本差异吻合。
- **执法**：2011 年无执法介入；美日多起调查在 2014 年崩溃后才启动。
- **行业连锁**：
  - 触发社区对"热钱包/冷钱包分离"的最早讨论（见 bitcointalk 讨论串）。
  - 催生了 Armory、Electrum 等离线签名钱包方案的普及。

## 5. 启发与教训

- **对开发者 / CEX**：
  - 密码存储必须使用 bcrypt / scrypt / Argon2，不可使用 MD5/SHA1，无论是否 salt。
  - 管理后台账户必须最小权限（least privilege），审计员账户绝不应有下单权。
  - 必须实现价格/数量熔断：偏离 VWAP > N% 或超出单账户日历史峰值 M 倍的订单需人工审核。
- **对审计方**：CEX 安全审计必须覆盖：账号权限矩阵、密码存储、风控规则、冷热钱包比例。2011 年这些都属空白。
- **对用户**：不要在多个加密服务间复用密码；启用 2FA（2011 年 Mt.Gox 尚未强制 2FA）。
- **对协议 / 生态**：推动了 BIP-32/39/44 等 HD 钱包标准和多签托管方案的落地。

## 6. 参考资料

- bitcointalk 原帖：Mark Karpelès 官方声明 `bitcointalk.org/index.php?topic=19139`
- Wired：《Bitcoin Flaw Costs User $500K》 2011-06
- Ars Technica：《Bitcoin prices plummet on hacked exchange》 2011-06
- en.bitcoin.it wiki：MtGox 历史条目（社区维护）
- Wayback Machine 归档：Mt.Gox 2011 年 6 月 19–27 日公告页
- 慢雾 / CertiK 无原始分析（事件早于其成立），仅事后回顾性引用

---

*Last verified: 2026-04-23*
