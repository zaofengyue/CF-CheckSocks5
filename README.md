# CF-Workers-CheckSocks5

一个基于 Cloudflare Workers 的代理可用性检测工具。项目以单个 `_worker.js` 运行为核心，支持 SOCKS5、HTTP、HTTPS、TURN、SSTP 代理检测，提供网页端单条/批量检测、域名解析、出口 IP 信息展示、地图定位、结果筛选与导出。

> 支持可选的 `TOKEN` 鉴权。未设置 `TOKEN` 环境变量时默认公开访问；设置后将自动保护 `/check`、`/resolve`、`/resolve-batch` 等核心接口，防止被未授权盗刷。

## 功能特性


- 支持 `socks5://`、`http://`、`https://`、`turn://`、`sstp://` 五类代理协议。
- 支持无认证代理、`username:password` 认证代理，以及 IPv4、域名、方括号 IPv6 地址。
- TURN 检测使用 TCP Allocation / CONNECT / ConnectionBind 流程，支持无认证 TURN 服务器和长期凭据认证。
- SSTP 检测使用 HTTPS SSTP 握手、PPP / IPCP 建链，并通过 PPP 内 TCP 连接读取出口信息。
- 支持单条检测和批量检测；批量模式会自动去重、解析域名并并发验证。
- 支持域名解析为 A / AAAA 记录，优先使用 Cloudflare DoH，失败后回退到 Google DoH。
- 支持出口 IP 查询多源自由切换，涵盖 `iplocate.io`、`HackMyIP`（风控首选 · 原生住宅/机房/VPN与纯净度评级）、`IP2Location.io`（全球权威顶牌 · 自带代理识别 / 免密即用）、`ipwho.is`（高精度全字段规范）、`api.ip.sb`（极速大批量）、`Cloudflare 官方 Trace`（1w+ 超大批量首选，零风控无上限）、`ipinfo.io`、`ipapi.co` 以及 `api.ipapi.is`（深度风控识别 · 支持多 Key 轮询）。
- 支持代理出口信息展示，包括出口 IP、地区、ASN、运营商、风险标签、响应耗时等。
- 支持 Leaflet / OpenStreetMap 地图展示出口位置。
- 支持结果筛选，支持将有效及失败结果复制到剪贴板或导出为 TXT / CSV（包含详细报错原因）。
- 彻底移除第三方追踪脚本，保障节点数据与访客隐私绝对安全。
- 支持深浅色主题、历史记录快速回填和自定义页脚备案内容。

## 在线体验

Demo: <https://check.socks5.cmliussss.net>

## 部署方式

### 方式一：Cloudflare Workers 控制台粘贴代码部署（推荐）

1. 在 Cloudflare 控制台 -> **Workers 和 Pages** -> 点击 **创建 Worker**。
2. 点击部署生成默认 Worker，进入详情页后点击右上角 **编辑代码**。
3. 将本项目中的 [_worker.js](./_worker.js) 全部内容复制，粘贴替换编辑器中的所有代码，点击 **部署**。
4. **（可选，强烈推荐）开启后台管理与分布式集群功能**：
   - 在 Cloudflare 控制台 -> **KV** 中创建一个新的 KV 命名空间（例如：`CHECK_SOCKS5_KV`）。
   - 进入该 Worker 的 **设置** -> **绑定 (Bindings)** -> 添加 **KV 命名空间绑定**：
     - **变量名称**：`KV`（兼容 `CONFIG_KV`）
     - **KV 命名空间**：选择刚才创建的命名空间。
   - 访问你的域名 `/admin`，首次打开会提示初始化管理员密码（亦可在环境变量设置 `ADMIN`），进入后台即可图形化管理 Token、接口 Key 池和集群调度。

### 方式二：Cloudflare Pages 上传压缩包 / 文件夹部署

本项目已内置 `_routes.json` 与 `index.html`，完美支持 Pages **直接上传（Direct Upload）**：

1. 在 Cloudflare 控制台 -> **Workers 和 Pages** -> 点击 **创建** -> 选择 **Pages** -> **直接上传（Direct Upload）**。
2. 输入项目名称，在上传区域直接上传：
   - **方式 A（上传文件夹）**：解压下载的项目包，直接将包含 `_worker.js` 的文件夹拖入上传区域。
   - **方式 B（上传压缩包）**：将 `_worker.js`、`_routes.json`、`index.html` 等文件全选压缩为 zip 上传（请确保 `_worker.js` 位于压缩包最外层根目录，不要嵌套子文件夹）。
