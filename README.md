# Cowjiang AdGuard

自建 DNS 广告过滤：AdGuard Home + 自有云服务器，iOS / Android / 全屋设备广告与追踪拦截。

## 快速开始

### 已安装 AdGuard Home

在 AdGuard Home 管理界面的「过滤 → DNS 拦截」中添加以下订阅规则：

| 名称                     | 订阅地址                                                                                            |
|------------------------|-------------------------------------------------------------------------------------------------|
| Cowjiang SR Reject     | `https://raw.githubusercontent.com/Cowjiang/cowjiang-adguard/master/dist/agh_sr_reject.txt`     |
| Cowjiang Custom Reject | `https://raw.githubusercontent.com/Cowjiang/cowjiang-adguard/master/dist/agh_custom_reject.txt` |

添加后启用即可生效，规则每日自动更新。

### 未安装 AdGuard Home

使用 `deployment/` 下的一键部署脚本：

```bash
cd deployment
docker-compose up -d
```

启动后访问 `http://localhost:3000` 账号密码:admin/adguardadmin

### 修改账号密码(务必修改否则不要对外暴露3000)

- 使用`bcrypt`算法加密，将密文填写到配置文件`deployment/conf/AdGuardHome.yaml`即可
- `bcrypt`可以直接搜索，有在线工具。也可以按照`ZTools`然后安装`CryptTool`插件

```yaml
users:
  - name: admin
    password: 密文 
```
- 修改完成后`docker-compose restart`
### 客户端配置

**iOS**：TODO

**Android**：TODO

**路由器 / 其他设备**：DNS 填服务器 IP

---

## 项目主线：规则转换框架

将不同来源、不同格式的广告过滤规则统一转换为 AdGuard DNS 过滤语法。

### 架构

```
sources/*.yaml        # 数据源配置（URL + 格式类型）
converters/           # 格式转换器（每种格式一个）
  shadowrocket.ts     # Shadowrocket → AdGuard
lib/                  # 核心库
  fetcher.ts          # HTTP/文件获取（支持代理）
  deduplicator.ts     # 规则去重
  merger.ts           # 规则合并
index.ts              # 主入口
dist/                 # 输出目录
```

### 工作流程

```
sources/*.yaml → 拉取源 → 格式转换 → 去重合并 → dist/*.txt
```

### 构建

```bash
npm install
npm run build
```

### 订阅地址

| 规则              | 订阅地址                                                                                    |
|-----------------|-----------------------------------------------------------------------------------------|
| Shadowrocket 拦截 | `raw.githubusercontent.com/Cowjiang/cowjiang-adguard/master/dist/agh_sr_reject.txt`     |
| 自定义拦截           | `raw.githubusercontent.com/Cowjiang/cowjiang-adguard/master/dist/agh_custom_reject.txt` |

---

## 扩展指引

### 添加新的数据源格式

**1. 创建转换器** `converters/your-format.ts`

```typescript
import {Converter} from '../types';

export const yourFormatConverter: Converter = {
    format: 'your-format',
    convert(lines: string[]): string[] {
        const rules: string[] = [];
        // 实现转换逻辑
        // 返回 AdGuard DNS 格式规则数组
        return rules;
    }
};
```

**2. 注册转换器** `converters/index.ts`

```typescript
import {yourFormatConverter} from './your-format';

const converters = {
    shadowrocket: shadowrocketConverter,
    yourFormat: yourFormatConverter  // 添加
};
```

**3. 创建数据源配置** `sources/your-format.yaml`

```yaml
format: your-format
sources:
  - name: source-name
    url: https://example.com/rules.txt
    output: agh_output.txt
  - name: local-source
    path: local/rules.txt
    output: agh_local.txt
```

**4. 构建测试**

```bash
npm run build
```

### 支持的格式

| 格式           | 转换器               | 说明                                    |
|--------------|-------------------|---------------------------------------|
| Shadowrocket | `shadowrocket.ts` | DOMAIN, DOMAIN-SUFFIX, DOMAIN-KEYWORD |

### 添加新的数据源

在对应的 `sources/*.yaml` 中添加条目：

```yaml
format: shadowrocket
sources:
  - name: existing-source
    url: https://...
    output: agh_sr_reject.txt
  - name: new-source           # 新增
    url: https://...           # 远程 URL
    output: agh_new.txt
```

支持两种源类型：

- `url`: 远程文件，自动拉取
- `path`: 本地文件，直接读取

---

## 项目支线：快速部署 AGH 服务

`deployment/` 目录提供一键启动 AdGuard Home 的脚本，并自动订阅当前仓库的过滤规则。

### 快速启动

```bash
cd deployment
docker-compose up -d
```

### 服务地址

| 服务       | 地址                    |
|----------|-----------------------|
| Web 管理界面 | http://localhost:3000 |
| DNS 服务   | localhost:53          |

### 首次配置

1. 访问 `http://localhost:3000` 进入设置向导
2. 设置管理员密码
3. 过滤规则已预配置，无需手动添加

### 默认配置

**预置过滤规则：**

- Cowjiang AdGuard - SR Reject
- Cowjiang AdGuard - Custom Reject

**上游 DNS：**

- 阿里 DNS: `223.5.5.5`
- 腾讯 DNS: `119.29.29.29`

### 自定义配置

编辑 `deployment/conf/AdGuardHome.yaml` 后重启：

```bash
docker-compose restart
```

### 常用命令

```bash
# 启动
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止
docker-compose down

# 重启
docker-compose restart

# 更新镜像
docker-compose pull
docker-compose up -d
```

### 使用本地规则

如果需要使用本地构建的规则，可以挂载 `dist/` 目录：

```yaml
# docker-compose.yml
volumes:
  - ../dist:/rules
```

然后修改 `conf/AdGuardHome.yaml` 中的过滤规则路径：

```yaml
filters:
  - enabled: true
    url: file:///rules/agh_sr_reject.txt
    name: Local SR Reject
    id: 1
```

---

## 自动化

| Workflow    | 触发               | 作用                |
|-------------|------------------|-------------------|
| `build.yml` | 每日 + push master | 转换规则 → 提交 `dist/` |

## License

GPL-3.0
