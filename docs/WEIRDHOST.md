# Weirdhost 部署实战手册（防封版）

> 本文档针对前账号已被封过的用户，提供**降低再被封概率**的完整方案。
> 包含：防关联多账号管理、降低检测的 sbx-native 配置、自动续期 workflow。

---

## 一、为什么之前被封？大概率原因复盘

| 原因 | 概率 | 说明 |
|------|------|------|
| **多账号关联** | ~60% | 同 IP/同邮箱前缀/同 CF 账号/同 UUID → 一个被识别，全部连坐 |
| **流量特征异常** | ~25% | VMess Ws 长连接 + 持续高吞吐 ≠ 正常 MC 服务器流量曲线 |
| **进程/端口特征** | ~15% | 平台扫文件系统发现 `sbx.so` / `cert.pem` / `keypair.properties` |

---

## 二、再战前必读：防关联多账号管理清单

每个 Weirdhost 账号必须做到 **7 个完全独立**：

| 维度 | 账号 A | 账号 B | 账号 C | 说明 |
|------|--------|--------|--------|------|
| **注册邮箱** | `wh-a@proton.me` | `wh-b@tutanota.com` | `wh-c@gmail.com` | 不同邮箱服务、不同前缀 |
| **注册 IP** | 美国 VPN 节点 1 | 日本 VPN 节点 2 | 手机流量 | 不要用同一 VPN 出口 |
| **浏览器** | Chrome 主 profile | Firefox 新 profile | Edge 隐身 | 不同 fingerprint |
| **Cloudflare 账号** | CF 账号 A | CF 账号 B | CF 账号 C | Argo 隧道必须分散在不同 CF 账号下 |
| **Argo 域名** | `argo-a.yourdomain1.com` | `argo-b.yourdomain2.com` | `argo-c.yourdomain3.com` | 用不同域名 |
| **sbx UUID** | `uuid-a` | `uuid-b` | `uuid-c` | 全新生成，不复用 |
| **订阅路径 SUB_PATH** | `x9k2m7` | `p3q8n1` | `t5w6y2` | 随机字符串，不用默认 `sub` |

### 推荐身份工具