3. 点击 **部署站点**。
4. 同样可以在 Pages 项目的 **设置** -> **函数 (Functions)** -> **KV 命名空间绑定** 中添加变量名 `KV` 激活后台管理系统。

## 环境变量与 KV 绑定

当前源码支持读取以下配置（最简仅需 `KV` 与 `ADMIN`）：

| 变量 / 绑定名 | 类型 | 说明 | 示例 | 必需 |
| --- | --- | --- | --- | --- |
| `KV` | KV 命名空间 | 绑定 Cloudflare KV 命名空间，激活 `/admin` 图形后台管理系统、多 Key 自动轮询、分布式集群调度与动态 Token 增删。亦兼容旧名 `CONFIG_KV`。 | 绑定至 KV 命名空间 | 否 |
| `ADMIN` | 环境变量 | 管理员后台密码。亦可在初次访问 `/admin` 时直接网页输入初始化。兼容 `ADMIN_PASSWORD`、`ADMIN_TOKEN`。 | `admin123456` | 否 |
| `TOKEN` | 环境变量 | 全局兜底访问鉴权令牌（兼容 `AUTH_TOKEN`）。若绑定了 KV，亦可在 `/admin` 后台动态新增和管理多个 Token。 | `my-secret-token` | 否 |
| `BEIAN` | 环境变量 | 自定义页面页脚 HTML。未设置时使用默认页脚，包含项目链接和维护者链接。 | `© 2026 Example.com · ICP 备案号` | 否 |

## 后台管理系统 (`/admin`)

当 Worker / Pages 绑定了 `KV`（亦兼容 `CONFIG_KV`）命名空间时，访问 `https://your-domain/admin` 将开启管理控制台：

- **访问 Token 管理 (Tokens)**：可视化创建、禁用、启用与删除用户 Token，支持为每个 Token 备注用途（如“自用客户端”、“群友测试”等）。
- **接口与多 Key 池管理 (Sources & Key Pool)**：
  - 支持为商业 IP 查询接口（如 `api.ipapi.is`、`api.ip2location.io`、`ipdata.co` 等）配置多组 API Key，自动 Round-Robin 轮询调度与 429 智能熔断。
  - **自定义接口 (Custom IP Sources)**：支持任意添加第三方自建或商业 IP 数据库，支持在 URL 中设置 `{{KEY}}` 搭配 Key 池轮询，并通过 JSON 点操作符路径（如 `data.ip`、`location.country`）自由映射提取出口属性。
- **Worker 分布式集群调度与纯集群模式 (Worker Cluster)**：
  - **⚡ 纯集群调度模式 (主账号 0 消耗)**：一键开启纯集群模式，主 Worker 仅作为控制面板与管理中枢，所有测活流量 100% 分发给子 Worker 节点，彻底保持主账号每日 10 万次配额零消耗！
  - **📋 极简子节点专属脚本**：后台支持一键复制约 200 行免 KV 空间、免环境变量的纯执行独立脚本，在小号 Cloudflare 直接粘贴部署，秒级上线从节点。
  - **10w+ 智能容灾与熔断重试**：大批量测活时，当某个子节点因高频请求遇到 429/1015/1027 或网络离线时，前端调度器自动将其标记 5 分钟冷却，并自动切换至下一健康子节点重试，大批量检测持续不中断。
  - **一键全节点测速巡检**：实时探测所有子 Worker 的 HTTP 往返延迟与 Cloudflare 边缘机房代码（如 HKG、SJC 等）。
- **配置导入导出与备份 (Settings & Backup)**：支持一键导出全站配置 JSON 备份文件，迁移站点时一键上传还原。

## 支持的代理格式

```text
socks5://host:1080
socks5://username:password@host:1080
socks5://username:password@[2001:db8::1]:1080
http://host:80
http://username:password@host:80
https://host:443
https://username:password@host:443
turn://host:3478
turn://username:password@host:3478
sstp://host:443
sstp://username:password@host:443
```

网页端输入缺少协议头时，会默认按 `socks5://` 处理。端口缺省值分别为：

| 协议 | 默认端口 |
| --- | --- |
| `socks5` | `1080` |
| `http` | `80` |
| `https` | `443` |
| `turn` | `3478` |
| `sstp` | `443` |

### TURN 支持说明

