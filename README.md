# CheckSocks5 · 多账号分布式版

基于 [cmliu/CF-Workers-CheckSocks5](https://github.com/cmliu/CF-Workers-CheckSocks5) 改造的前后端分离版本，支持 SOCKS5 / HTTP / HTTPS / TURN / SSTP 检测。把检测分散到多个 Cloudflare 账号，避免单个账号每天 10 万次 Workers 请求的限制。检测逻辑与原项目完全一致（检测节点就是原 `_worker.js` 去掉页面后的代码）。

## 目录
- [架构](#架构)
- [文件说明](#文件说明)
- [快速部署](#快速部署)
- [后台管理](#后台管理)
- [检测节点调度](#检测节点调度)
- [用量统计](#用量统计)
- [可选：VPS Node 后端](#可选vps-node-后端)
- [接口说明](#接口说明)
- [安全设计](#安全设计)
- [常见问题](#常见问题)
- [与原项目的区别](#与原项目的区别)

---

## 架构

```
                    ┌──────────────────────────────────┐
                    │ 前端 Worker（_worker.js）         │
 浏览器 ──页面────▶ │  /        检测页                   │
                    │  /admin   后台（KV 存储账号/节点）  │
                    │  /locations 机房列表（缓存）        │
                    └──────────────────────────────────┘
    │
    │  /check（轮询分配，失败自动切换）
    ├────────▶ 节点 1：账号 A 的 check-node Worker
    ├────────▶ 节点 2：账号 B 的 check-node Worker
    ├────────▶ 节点 3：账号 C 的 check-node Worker
    └────────▶ 节点 N：VPS Node 后端（可选）

    域名解析：浏览器直接查 DoH（cloudflare-dns.com），不经过任何 Worker
```

- **前端 Worker 不做检测**：只提供页面和后台，请求量很小（打开一次页面约 2 次请求）
- **每个检测节点各有每天 10 万次额度**：N 个账号就是 N × 10 万次
- **检测节点由后台一键部署**：填入 API 令牌即可，不用手动粘贴代码

---

## 文件说明

| 文件 | 用途 | 必需 |
|---|---|---|
| `worker/_worker.js` | 前端 + 后台 + 检测节点代码（部署时自动上传），单文件 | ✅ |
| `node/server.mjs` | VPS 上运行的 Node 版检测后端，Node.js 20+，无依赖 | 可选 |


---

## 快速部署

### 1. 部署前端 Worker
1. Cloudflare 控制台 → Workers & Pages → 创建 → 创建 Worker
2. 编辑代码，粘贴 `worker/_worker.js` 的全部内容 → 部署

### 2. 绑定 KV
1. 存储和数据库 → KV → 创建命名空间（名字随意，比如 `checkproxyip`）
2. Worker → 设置 → 绑定 → 添加 → KV 命名空间，**变量名填 `KV`**

### 3. 设置变量
Worker → 设置 → 变量和机密：

| 变量 | 类型 | 必需 | 说明 |
|---|---|---|---|
| `ADMIN` | 密钥 | ✅ | 后台登录密码，至少 8 位；同时用来加密 KV 里保存的令牌 |
| `CHECK_NODES` | 文本 | | 固定节点（比如 VPS），用逗号分隔，会和后台的节点合并 |
| `DAILY_LIMIT` | 文本 | | 每个账号每天的请求上限，默认 `100000`（付费版可以改大） |

修改设置后需要重新部署一次才会生效。

### 4. 创建 API 令牌
在 [API 令牌页面](https://dash.cloudflare.com/profile/api-tokens) → 创建令牌 → 自定义令牌，权限：

| 权限 | 用途 |
|---|---|
| 账户 · **Workers 脚本** · 编辑 | 部署、更新、删除检测节点 |
| 账户 · **账户设置** · 读取 | 自动获取 Account ID（没有这个权限就在后台手动填写） |
| 账户 · **账户分析** · 读取 | 用量统计（不需要统计可以不加） |

每个要用的 Cloudflare 账号各创建一个令牌。

### 5. 部署检测节点
1. 打开 `https://你的前端域名/admin`，用 `ADMIN` 的值登录
2. 添加账号：粘贴令牌 → 添加账号
3. 点该账号的「部署」，或者点「部署到全部账号」
4. 日志显示 `✓ 部署成功：https://check-node.xxx.workers.dev`，过几秒点「测试」，显示机房和延迟就说明正常
5. 回到检测页 `/` 刷新，右下角「检测节点」会显示节点数量

> 新账号第一次开通 workers.dev 子域，可能需要等 1–2 分钟才能访问。

---

## 后台管理

访问 `/admin`。

### 账号
- **添加**：填入令牌，自动识别令牌下的账号。按 Account ID 去重，已经存在的账号会跳过
- **部署**：给单个账号部署检测节点；账号没有 workers.dev 子域时会自动创建
- **部署到全部账号**：对列表里每个账号都执行一次部署。已部署的账号会用最新代码覆盖同名 Worker（地址不变，相当于更新）
- **移除**：从 KV 删除令牌；已经部署的 Worker 不会被删除

### 节点
- **启用 / 禁用**：禁用的节点不会出现在检测页
- **测试**：请求节点的 `/cdn-cgi/trace`，显示所在机房和延迟
- **更新代码**：把最新的检测代码重新上传到该节点
- **全部更新代码**：替换前端 Worker 代码后，用它把新的检测代码推送到所有已部署的节点
- **删除**：可以选择同时删除 Cloudflare 账号里的 Worker
- **手动添加**：添加 VPS 或其它地方的节点

后台的节点改动约 1 分钟内对检测页生效。

### 推荐操作
- **新加了账号**：只在新账号那一行点「部署」
- **更新了检测代码**：点「全部更新代码」
- **改了 Worker 名称**再点「部署到全部账号」：老账号会多出一个新 Worker，原来那个仍然存在，需要手动删除旧节点

---

## 检测节点调度

检测页的节点来源（合并后去重）：
1. 前端 Worker 的 `CHECK_NODES` 变量
2. 后台启用的节点（KV）
3. 检测页右下角面板里临时添加的节点（只保存在当前浏览器）

### 分配规则
- **所有节点一起用**：每检测一个 IP 就轮到下一个节点，起点随机，各账号的用量会差不多同时增长
- **失败自动切换**：节点返回 429、5xx、网络错误或不是 JSON 时，换下一个节点重试，最多试 3 个
- **冷却**：出错的节点暂停 60 秒，期间优先使用其它节点；60 秒后再试一次
- **正常的“IP 不可用”结果不会触发切换**

某个账号额度用完后检测还能继续，由其它账号承担；所有账号都用完时检测才会失败。

### 检测页节点面板
右下角「检测节点 N/M」按钮（启用数 / 总数）：
- 显示每个节点的状态（可用 / 检测中 / 冷却中 / 已禁用）、成功和失败次数、成功率、平均耗时、最近一次错误
- 可以临时禁用节点、测试、添加节点（只保存在当前浏览器）
- 浏览器控制台执行 `getCheckNodeStats()` 可以查看统计数据

> 面板里的统计只是本次打开页面后在当前浏览器里的计数，不是 Cloudflare 后台的真实用量。

---

## 用量统计

后台账号列表的「今日请求」列：
- 当天请求数 / 上限、百分比和进度条，用到 70% 变黄、90% 变红
- 鼠标停在上面，可以看这个账号下每个 Worker 各用了多少
- 标题旁显示所有账号的合计用量，以及距离重置还有几小时
- 打开后台时自动查询，之后每 5 分钟刷新；也可以点「刷新用量」
- 有账号超过 90% 时，操作日志会提示

说明：
- 免费版额度按 **UTC 0 点（北京时间 08:00）** 重置
- 统计的是**整个账号所有 Worker 的总请求数**（和 10 万次额度的计算方式一致）
- 数据来自 Cloudflare GraphQL Analytics，有几分钟延迟
- 令牌没有「账户分析 · 读取」权限时，这一列会显示权限提示，其它功能不受影响

---

## 可选：VPS Node 后端

`node/server.mjs` 是原 Worker 检测代码的 Node 版本，**没有请求次数和 CPU 限制**，可以作为节点加入后台。`connect()` 用 Node 的 `net` / `tls` 实现，HTTPS、SSTP 代理所需的 `secureTransport` 也支持。

```bash
# Node.js 20+
PORT=8788 COLO=HKG node server.mjs
curl "http://127.0.0.1:8788/check?proxy=socks5://1.2.3.4:1080"
```

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PORT` | 8788 | 监听端口 |
| `HOST` | 0.0.0.0 | 监听地址 |
| `COLO` | HKG | 前端显示的后端位置，填离 VPS 最近的三字码（HKG / NRT / SIN / LAX…） |
| `TRUST_PROXY` | 0 | 放在 Cloudflare / Nginx / Caddy 后面时设为 1 |

systemd 长期运行：
```ini
[Unit]
Description=CheckSocks5
After=network.target

[Service]
WorkingDirectory=/opt/checksocks5
ExecStart=/usr/bin/node server.mjs
Environment=PORT=8788
Environment=COLO=HKG
Environment=TRUST_PROXY=1
Restart=always
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

需要 HTTPS（检测页是 HTTPS，浏览器不允许请求 HTTP 节点）：
```bash
caddy reverse-proxy --from check.example.com --to 127.0.0.1:8788
```
然后在后台手动添加节点 `https://check.example.com`。

> ⚠️ VPS 检测时，流量从 VPS 自己的 IP 连到代理，而不是从 Cloudflare 的机房连过去；某些代理对来源 IP 有限制，结果可能和 Cloudflare 节点不同。

---

## 接口说明

### 前端 Worker
| 路径 | 说明 |
|---|---|
| `/` | 检测页 |
| `/admin` | 后台 |
| `/api/nodes` | 公开：启用的节点列表 `{"nodes":[...]}` |
| `/locations` | Cloudflare 机房列表（缓存 1 天） |
| `/node.js` | 查看会上传到各账号的检测节点代码 |
| `/api/admin/login` | POST `{password}` 登录 |
| `/api/admin/logout` | 退出 |
| `/api/admin/state` | GET 账号和节点列表（不含令牌） |
| `/api/admin/accounts` | POST `{token, accountId?, label?}` 添加账号 |
| `/api/admin/accounts/<id>` | DELETE 移除账号 |
| `/api/admin/deploy` | POST `{account, name?}` 部署 |
| `/api/admin/redeploy` | POST `{url}` 更新节点代码 |
| `/api/admin/nodes` | POST 添加 / PATCH `{url, enabled}` 启用或禁用 / DELETE `{url, undeploy?}` 删除 |
| `/api/admin/usage` | GET 今日用量 |

### 检测节点（check-node）
所有接口都允许跨域，与原项目一致。

| 路径 | 说明 |
|---|---|
| `/check?proxy=socks5://user:pass@host:1080` | 检测；也支持 `?socks5=` `?http=` `?https=` `?turn=` `?sstp=` 的写法 |
| `/resolve?proxyip=域名` | 解析域名（前端已改为在浏览器里解析，保留兼容） |
| `/resolve-batch` | POST 批量解析 |
| `/ip.json` | 访客 IP 信息 |
| `/locations` | 机房列表 |
| `/cdn-cgi/trace` | Cloudflare 自带，显示节点所在机房 |

---

## 安全设计
- **令牌加密存储**：用 `ADMIN` 派生的密钥 AES-GCM 加密后存入 KV，接口不返回令牌，页面只显示末 4 位
- **会话**：HMAC 签名 Cookie，HttpOnly + Secure + SameSite=Strict，7 天有效
- **防 CSRF**：所有写操作校验 Origin，必须来自本站
- **防暴力破解**：密码错误时延迟返回
- **没有开放转发**：调用 Cloudflare API 都要先登录后台，再由服务端发起

⚠️ 注意：
- 修改 `ADMIN` 后，已保存的令牌就解不开了，需要删除账号重新添加
- 令牌只填在你自己部署的后台；不要在别人搭建的站点上填写
- 用多个账号叠加额度，请自行确认符合 Cloudflare 的服务条款

---

## 常见问题

**后台提示“未绑定 KV”或“未设置 ADMIN”**
检查 KV 的变量名是否为 `KV`、变量名是否为 `ADMIN`，改完后重新部署。

**添加账号提示 `Authentication error` / `10000`**
令牌权限不够，需要 Workers 脚本 · 编辑 + 账户设置 · 读取。

**提示“令牌下没有账号”**
令牌缺少「账户设置 · 读取」，可以手动填写 Account ID（控制台右侧边栏能找到）。

**部署成功但测试不通**
新账号首次开通 workers.dev 需要等 1–2 分钟。

**用量一列显示权限提示**
到 Cloudflare 编辑令牌，加上「账户分析 · 读取」后点「刷新用量」；仍然不行就删除账号重新添加。

**原项目没有鉴权，公开后谁都能用检测接口**
本版本同样不限制检测页和节点的访问（后台有密码保护）。如需私有，请在 Cloudflare 侧加 Access 等访问控制。

**检测结果和原版不一致**
直接访问 `节点地址/check?proxy=socks5://host:port` 对比 JSON。不同账号的 Worker 可能落在不同机房，结果可能略有差异。

**能部署到 Cloudflare Snippets 吗？**
不能。Snippets 不支持 `connect()`，无法建立 TCP 连接。

**为什么前端不用 Pages？**
Pages 托管静态页面不限次数，但后台需要 Pages Functions，这部分调用同样计入 Workers 额度，而且带 Functions 的项目不能在控制台拖拽上传。前端请求量本来就很小，用单个 Worker 部署更简单。

---

## 与原项目的区别

| 项目 | 原项目 | 本版本 |
|---|---|---|
| 部署 | 单个 Worker | 前端 Worker + 多个检测节点 |
| 每日额度 | 10 万次 | N 个账号 × 10 万次 |
| 域名解析 | Worker 请求 DoH | 浏览器直接请求 DoH（解析规则相同；sstp / https 仍保留域名用于 SNI） |
| 节点管理 | 无 | KV 后台：账号、一键部署、更新、删除、用量统计 |
| 失败处理 | 无 | 自动切换节点 + 冷却 |
| 检测逻辑 | — | 与原项目一致（SOCKS5 / HTTP / HTTPS / TURN / SSTP） |
| VPS 部署 | 不支持 | 可选 Node 后端 |

## 致谢与许可
- 检测逻辑来自 [cmliu/CF-Workers-CheckSocks5](https://github.com/cmliu/CF-Workers-CheckSocks5)，TLS 客户端作者 @Alexandre_Kojeve
- 原项目使用 **GPL-3.0** 许可，本修改版同样遵循 GPL-3.0；分发时请保留许可证和原作者信息
