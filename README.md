# OptikLink 每日自动登录

> 利用 Discord OAuth2 Token 自动完成 OptikLink 每日登录，通过 GitHub Actions 定时执行，并将结果推送至微信（WxPusher）。

---

## 功能特性

- 🤖 **全自动**：每天北京时间 09:00 自动运行，无需手动操作
- 🛡️ **绕过 Cloudflare**：使用 `cloudscraper` 处理人机验证
- 🌐 **代理支持**：可选接入 Xray / V2Ray SOCKS5 代理出口
- 🔄 **client_id 自动更新**：检测到 OAuth 参数变更时，自动切换并更新 GitHub Secret
- 📲 **WxPusher 推送**：登录结果、到期预警实时推送至微信
- 🔒 **安全设计**：所有凭证均通过 GitHub Secrets 管理，日志全部脱敏输出

---

## 工作原理

```
GitHub Actions 定时触发
        │
        ▼
[A] 访问 /auth 页面，动态探测 OAuth 参数
        │
        ▼
[B] 用 Discord Token 向 Discord API 提交 OAuth 授权，获取回调 code
        │
        ▼
[C] 携带 code 访问 OptikLink 回调地址，完成登录（cloudscraper 绕过 CF）
        │
        ▼
[D] 访问 Dashboard，读取登录状态、用户名、到期时间、服务器数量
        │
        ▼
WxPusher 推送执行报告（含到期剩余天数提醒）
```

---

## 快速开始

### 1. Fork 本仓库

点击右上角 **Fork**，将项目复制到你自己的 GitHub 账号下。

### 2. 配置 GitHub Secrets

进入仓库 → **Settings → Secrets and variables → Actions → New repository secret**，依次添加以下 Secret：

| Secret 名称 | 必填 | 说明 |
|---|---|---|
| `DISCORD_TOKEN` | ✅ | 你的 Discord 账号 Token（不是 Bot Token） |
| `WXPUSHER_TOKEN` | ✅ | WxPusher 应用的 `appToken` |
| `WXPUSHER_UID` | ✅ | WxPusher 接收消息的用户 UID |
| `DISCORD_CLIENT_ID` | ✅ | OptikLink 的 Discord OAuth2 client_id |
| `DISCORD_REDIRECT_URI` | ✅ | OAuth2 回调地址（如 `https://optiklink.net/callback`） |
| `EXPIRE_DATE` | ⬜ | 服务到期日，格式 `DD.MM.YYYY`（用于到期提醒） |
| `V2RAY_CONFIG` | ⬜ | Xray/V2Ray 配置文件 JSON 内容（不需代理可不填） |

> **Discord Token 获取方式**：在浏览器打开 Discord Web 版，按 F12 → Network，随意点击一个请求，找到请求头中的 `Authorization` 字段值即为 Token。请勿泄露，妥善保管。

### 3. 启用 Workflow

进入仓库 → **Actions** 页面，如提示未启用，点击 **I understand my workflows, go ahead and enable them**。

首次可点击 **Run workflow** 手动触发，验证配置是否正确。

---

## 定时计划

```yaml
schedule:
  - cron: "0 1 * * *"   # UTC 01:00 = 北京时间 09:00，每天执行
```

如需修改时间，编辑 `.github/workflows/optiklink.yml` 中的 `cron` 表达式即可。

---

## WxPusher 配置

1. 访问 [WxPusher 官网](https://wxpusher.zjiecode.com/) 注册并创建应用，获取 `appToken`
2. 关注 WxPusher 公众号，扫码绑定账号，获取你的 `UID`
3. 将 `appToken` 和 `UID` 填入对应 Secret

推送示例：

```
OptikLink 签到 | ✅ 登录成功

| 项目       | 内容                    |
|----------|----------------------|
| 状态       | ✅ 登录成功               |
| 用户名      | yourname#0           |
| 运行服务器    | 1557 个               |
| 服务到期     | 23.05.2026           |
| 剩余天数     | 8 天                  |
| 执行时间     | 2026-05-15 01:00 UTC |
```

**到期提醒规则：**

| 剩余天数 | 提醒级别 |
|---|---|
| > 30 天 | 无提醒 |
| ≤ 30 天 | 正文显示剩余天数 |
| ≤ 7 天 | ⚠️ 标题警告 |
| ≤ 3 天 | 🚨 紧急警告 |

---

## client_id 自动更新

脚本在每次执行时会访问 `/auth` 页面，动态解析最新的 OAuth 参数。若检测到 `client_id` 与 Secret 中保存的值不一致：

1. 本次自动切换为新值继续执行，**不会中断登录**
2. 将新值写入 `GITHUB_OUTPUT`，Workflow 自动调用 `gh secret set` 更新 Secret
3. 同步通过 WxPusher 推送变更通知

正常情况下无需手动维护 `DISCORD_CLIENT_ID`。

---

## 目录结构

```
├── .github/
│   └── workflows/
│       └── optiklink.yml      # GitHub Actions 工作流
└── optiklink_login.py         # 主脚本
```

---

## 常见问题

**Q：运行报错 `Missing dependencies for SOCKS support`**

在 `optiklink.yml` 的安装依赖步骤中确认包含 `requests[socks]`：

```yaml
run: pip install requests cloudscraper "requests[socks]"
```

**Q：Discord 授权报 401 / Token 失效**

Discord Token 可能因以下原因失效：重置密码、被 Discord 检测、Token 被修改。重新获取 Token 后更新 Secret 即可。

**Q：Cloudflare 验证失败**

`cloudscraper` 版本过旧可能导致绕过失败。可在 yml 中将安装命令改为：

```yaml
run: pip install "cloudscraper>=1.2.71" "requests[socks]"
```

**Q：需要代理但没有 V2Ray 配置**

不填 `V2RAY_CONFIG` Secret 时，脚本会直连运行，不启用 SOCKS5 代理。

---

## 免责声明

本项目仅供学习和个人自动化使用。使用前请确认符合 OptikLink 及 Discord 的服务条款。因使用本脚本导致的账号风险由使用者自行承担。