`turn://` 目标会被当作 TURN over TCP 服务器检测。Worker 会先连接 TURN 服务器，再通过 TURN TCP 中继访问 `www.iplocate.io:443`，最后读取出口 IP 信息。

当前 TURN 实现有以下边界：

- 支持 RFC 6062 风格的 TCP Allocation、CreatePermission、CONNECT 和 ConnectionBind。
- 支持无认证服务器；如果服务端返回 `401` 认证挑战，并且链接中提供了 `username:password`，会使用长期凭据认证继续握手。
- 目标出口检测地址会解析为 IPv4 后发起 TURN CONNECT；当前不走 TURN UDP relay，也不支持 `turns://`。
- `turn://` 中的主机可以是 IP 或域名，端口未填写时默认使用 `3478`。

### SSTP 支持说明

`sstp://` 目标会被当作 SSTP over TLS 服务器检测。Worker 会先建立 SSTP HTTP 隧道，再完成 PPP / IPCP 协商，随后在 PPP 内构造 TCP 连接访问 `www.iplocate.io:443`，最后读取出口 IP 信息。

当前 SSTP 实现有以下边界：

- 支持无认证 SSTP 服务器；如果 PPP 协商要求认证，仅支持 PAP，并使用链接中提供的 `username:password`。
- 目标出口检测地址会解析为 IPv4 后建立 PPP 内 TCP 连接；当前 SSTP 检测依赖服务端分配 IPv4 地址。
- `sstp://` 中的主机可以是 IP 或域名，端口未填写时默认使用 `443`。

## API

所有 JSON 接口都带有 CORS 响应头，并支持 `OPTIONS` 预检请求。

### 接口鉴权 (Token)

若配置了 `TOKEN` 环境变量，访问受保护接口必须携带 Token。支持以下多种方式（优先级由高到低）：

1. **URL 路径前缀（最简便直观）**：
   - 网页直接访问：`https://your-worker.example.workers.dev/<token>`（访问即自动完成鉴权，并自动清洗地址栏）
   - API 接口调用：`https://your-worker.example.workers.dev/<token>/check?proxy=socks5://host:port`
   - 域名解析调用：`https://your-worker.example.workers.dev/<token>/resolve?target=domain:port`
2. **Authorization 请求头**：`Authorization: Bearer <token>`
3. **X-Token 请求头**：`X-Token: <token>`
4. **URL 查询参数**：`?token=<token>` 或 `?key=<token>`

未提供或 Token 错误时返回 HTTP `401 Unauthorized`：
```json
{
  "success": false,
  "error": "Unauthorized: missing or invalid token"
}
```

### `GET /check`

检测单个代理是否可用。Worker 会通过代理建立到 `www.iplocate.io` 的连接，并读取该服务返回的出口 IP 信息。

请求参数支持以下写法：

```text
/check?socks5=proxy.example.com:1080
/check?http=proxy.example.com:80
/check?https=proxy.example.com:443
/check?turn=turn.example.com:3478
/check?sstp=vpn:vpn@vpn205396913.opengw.net:1922

/check?proxy=socks5://user:pass@proxy.example.com:1080
/check?proxy=http://proxy.example.com:80
/check?proxy=https://proxy.example.com:443
/check?proxy=turn://user:pass@turn.example.com:3478
/check?proxy=sstp://vpn:vpn@vpn890321947.opengw.net:1630
/check/proxy=socks5://proxy.example.com:1080
```

响应示例：

```json
{
  "candidate": "proxy.example.com:1080",
  "type": "socks5",
  "username": null,
  "password": null,
  "hostname": "proxy.example.com",
  "port": 1080,
  "link": "socks5://proxy.example.com:1080",
  "success": true,
  "responseTime": 523,
  "exit": {
    "ip": "203.0.113.10",
    "rir": "APNIC",
    "is_datacenter": true,
    "is_proxy": false,
    "is_vpn": false,
    "asn": {
      "asn": 64500,
      "org": "Example Network"
    },
    "location": {
      "country": "Japan",
      "country_code": "JP",
      "city": "Tokyo",
      "latitude": 35.6895,
      "longitude": 139.6917
    }
  }
}
```

失败时会返回 `success: false` 和 `error` 字段。

### `GET /resolve`

将域名或代理链接解析为可检测的 `host:port` 列表。

参数别名：

- `proxyip`
- `target`
- `host`

示例：

```bash
curl "https://your-worker.example.workers.dev/resolve?proxyip=socks5://proxy.example.com:1080"
```

响应示例：

