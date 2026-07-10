# Weirdhost 部署 + 续期保姆级教程

> 本教程面向**完全没操作过**的用户，每一步都标明"在哪点哪个按钮"。
> 操作前请先读完 [WEIRDHOST.md](WEIRDHOST.md) 的"防关联清单"——前账号被封过的话，**新号必须用全新身份**。

---

## 📋 总览：6 步搞定

| 步骤 | 操作 | 耗时 |
|------|------|------|
| ① 注册 Weirdhost | 邮箱注册 + 验证 | 5 分钟 |
| ② 创建 Node.js 服务器 | 在面板申请容器 | 3 分钟 |
| ③ 配置 Cloudflare Argo 隧道 | CF 后台建隧道 | 10 分钟 |
| ④ 在 Weirdhost 上传代码 + 环境变量 | 部署 sbx-native | 10 分钟 |
| ⑤ 启动并取订阅 | 验证节点可用 | 2 分钟 |
| ⑥ 配置自动续期 | GitHub Secrets + 测试 | 10 分钟 |

**总计 ~40 分钟**。建议一次性做完，避免中途 cookie 过期。

---

## ① 注册 Weirdhost 账号

### 1.1 准备全新身份（被前号封过的用户必读）

| 维度 | 推荐方案 |
|------|----------|
| 邮箱 | 全新 [ProtonMail](https://proton.me/)（不要用之前被封号的邮箱） |
| IP | 手机 4G 流量（不要用之前注册过的 WiFi/VPN） |
| 浏览器 | Firefox / Edge 隐身窗口（不要用之前登录过的 Chrome profile） |

### 1.2 注册流程

1. 访问 **https://hub.weirdhost.xyz**
2. 点击右上角 **"회원가입"**（韩语：注册）
3. 填写表单：
   - 아이디（用户名）：自定义
   - 이메일（邮箱）：填新邮箱
   - 비밀번호（密码）：自定义
4. 提交后到邮箱收验证邮件，点验证链接
5. 回到 https://hub.weirdhost.xyz 登录

> ⚠️ 如果收不到验证邮件：检查垃圾箱，或换一个邮箱服务商重试。QQ/163 邮箱大概率收不到，用 Gmail / ProtonMail / Outlook。

---

## ② 创建 Node.js 服务器

### 2.1 进入面板

登录后默认进入面板首页，看到 **"서버 만들기"**（创建服务器）按钮。

### 2.2 创建服务器

1. 点击 **"서버 만들기"**
2. 选择服务器类型：**Node.js**（不要选 Java Paper / Minecraft）
3. 选择版本：**Node.js 18**（或更高，至少 16+）
4. 服务器名称：随意（如 `myapp`）
5. 资源套餐：选**免费套餐**（무료）
6. 点击 **"만들기"**（创建）

### 2.3 等待创建完成

- 创建过程约 1-2 分钟
- 完成后服务器列表会显示一个**公网域名**，形如：
  ```
  https://yourname.weirdhost.xyz
  ```
- **记下这个域名**，后面要用

### 2.4 查看服务器详情

点击刚创建的服务器，进入详情页。能看到：
- **도메인**（域名）：公网访问地址
- **포트**（端口）：对外端口（一般是 3000 或 80）
- **환경 변수**（环境变量）：配置入口
- **시작/중지**（启动/停止）按钮
- **파일 관리**（文件管理）：上传代码入口

---

## ③ 配置 Cloudflare Argo 隧道

Weirdhost 容器只暴露 1 个端口，VMess Argo 通过 Cloudflare 隧道反代到容器内 8001 端口，**不需要额外端口**。

### 3.1 注册 Cloudflare 账号（必须新账号）

1. 访问 https://dash.cloudflare.com/sign-up
2. 用**另一个新邮箱**注册（不要用 Weirdhost 同一个邮箱）
3. 验证邮箱

### 3.2 添加域名

1. 登录 CF → 右上角 **"Add a site"**
2. 输入你的域名（如果没有，可以：
   - 在 [Freenom](https://freenom.com) 免费注册 `.tk`/`.ml` 域名（不稳，备选）
   - 在 Namesilo / Porkbun 花 $1 买个 `.xyz`/`.top` 域名（推荐）
3. 选择 **Free 计划**（免费）
4. 按提示到域名注册商修改 **Nameserver** 为 CF 提供的两个 NS
5. 等待 NS 生效（5-30 分钟）

### 3.3 创建 Argo Tunnel

1. CF 后台左侧 → **Zero Trust**（首次需点 "Get started" 免费开通）
2. 左侧菜单 → **Networks → Tunnels**
3. 点击 **"Create a tunnel"**
4. 选择 **Cloudflared** → Next
5. 给 tunnel 命名：`wh-sbx`（自定义）
6. **保存**后页面会显示一个 **Token**，形如：
   ```
   eyJhIjo...
   ```
   **完整复制这个 Token**，后面要用

### 3.4 配置 Public Hostname

1. 在刚创建的 tunnel 详情页 → **Public Hostname** 标签
2. 点击 **"Add a public hostname"**
3. 填写：
   - **Subdomain**：`argo`（自定义，最终域名是 `argo.your-domain.com`）
   - **Domain**：选你的域名
   - **Path**：留空
   - **Service Type**：`HTTP`
   - **URL**：`localhost:8001`
4. 点击 **"Save hostname"**

### 3.5 记下关键信息

到这一步，您应该有：
- ✅ Argo 域名：`argo.your-domain.com`
- ✅ Argo Token：`eyJhIjo...`（长字符串）
- ✅ Weirdhost 域名：`yourname.weirdhost.xyz`

---

## ④ 在 Weirdhost 上传代码 + 环境变量

### 4.1 上传 sbx-native 代码

Weirdhost 面板的文件管理界面操作：

1. 进入 Weirdhost 服务器详情 → **"파일 관리"**（文件管理）
2. 进入 `domains/yourname.weirdhost.xyz/public_nodejs/` 目录（这是 Node.js 应用根目录）
3. **方案 A（推荐）：从 GitHub clone**

   在 Weirdhost 找到 "터미널"（终端）或 SSH 入口，执行：
   ```bash
   cd ~/domains/yourname.weirdhost.xyz/public_nodejs
   git clone https://github.com/weikkadd/skx-kuyt.git /tmp/sbx
   cp /tmp/sbx/nodejs/* .
   cp -r /tmp/sbx/nodejs/.* . 2>/dev/null
   npm install
   ```
4. **方案 B：手动上传**

   从 https://github.com/weikkadd/skx-kuyt/tree/main/nodejs 下载这 2 个文件：
   - `index.js`
   - `package.json`
   
   通过 Weirdhost 文件管理器的"上传"功能传到 `public_nodejs/` 目录

### 4.2 生成必要参数

在本地终端（任何能跑 Python 的电脑）生成：

```bash
# 1. 生成 UUID
python3 -c "import uuid; print(uuid.uuid4)"

# 2. 生成 8 位随机 SUB_PATH
python3 -c "import secrets,string; print(''.join(secrets.choice(string.ascii_lowercase+string.digits) for _ in range(8)))"
```

记下：
- UUID（例：`8f3a2c4d-1234-5678-9abc-def012345678`）
- SUB_PATH（例：`x9k2m7p3`）

### 4.3 配置环境变量

在 Weirdhost 服务器详情 → **"환경 변수"**（环境变量），逐条添加：

| 变量名 | 值 |
|--------|-----|
| `UUID` | 上面生成的 UUID |
| `NAME` | `WH-A`（自定义标识） |
| `PORT` | `3000` |
| `SUB_PATH` | 上面生成的随机串 |
| `DISABLE_ARGO` | `false` |
| `ARGO_DOMAIN` | `argo.your-domain.com` |
| `ARGO_AUTH` | Cloudflare Token `eyJhIjo...` |
| `ARGO_PORT` | `8001` |
| `CFIP` | `saas.sin.fan`（默认） |
| `CFPORT` | `443` |
| `PROJECT_URL` | `https://yourname.weirdhost.xyz` |
| `AUTO_ACCESS` | `true` |
| `FILE_PATH` | `.npm` |
| `CHAT_ID` | （可选）你的 TG Chat ID |
| `BOT_TOKEN` | （可选）你的 TG Bot Token |

> ⚠️ **不要设置** `REALITY_PORT` / `HY2_PORT` / `TUIC_PORT` / `ANYTLS_PORT` / `S5_PORT`——Weirdhost 不放行额外端口，开了反而暴露特征。

### 4.4 配置启动命令

在 Weirdhost 服务器详情 → **"시작 명령"**（启动命令）：

```
node index.js
```

或如果面板支持 `package.json` scripts，确保 `package.json` 里：
```json
{
  "scripts": {
    "start": "node index.js"
  }
}
```

---

## ⑤ 启动并取订阅

### 5.1 启动服务器

1. 回到服务器详情页
2. 点击 **"시작"**（启动）
3. 等待 30 秒
4. 点击 **"로그"**（日志）查看启动日志

### 5.2 检查启动成功

日志里应该看到（类似）：
```
web is running
HTTP server is listening on 3000
sub.txt saved successfully
```

如果看到 `Downloading sbx.so ...` 后报错，是网络问题，重启一次。

### 5.3 取订阅

浏览器访问：
```
https://yourname.weirdhost.xyz/<SUB_PATH>
```

例如 `SUB_PATH=x9k2m7p3`，访问：
```
https://yourname.weirdhost.xyz/x9k2m7p3
```

页面会显示一长串 base64 字符串，**这就是订阅内容**。

### 5.4 导入客户端

#### v2rayN（Windows）
1. 打开 v2rayN → 订阅 → 订阅设置 → 新增
2. 地址（URL）：填上面的订阅 URL
3. 保存 → 更新订阅

#### Clash Meta / Mihomo
1. 配置 → Profiles → 粘贴订阅 URL → Download
2. 选中该 Profile，启用

#### sing-box 客户端
1. Profile → Add Profile → Remote
2. URL 填订阅地址 → Save → Update

#### Shadowrocket（iOS）
1. + → Subscribe → 粘贴 URL → Save

### 5.5 验证节点可用

在客户端选中刚导入的节点（名称形如 `WH-A-VMess-Argo`），访问 https://www.google.com 验证连通。

---

## ⑥ 配置自动续期

Weirdhost 服务器默认有效期短（一般几天），到期会被回收。本仓库已配置续期 workflow，每天自动续期。

### 6.1 获取 Weirdhost Cookie

1. 浏览器登录 https://hub.weirdhost.xyz
2. 按 **F12** 打开开发者工具
3. 顶部标签 → **Application**（Edge/Chrome）或 **Storage**（Firefox）
4. 左侧 → **Cookies** → `https://hub.weirdhost.xyz`
5. 找到以 `remember_web_` 开头的 Cookie，**完整复制** `名称=值`

例如：
```
remember_web_59ba36addc2b2f940CCCC=eyJpdiI6IkJ...
```

### 6.2 配置 GitHub Secrets

1. 访问 https://github.com/weikkadd/skx-kuyt
2. 顶部 → **Settings** → 左侧 **Secrets and variables** → **Actions**
3. 点击 **"New repository secret"**
4. 逐条添加：

| Name | Value（示例） | 必填 |
|------|---------------|------|
| `WEIRDHOST_COOKIE_1` | `WH-A-----remember_web_xxx=yyy` | ✅ |
| `WEIRDHOST_COOKIE_2` | `WH-B-----remember_web_xxx=yyy`（第二个号） | 可选 |
| `REPO_TOKEN` | GitHub PAT（见 6.3） | 推荐 |
| `TG_BOT_TOKEN` | `123456789:ABC-XYZ...` | 可选 |
| `TG_CHAT_ID` | `123456789` | 可选 |

> Cookie 格式：`备注-----Cookie键=Cookie值`（5 个连字符作分隔符）

### 6.3 生成 GitHub PAT（用于自动刷新 Cookie）

1. 访问 https://github.com/settings/tokens
2. **Generate new token (classic)**
3. 勾选权限：`repo`（完整勾选）+ `workflow`
4. 生成后**立即复制**（关闭页面就看不到了）
5. 把这个 token 配置到仓库 Secret `REPO_TOKEN`

### 6.4 手动触发首次续期

1. 访问 https://github.com/weikkadd/skx-kuyt/actions
2. 左侧选 **"Weirdhost 自动续期"**
3. 右上角 **"Run workflow"** → Branch 选 `main` → **Run workflow**
4. 等待 1-2 分钟，点进去看日志

### 6.5 检查运行结果

**成功标志**：日志里看到
```
账号 WH-A 续期成功
```
或韩语点击成功的截图（artifact 会保存 3 天）

**失败排查**：
- Cookie 过期 → 重新获取 Cookie 更新到 Secret
- 网络问题 → 重试
- 截图上传 artifact → 下载查看具体卡在哪步

### 6.6 配置 Telegram 通知（可选但推荐）

1. 在 Telegram 找 [@BotFather](https://t.me/BotFather)
2. 发送 `/newbot` 创建一个 bot
3. 记下 Bot Token（形如 `123456789:ABC-XYZ...`）
4. 把 bot 加入你的频道或群组，发一条消息
5. 访问 `https://api.telegram.org/bot<TOKEN>/getUpdates` 获取 `chat_id`
6. 把 Token 和 Chat ID 配置到 Secrets

续期成功/失败都会发 TG 通知。

---

## 🚨 常见问题

### Q1：启动后日志显示 `sbx.so download failed`
**A**：Weirdhost 网络偶发问题，重启 1-2 次。如果持续失败，可能是架构不对（确认容器是 amd64）。

### Q2：访问订阅 URL 返回 404
**A**：`SUB_PATH` 没对上。检查环境变量里的 `SUB_PATH` 和访问 URL 路径是否一致。

### Q3：订阅内容为空
**A**：环境变量没配置好。检查：
- `ARGO_AUTH` 是否完整 token
- `ARGO_DOMAIN` 是否和 CF 隧道配置一致
- 启动日志是否有 `argo tunnel domain: argo.your-domain.com`

### Q4：节点连不上
**A**：检查：
- 客户端配置的端口是 443（不是 8001）
- 客户端配置的 host 和 path 是 `argo.your-domain.com` 和 `/vmess`
- CFIP 用 `saas.sin.fan` 或自定义优选 IP

### Q5：续期 workflow 报错 "Cookie expired"
**A**：Cookie 已过期，需要重新登录 hub.weirdhost.xyz 获取新 Cookie，更新到 GitHub Secret。

### Q6：续期 workflow 找不到续期按钮
**A**：Weirdhost 改版了 UI。检查下载的截图 artifact 看页面长啥样，可能需要更新 `scripts/weirdhost_renew.py` 里的 selector。

### Q7：又被封了怎么办
**A**：参考 [WEIRDHOST.md](WEIRDHOST.md) 第五节"被封后的应急流程"。核心是**所有身份重新生成**（邮箱/IP/CF账号/UUID/SUB_PATH），不要复用任何旧配置。

---

## 📚 相关文件

- [WEIRDHOST.md](WEIRDHOST.md) — 防封原理 + 多账号隔离清单
- [README.md](../README.md) — 项目总说明
- [NODE.md](../NODE.md) — Node.js 版环境变量完整文档
- [.github/workflows/weirdhost-renew.yml](../.github/workflows/weirdhost-renew.yml) — 续期 workflow
- [scripts/weirdhost_renew.py](../scripts/weirdhost_renew.py) — 续期脚本源码

---

## 🎯 下一步

部署成功后建议：
1. **低调使用 1 周**：每天流量 < 1GB，只看网页/TG/ChatGPT，不看视频
2. **观察是否被风控**：如果 7 天没事，可以适度增加使用强度
3. **准备 Plan B**：同时部署一个 Serv00 账号备用（https://github.com/eooce/sbx-native 的 `serv00/ct8` 分支有一键脚本）
