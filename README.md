# Cowjiang AdGuard

自建 DNS 广告过滤系统：基于 AdGuard Home + 自有云服务器，为 iOS / Android / 全屋设备提供广告与追踪拦截。

- iOS：安装 [Cowjiang-AdGuard.mobileconfig](https://github.com/Cowjiang/cowjiang-adguard/releases/download/profile/Cowjiang-AdGuard.mobileconfig) 描述文件（加密 DNS DoT，蜂窝/Wi-Fi 全局生效）
- Android：设置 → 网络 → 私人 DNS → 填入 `dns.inceptae.com`
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
    └── renew-cert.yml             # 双月: 签发/部署 dns.inceptae.com 证书
```

## 规则语法映射

```
DOMAIN,x,REJECT        →  ||x^
DOMAIN-SUFFIX,x,REJECT →  ||x^
DOMAIN-KEYWORD,x       →  x        (子串匹配)
IP-CIDR                →  (跳过, DNS 层无法按 IP 拦截)
```

## 初始部署:配置 GitHub Secrets

仓库的自动化依赖 3 个 GitHub Actions secrets,用于证书签发与部署(`renew-cert.yml`)。首次部署或迁移仓库时,在 GitHub 仓库页 **Settings → Secrets and variables → Actions → New repository secret** 依次添加:

| Secret 名称 | 内容 | 获取方式 |
|---|---|---|
| `CF_TOKEN` | Cloudflare API Token | [Cloudflare Dashboard → My Profile → API Tokens](https://dash.cloudflare.com/profile/api-tokens) 创建,权限选 **Zone → DNS → Edit**,并授权 `dns.inceptae.com` 所在 Zone。证书通过 DNS-01 验证签发,需要它写 TXT 记录 |
| `SSH_HOST` | 云服务器公网 IP | 你的 AdGuard Home 所在服务器 IP |
| `SSH_KEY` | 服务器 SSH 私钥 | 本地执行 `ssh-keygen -t ed25519` 生成密钥对,把**私钥**文件全文(含 `-----BEGIN OPENSSH PRIVATE KEY-----` 和结尾空行)粘贴进去,并将对应**公钥**追加到服务器的 `/root/.ssh/authorized_keys` |

配置完成后,到 **Actions → Renew DNS Cert → Run workflow** 手动触发一次,验证证书能签发并部署到服务器(日志末尾应出现 `Verify return code: 0 (ok)`)。

> 安全提示:secrets 仅在 Actions 运行时注入,不会出现在日志和代码中;私钥不要提交到仓库。

## License

GPL-3.0. Rule data is sourced daily from [GMOogway/shadowrocket-rules](https://github.com/GMOogway/shadowrocket-rules) (GPL-3.0); the converted lists in this repository are therefore distributed under the same license.
