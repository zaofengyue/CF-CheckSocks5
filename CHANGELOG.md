# 更新日志 (CHANGELOG)

本项目遵循语义化版本规范，记录所有重要功能更新与修复。

---

## [v1.2.0] - 2026-10-02

### 🚀 新增功能
1. **出口 IP 接口多源容灾与前端面板选项切换**：
   - 在 Workspace 控制面板新增**出口 IP 接口选择卡片**，与“批量检测开关”和“开始检测按钮”并排整合，完美契合深蓝磨砂玻璃质感。
   - 支持 5 大特色服务源自由切换：
     - **`iplocate.io`**：默认综合地理定位与 ASN 组织。
     - **`api.ip.sb`**：极速 GeoIP 解析，毫秒级响应，大批量测试推荐。
     - **`api.ipapi.is`**：原生识别机房/数据中心 IP、VPN 标记与纯净度风控。
     - **`ipinfo.io`**：全球知名老牌权威 IP 数据库。
     - **`Cloudflare 官方 Trace (1.1.1.1/cdn-cgi/trace)`**：基于 Workers 骨干网络直连，零风控、无额度上限，**1w+ 超大批量测活首选**。
   - **本地自动持久化**：用户所选的查询源自动保存在浏览器 `localStorage` 中，刷新页面不丢失。
   - **后端协议适配**：`checkProxy` 统一根据前端传入的 `source` 参数建立对应 Host 的 TLS 隧道，针对 JSON 及 Cloudflare key=value 纯文本格式进行统一标准化清洗。

2. **支持测试失败节点导出（剪贴板 / TXT / CSV）**：
   - 彻底解除以往只允许导出 `status === 'success'` 节点的硬性限制。
   - **与表格筛选标签页完美联动**：
     - 在“失败”筛选页下，点击导出按钮将专门导出所有测试失败的节点。
     - 在“全部”筛选页下，导出全部测试节点（包含成功与失败）。
     - 在“有效”筛选页下，保持仅导出有效节点的原有体验。
   - **多格式导出优化**：
     - **剪贴板 / TXT**：失败节点格式为 `socks5://... #[检测失败: 错误原因]`，方便比对排查。
     - **CSV 文件**：新增 `STATUS`（成功/失败）与 `ERROR`（详细错误信息，如超时、握手失败、HTTP 401 等）首列，方便使用 Excel 等表格软件直接筛选过滤。

3. **GitHub Releases 自动打包与发布 Cloudflare Pages 部署包**：
   - 优化 GitHub Actions 工作流，推送至 `beta` 分支时自动生成 `beta` 预发布版本，推送至 `main` 分支或版本标签时自动发布正式版。
   - 自动生成符合 Cloudflare Pages Direct Upload 规范的最外层直装包 `CF-CheckSocks5-Pages.zip`，并上传为 Release 附件供随时下载部署。

### 🔒 隐私与安全性优化
1. **彻底移除第三方访客统计脚本**：
   - 移除页脚原有的 `https://tongji.090227.xyz` 统计脚本及 `#visit-count` 元素。
   - 确保节点测试过程完全私密，不向任何未授权第三方服务器上报访客域名与节点信息。

---

## [v1.1.0] - 2026-10-02

### 🚀 新增功能
1. **全功能 Token 访问鉴权**：
   - 服务端原生支持通过环境变量 `TOKEN` 启用访问鉴权。
   - 支持 HTTP Header (`X-Token` / `Authorization: Bearer <token>`) 与 URL 路径直通 (`/<token>`) 双模式。
   - 前端增加全屏密码锁遮罩（Auth Lock Overlay），未授权用户禁止访问检测功能与节点数据。
2. **多部署模式兼容**：
   - 新增 `_routes.json` 规则配置。
   - 适配 Cloudflare Pages Direct Upload 静态打包部署与 GitHub Actions 自动构建工作流。
