# Cowjiang AdGuard

自建 DNS 广告过滤系统：基于 AdGuard Home + 自有云服务器，为 iOS / Android / 全屋设备提供广告与追踪拦截。

- iOS：安装 [Cowjiang-AdGuard.mobileconfig](https://github.com/Cowjiang/cowjiang-adguard/releases/download/profile/Cowjiang-AdGuard.mobileconfig) 描述文件（加密 DNS DoT，蜂窝/Wi-Fi 全局生效）
- Android：设置 → 网络 → 私人 DNS → 填入你的 DoT 域名（如 `dns.example.com`）
- 家庭路由器 / 其他设备：DNS 填服务器 IP（明文 53）

## 数据来源与处理链路

每日自动从 [GMOogway/shadowrocket-rules](https://github.com/GMOogway/shadowrocket-rules) 拉取最新的 `upstream/sr_reject_list.module`（约 19 万条拦截规则），加上本仓库人工维护的 `rules/custom_reject_list.module`，由 `factory/build_agh.sh` 转换为 AdGuard DNS 过滤语法：

| 生成物 | 用途 | AGH 订阅地址 |
|---|---|---|
| `dist/agh_sr_reject.txt` | 上游全量拦截列表 | `raw.githubusercontent.com/<你>/<新仓库>/master/dist/agh_sr_reject.txt` |
| `dist/agh_custom_reject.txt` | 自定义拦截规则 | `raw.githubusercontent.com/<你>/<新仓库>/master/dist/agh_custom_reject.txt` |
| `dist/Cowjiang-AdGuard.mobileconfig` | iOS 描述文件 | [Release 下载](https://github.com/Cowjiang/cowjiang-adguard/releases/download/profile/Cowjiang-AdGuard.mobileconfig) |

## 仓库结构

```
├── rules/                          # 源规则 (人工维护)
│   └── custom_reject_list.module   # 自定义拦截规则 (Shadowrocket 语法)
├── upstream/                       # 上游规则快照 (CI 生成, 勿手改)
│   └── sr_reject_list.module       # 每日从 GMOogway/shadowrocket-rules 拉取
├── dist/                           # 生成物 (CI 生成, 勿手改)
│   ├── agh_sr_reject.txt           # 转换后的上游全量拦截列表
│   ├── agh_custom_reject.txt       # 转换后的自定义拦截列表
│   └── Cowjiang-AdGuard.mobileconfig  # iOS 描述文件
├── factory/build_agh.sh            # 语法转换 + 描述文件生成
└── .github/workflows/
    ├── build.yml                  # 每日: 拉上游 → 转换 → 提交 → 更新 Release
    └── renew-cert.yml             # 双月: 签发/部署 DoT 域名证书
```

## 规则语法映射

```
DOMAIN,x,REJECT        →  ||x^
DOMAIN-SUFFIX,x,REJECT →  ||x^
DOMAIN-KEYWORD,x       →  x        (子串匹配)
IP-CIDR                →  (跳过, DNS 层无法按 IP 拦截)
```

## 初始部署:配置 GitHub Secrets

证书签发与部署(`renew-cert.yml`)依赖 3 个 secrets,在 **Settings → Secrets and variables → Actions** 中添加:

- `CF_TOKEN` — Cloudflare API Token(权限:Zone → DNS → Edit),用于 DNS-01 验证
- `SSH_HOST` — 服务器地址
- `SSH_KEY` — 服务器 SSH 私钥

## License

GPL-3.0. Rule data is sourced daily from [GMOogway/shadowrocket-rules](https://github.com/GMOogway/shadowrocket-rules) (GPL-3.0); the converted lists in this repository are therefore distributed under the same license.
