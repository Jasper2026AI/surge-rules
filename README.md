# surge-rules

自用 Surge 分流规则集（iOS Surge 5）。

## 规则列表

| 文件 | 用途 | 引用方式 |
| --- | --- | --- |
| `Binance.list` | 币安 | `RULE-SET,https://raw.githubusercontent.com/Jasper2026AI/surge-rules/main/Binance.list,Binance,update-interval=86400` |
| `Polymarket.list` | Polymarket | `RULE-SET,https://raw.githubusercontent.com/Jasper2026AI/surge-rules/main/Polymarket.list,Polymarket,update-interval=86400` |

规则文件内不含策略名，策略在引用处指定。所有 RULE-SET 需放在 GEOIP、FINAL 等 IP 类规则之前。

---

## Binance

### 来源

- 主体来自 [StricklandF/Filter](https://github.com/StricklandF/Filter/blob/main/Binance.list)（2026-07-22，Quantumult X 格式），转换为 Surge 格式。
- 合并了 [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Surge/Binance) 中独有的条目。blackmatrix7 版本最后更新于 2025-06，只有 12 条，缺少 `binance.info`、`binanceapi.com`、Forter 风控等登录相关域名。
- 共 65 条：主域名、备用域名、静态资源、登录风控/证书、AWS 及少量硬编码 IP。

### 已知问题：登录记录显示真实 IP（未解决）

**现象**：使用本规则后，币安账户的「登录记录」里仍显示本机真实 IP，而不是代理节点 IP。

**抓包结论（2026-10-18）**：

- 带域名的请求（`themis.binance.com`、`static.bscdnweb.com`、`*.cdn.forter.com`、`edge-api.saasexch.com:34222` 等）都正常命中规则、走了代理。
- 同时有一批**不带域名、直接连 IP** 的 HTTPS 请求走了 DIRECT，IP 都在阿里云（深圳）和腾讯云（广州）。
- 判断是币安 App 的"大陆加速线路"：App 通过 `edge-api.saasexch.com` 获取大陆服务器 IP 列表，再直接连这些 IP。真实 IP 就是从这些连接泄露的。

**尝试记录**：

| # | 做法 | 结果 |
| --- | --- | --- |
| 1 | 把抓到的 6 个大陆 IP 以 `IP-CIDR` 加入 Binance.list，走代理 | 规则命中，但经代理全部连不上（↓0 B ↑517 B，无回包）。App 随即换用其他大陆 IP 继续直连 |
| 2 | 把大陆 IP（共 12 个）移到单独的 `BinanceCN.list`，策略 REJECT | 被拒绝的 IP 不再出现，但 App 又换了一批新 IP 直连（`47.107.38.4`、`39.108.62.1`、`39.108.140.235`、`112.74.86.215`、`182.254.194.76` 等） |
| 3 | `BinanceCN.list` 改为拒绝线路调度域名（`saasexch.*`、`yingwangtech.*`）+ 按 /16 网段拒绝（阿里云深圳 7 段、腾讯云广州 3 段） | 失败 |

**处理**：以上改动已全部撤销，`Binance.list` 恢复为最初版本，`BinanceCN.list` 已删除。

**涉及的大陆 IP（供参考）**：

```
阿里云：47.106.193.36  47.107.38.4  47.107.32.31  47.107.98.27  47.107.127.135
        39.108.62.1  39.108.140.235  39.108.213.59  120.25.76.84  120.25.250.183
        120.77.220.2  112.74.86.215  112.74.111.231  8.133.231.173
腾讯云：81.71.41.114  106.55.239.211  182.254.194.76
```

**结论与后续方向**：

- App 的大陆线路使用 IP 直连且 IP 池会轮换，靠分流规则很难完全拦住。
- 未尝试的方向：在币安 App 内把地区/语言改为非中文；删除 App 重装，让它重新选择线路；换其他地区的代理节点测试。