- **邮箱**：[ProtonMail](https://proton.me/)（免费、匿名、可注册多个）
- **VPN 分流**：[Warp+](https://1.1.1.1/) 或 [Mullvad](https://mullvad.net/)（不同节点 = 不同 IP）
- **浏览器隔离**：[Firefox Multi-Account Containers](https://addons.mozilla.org/firefox/addon/multi-account-containers/) 或 [Chrome Profile Switcher](chrome://settings/manageProfile)
- **UUID 生成**：`python3 -c "import uuid; print(uuid.uuid4())"` 或 https://www.uuidgenerator.net

### 一句话原则

**让平台从任何技术信号都看不出"这 3 个号是同一个人"。**

---

## 三、降低检测的 sbx-native 配置（Weirdhost 专用）

### 3.1 环境变量模板（在 Weirdhost 容器面板填）

只开 **VMess Argo**，禁用所有其他协议。复制后按提示替换 `<...>`：

```bash
# ============ 身份（必改）============
UUID=<新生成的 UUID-1，例如 8f3a2c4d-...>
NAME=WH-A

# ============ HTTP 订阅服务 ============
PORT=3000
SUB_PATH=<8 位随机字符串，例如 x9k2m7>
# 重要：把 SUB_PATH 改成随机字符串，平台扫 /sub 也能识别

# ============ Cloudflare Argo 隧道（必填）============
DISABLE_ARGO=false
ARGO_DOMAIN=<argo-a.your-domain.com>
ARGO_AUTH=<Cloudflare Tunnel Token，形如 eyJh...>
ARGO_PORT=8001

# ============ 协议端口（全部留空！）============
# Weirdhost 容器只暴露 1 个端口，开这些会被检测且用不了
S5_PORT=
TUIC_PORT=
HY2_PORT=
ANYTLS_PORT=
REALITY_PORT=

# ============ CF 优选（影响速度，不影响被封）============
CFIP=saas.sin.fan
CFPORT=443

# ============ 保活（指向 Weirdhost 容器公网 URL）============
PROJECT_URL=https://<你的容器域名>.weirdhost.xyz
AUTO_ACCESS=true

# ============ Telegram 通知（推荐配置）============
CHAT_ID=<你的 TG chat_id>
BOT_TOKEN=<你的 TG bot_token>

# ============ 运行目录（保持默认）============
FILE_PATH=.npm

# ============ 可选：让 YouTube 走 WARP 出站，降低流量特征 ============
YT_WARPOUT=false
# 注：YT_WARPOUT=true 需要 sing-box 配 WARP outbound，本仓库未默认配置
```

### 3.2 关键防封技巧

#### ✅ DO（必须做）

1. **改 `SUB_PATH` 为随机字符串**
   - 默认 `sub` 是公开特征，平台扫 `/sub` 路径就能识别 sbx-native
   - 用 8 位随机串：`python3 -c "import secrets,string; print(''.join(secrets.choice(string.ascii_lowercase+string.digits) for _ in range(8)))"`

2. **只开 VMess Argo，不开 Reality/HY2/TUIC**
   - 这些协议需要监听额外端口，Weirdhost 不放行 = 配置失败 + 进程特征暴露

3. **限制客户端连接数**
   - sbx-native 没有内置限速，在**客户端**配置里限制：
     - v2rayN：节点 → 编辑 → "Mux" 关闭，限速 5MB/s
     - Clash：`max-conn: 3`
   - 避免单 IP 同时 50+ 连接（典型代理特征）

4. **错峰使用**
   - 用完就断开，不要 24/7 在线
   - 避免长时间连续下行（看 4K 视频 6 小时 = 100% 被识别）

5. **每天流量 < 1GB**
   - 只看网页、TG、ChatGPT
   - 视频用 480p，下载用其他渠道

#### ❌ DON'T（绝对不要做）

1. **不要在两个号之间共享 UUID** — 平台扫配置文件特征
2. **不要在两个号之间共享订阅 URL** — 流量来源 IP 重合 = 关联
3. **不要把订阅上传到公开聚合站**（如 slingr、subconverter 公共实例）— 流量爆发 = 秒封
4. **不要在同一台机器上同时管理 3 个号** — 浏览器 fingerprint 必然关联
5. **不要用同一 Cloudflare 账号做多个 Argo 隧道** — CF API token 关联

---

## 四、自动续期 Workflow（已配置在本仓库）

本仓库已包含 `.github/workflows/weirdhost-renew.yml`，每天 UTC 04:20 自动跑一次，给 Weirdhost 账号"打卡续命"。

### 4.1 配置步骤

#### 第 1 步：获取 Weirdhost Cookie

1. 登录 https://hub.weirdhost.xyz
2. 浏览器 F12 → Application → Cookies → `https://hub.weirdhost.xyz`
3. 找到以 `remember_web_` 开头的 Cookie，复制完整的 `名称=值`

#### 第 2 步：在 GitHub 仓库配置 Secrets

进入 `weikkadd/skx-kuyt` → Settings → Secrets and variables → Actions → New repository secret：

| Secret 名称 | 值 | 必填 |
|-------------|-----|------|
| `WEIRDHOST_COOKIE_1` | `账号1备注-----remember_web_xxx=yyy` | ✅ |
| `WEIRDHOST_COOKIE_2` | `账号2备注-----remember_web_xxx=yyy` | 可选 |
| `WEIRDHOST_COOKIE_3` | `账号3备注-----remember_web_xxx=yyy` | 可选 |
| `WEIRDHOST_COOKIE_4` | `账号4备注-----remember_web_xxx=yyy` | 可选 |
| `WEIRDHOST_COOKIE_5` | `账号5备注-----remember_web_xxx=yyy` | 可选 |
| `REPO_TOKEN` | GitHub PAT（`repo` + `workflow` 权限） | 推荐 |
| `TG_BOT_TOKEN` | Telegram Bot Token | 可选 |
| `TG_CHAT_ID` | Telegram Chat ID | 可选 |

#### Cookie 格式说明

```
账号备注-----remember_web_xxx=yyy
```

- `账号备注`：自定义标识（如 `WH-A`），便于日志识别
- `-----`：固定分隔符（5 个连字符）
- `remember_web_xxx=yyy`：从浏览器复制的完整 Cookie

**示例**：
```
WH-A-----remember_web_59ba36addc2b2f940CCCC=eyJpdiI6IkJ...
```

#### 第 3 步：手动触发一次验证

进入仓库 Actions 页 → 左侧选 "Weirdhost 自动续期" → Run workflow → 看运行日志

### 4.2 续期机制原理

| 步骤 | 动作 |
|------|------|
| 1 | seleniumbase + xvfb 启动无头浏览器 |
| 2 | 注入 Cookie 登录 hub.weirdhost.xyz |
| 3 | 访问 `/server/` 页面，定位"연장하기"（韩语：续期）按钮 |
| 4 | 自动点击所有服务器的续期按钮 |
| 5 | 截图保存（失败时上传 artifact 供调试） |
| 6 | 如配置了 TG，推送续期结果 |
| 7 | 如配置了 REPO_TOKEN，自动刷新过期的 Cookie 到 Secrets |

---

## 五、被封后的应急流程

如果再次被封，按以下顺序处理：

1. **立即停用**：所有客户端断开连接，避免平台关联到新号
2. **保留证据**：截图被封提示、记录最后使用时间
3. **更换身份**（参考第二节清单）：
   - 新邮箱（不同邮箱服务商）
   - 新 IP（手机流量或换 VPN 出口）
   - 新 Cloudflare 账号
   - 新域名（或子域名）
   - 新 UUID + 新 SUB_PATH
4. **重新部署**：使用本文档第三节的配置
5. **配置续期**：fork 本仓库，按第四节配置 Secrets
6. **低调使用**：第一周不要看视频、不要分享订阅

---

## 六、长期建议

免费平台永远是"猫鼠游戏"。如果您：

- 已经被封 3 次以上
- 想要稳定 24/7 在线
- 需要看 4K 视频 / 大流量下载
- 不想每隔几周重新配置

**强烈建议花 $10-20/年买个 VPS**（RackNerd、BandwagonHost 等），跑同样的 sbx-native，永远不会被封，还能开所有协议。

参考对比：

| 方案 | 成本 | 稳定性 | 性能 | 协议支持 |
|------|------|--------|------|----------|
| Weirdhost（免费） | $0 | ⭐⭐（1-4 周被封） | ⭐⭐ | 仅 VMess Argo |
| Serv00（免费） | $0 | ⭐⭐⭐ | ⭐⭐ | 全协议 |
| HuggingFace（免费） | $0 | ⭐⭐⭐⭐ | ⭐ | VMess Argo |
| 廉价 VPS | $10-20/年 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 全协议 |

---

## 附录：本仓库相关文件

- `.github/workflows/weirdhost-renew.yml` — 自动续期 workflow
- `scripts/weirdhost_renew.py` — 续期脚本（基于 oyz8/weirdhost-login）
- `.github/workflows/build.yml` — Java jar 构建 workflow（与本节无关）