```json
[
  "198.51.100.10:1080",
  "[2001:db8::10]:1080"
]
```

解析规则：

- 输入已经是 IPv4 或 IPv6 时，直接返回原目标和端口。
- 输入是域名时，解析 A / AAAA 记录。
- 未提供端口时，解析接口默认使用 `443`。

### `POST /resolve-batch`

批量解析目标。单次最多 `50` 个。

请求体支持 `targets` 或 `proxyips`：

```json
{
  "targets": [
    "socks5://proxy-a.example.com:1080",
    "proxy-b.example.com:1080"
  ]
}
```

响应示例：

```json
{
  "results": [
    {
      "input": "proxy-b.example.com:1080",
      "targets": [
        "198.51.100.20:1080"
      ]
    }
  ]
}
```

## 网页端使用

1. 打开部署后的 Worker 域名。
2. **Token 配置**（若服务端启用了 `TOKEN` 鉴权）：
   - **方式 A（直接路径访问，最推荐）**：在浏览器地址栏直接输入 `https://your-worker.example.workers.dev/你的Token值`，访问后会自动完成鉴权，页面将自动将 Token 存入本地并将地址栏清洗回根路径 `/`。
   - **方式 B（UI 手动配置）**：点击页面右上角工具栏中的 **钥匙图标**，输入 Token 并点击「保存」（Token 将保存在浏览器本地 `localStorage`，后续检测自动携带）。
   - **方式 C（URL 参数访问）**：直接在浏览器中访问带参数链接：`https://your-worker.example.workers.dev/?token=你的Token值`。
3. 在输入框中填写代理链接、`IP:端口`、`域名:端口` 或带认证的代理地址。
4. 如需批量检测，打开「批量检测」并粘贴多行目标。
5. 点击「开始检测」。
6. 检测完成后，可按全部/有效/失败/风控评级、协议和国家地区筛选，并导出有效结果。

也可以直接通过路径触发单条检测：

```text
https://your-worker.example.workers.dev/socks5://proxy.example.com:1080
```

## 运行参数

源码中的主要限制和超时：

| 参数 | 当前值 | 说明 |
| --- | --- | --- |
| `CHECK_TIMEOUT_MS` | `12000` | 单次代理检测总超时 |
| `CONNECT_TIMEOUT_MS` | `9999` | 代理连接和握手超时 |
| `READ_TIMEOUT_MS` | `8000` | 读取远端响应超时 |
| `MAX_RESPONSE_BYTES` | `96 KiB` | 读取出口信息响应的最大字节数 |
| `RESOLVE_BATCH_LIMIT` | `50` | 批量解析接口单次最大目标数 |
| 前端检测并发 | `32` | 网页端批量检测的并发数 |

## 注意事项

- Cloudflare Workers 的 TCP Socket 能力由 `cloudflare:sockets` 提供，请确保部署环境支持 Workers TCP 出站连接。
- 检测逻辑会把代理作为隧道访问 `www.iplocate.io`，因此结果反映的是该代理访问该目标服务时的可用性和出口信息。
- TURN 检测依赖 TURN 服务器支持 TCP relay / CONNECT；只支持 UDP relay 的 TURN 服务会检测失败。
- SSTP 检测依赖服务端支持 SSTP over TLS、PPP / IPCP 和 IPv4 分配；仅支持 PAP 认证，不支持 MS-CHAP 等其他 PPP 认证方式。
- 公开部署时建议在 Cloudflare 环境变量中配置 `TOKEN`，防止检测接口被外部未授权盗用。
- 大批量检测可能受到 Cloudflare Workers 执行时长、并发和外部 DNS/API 可用性的影响。

## 许可证

本项目基于 [GNU General Public License v3.0](./LICENSE) 发布。

## 开源代码引用
- [CF-Workers-HTTPS](https://github.com/ToiCF/CF-Workers-HTTPS)
- [CF-Workers-TURN](https://github.com/ToiCF/CF-Workers-TURN)
- [CF-Workers-SoftEther](https://github.com/ToiCF/CF-Workers-SoftEther)

## 致谢
- [@Alexandre_Kojeve](https://t.me/Alexandre_Kojeve)
- [Cloudflare Workers](https://workers.cloudflare.com/)
- [iplocate.io](https://www.iplocate.io/)
- [Cloudflare DNS](https://cloudflare-dns.com/)
- [OpenStreetMap](https://www.openstreetmap.org/)
- [Leaflet](https://leafletjs.com/)
