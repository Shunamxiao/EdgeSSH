# 原生 GitHub OAuth 登录

`AUTH_PROVIDER=github` 表示 EdgeSSH 直接验证 GitHub 身份。应用仍部署在 Cloudflare Workers，但**不用开通 Zero Trust、不用创建 Access 应用或 GitHub IdP，也不用额外 GitHub PAT**。

每一步都附有检查点；部署任务变绿后仍须完成一次真实 GitHub 授权，才能确认 Client Secret、回调配置与管理员账号都正确。

::: warning 已部署用户点击登录就报“服务暂时不可用”？
2026-10-07 已修复标准 Base64 加密密钥被误当 Base64URL 解码的问题。含 `+` 或 `/` 的自动生成密钥会触发旧版登录 500，这不是你配置错误，也不是缺少 Access 权限。先看本页[已有用户修复](#已有用户修复)，**不要删除 Worker、D1 或更换 ENCRYPTION_KEY**。
:::

## 1. Fork 并确定正式入口

1. Fork [EdgeSSH](https://github.com/aozorae/EdgeSSH)，进入**自己的 Fork**。
2. 打开 **Actions**，按页面提示启用工作流。使用最新 **main**，不是独立文档分支。
3. 在下面两种入口中选择一种：

| 方案 | `CUSTOM_DOMAIN` 应填什么 | 正式入口 |
| --- | --- | --- |
| 自定义域名 | `ssh.example.com`，只填主机名 | `https://ssh.example.com` |
| 免费 workers.dev | 不创建或留空，不能填 workers.dev 地址 | 以 Deploy 摘要给出的地址为准 |

自定义域名必须由部署账户托管，建议用尚未绑定其他应用的子域名。不要将 `https://`、路径、通配符、示例占位域名或本站文档域名填入 `CUSTOM_DOMAIN`。Action 会绑定 Worker，不需要手动创建 Worker/D1。

如入口之前启用了 Cloudflare Access，先按[切换登录方式](/deploy/switch-login)解除旧网关；脚本不会删除你的安全策略。

**检查点**：你选定了唯一入口；自定义域名方案会关闭备用 workers.dev 入口，后续必须打开摘要指定的地址。

## 2. 在 GitHub 个人设置创建 OAuth App

点击 GitHub 头像，进入：

```text
Settings → Developer settings → OAuth Apps → New OAuth App
设置 → 开发者设置 → OAuth 应用 → 新建 OAuth 应用
```

这是**个人设置**，不是仓库设置。选择 **OAuth Apps**，不要选择 GitHub Apps 或 Personal access tokens。

| 字段 | 自定义域名示例 |
| --- | --- |
| Application name（应用名称） | `My EdgeSSH`，可自定 |
| Homepage URL（主页 URL） | `https://ssh.example.com` |
| Authorization callback URL（授权回调 URL） | `https://ssh.example.com/auth/callback` |

使用 workers.dev 且还不知道地址时，可以先用 `https://example.com` 和 `https://example.com/auth/callback` 占位。**部署后必须在第 5 步改成自己的真实地址，不能用占位地址登录。**

保存应用，复制 **Client ID**，再点击 **Generate a new client secret**。安全保存这次生成的 **Client Secret**；这两个值必须来自同一个 OAuth App。

**检查点**：已经获得 Client ID 和 Client Secret。它们不是 Cloudflare API Token，也不是你的 GitHub 密码。OAuth App 需要你在 GitHub 创建，工作流不会索要账号密码代建。

## 3. 在 Fork 的 Actions 分别保存 Variable 和 Secret

现在回到**自己的 Fork 仓库**，进入：

```text
Settings → Secrets and variables → Actions
设置 → 机密和变量 → Actions
```

在 **Variables** 页签点 **New repository variable**；在 **Secrets** 页签点 **New repository secret**。不要建到未被工作流引用的 Environment 配置中。

| 页签 | Name（原样复制） | Value / Secret |
| --- | --- | --- |
| Variables | `AUTH_PROVIDER` | `github`，小写 |
| Variables | `GH_CLIENT_ID` | 第 2 步的 Client ID |
| Variables | `GH_ADMIN` | 唯一允许登录的个人 GitHub 用户名 |
| Variables | `CUSTOM_DOMAIN` | 仅自定义域名方案填写，如 `ssh.example.com` |
| Secrets | `GH_CLIENT_SECRET` | 第 2 步同一应用的 Client Secret |
| Secrets | `CLOUDFLARE_API_TOKEN` | 按 [API Token 最小权限](/deploy/api-token)创建的部署 Token |

例如管理员主页是 `https://github.com/example-user`，`GH_ADMIN` 就填 `example-user`，不是完整 URL、邮箱、组织名或显示昵称。值不加引号、尖括号或注释。

普通新部署**不需要填写** `GH_ADMIN_ID`、`ADMIN_EMAIL`、Access 参数或 `ENCRYPTION_KEY`。加密密钥由脚本首次安全生成，以后保留；不要照着示例自己填一个短字符串。

Token 只选择实际部署账户。默认需要 Account Settings Read 来自动发现唯一账户；也可以在 Variables 设置 32 位 `CLOUDFLARE_ACCOUNT_ID` 后省略该读取权限。Account ID 不是 Zone ID。

::: warning 易错位置
- `GH_CLIENT_SECRET` 与 `CLOUDFLARE_API_TOKEN` 必须是 Secret，不能放 Variable。
- `AUTH_PROVIDER`、`GH_CLIENT_ID`、`GH_ADMIN` 必须是 Variable；工作流不会从 Secret 读取它们。
- 旧版 `CUSTOM_DOMAIN` Secret 优先于同名 Variable。改域名前检查两边，不要留下旧值覆盖新值。
- 不要在 Cloudflare 手工创建同名运行时覆盖配置；普通配置入口是 Fork 的 Actions 设置。
:::

从旧名称升级时，把 `GITHUB_CLIENT_ID`、`GITHUB_ADMIN`、`GITHUB_ADMIN_ID` 改为对应的 `GH_*`，并重新保存 `GH_CLIENT_SECRET` Secret。`GITHUB_` 是 GitHub 保留前缀。Secret 不能读回明文；不要为变量迁移重建 OAuth App 或更换资料加密密钥。

**检查点**：两个 Secret 在 Secrets 页签；三个必需非敏感变量在 Variables 页签。只有 `GH_ADMIN` 对应管理员可以登录，不是任何 GitHub 用户都可以。

## 4. 运行最新 main 的 Deploy

进入 **Actions → Deploy → Run workflow**，选择 **main**，管理员邮箱框留空，然后运行。

等待**整次任务**变绿，不是只看某个“上传成功”步骤。脚本会准备 Worker/D1、执行迁移、固定管理员数字 ID，首次生成密钥，后续沿用原密钥与资料。

新版在宣布成功前实际检查：

| 请求 | 期望结果 | 覆盖内容 |
| --- | --- | --- |
| `/auth/login` | 302 到正确 GitHub 授权地址，并签发临时 Cookie | 正式入口、OAuth 配置与真实密钥签名 |
| 不携带 Cookie 的 `/api/auth/me` | 401 | D1 工作区可读取，认证方式匹配，未开放匿名访问 |

入口验收失败会让 Deploy 失败。日志应出现 **“GitHub 登录入口签名与 D1 工作区验收通过”**，摘要应给出 `github`、正式入口及回调地址。

::: tip 401 在这里是成功条件
未登录请求必须被拒绝；这不表示 Token 权限有问题。自动检查不跟随 GitHub 授权，也不能验证 Client Secret 是否被撤销，仍须做第 5 步。`/api/health` 正常或首页能打开不足以证明登录可用。
:::

## 5. 回填回调地址并真实登录

1. 回到个人 **Settings → Developer settings → OAuth Apps → 你的应用**。
2. 将 Deploy 摘要的正式入口复制到 **Homepage URL**，将完整回调地址复制到 **Authorization callback URL**，保存。必须是 `https://正式入口/auth/callback`，不要多加末尾斜杠。
3. 打开摘要中的正式入口，点击 **登录**。在 GitHub 使用 `GH_ADMIN` 指定的账号授权；浏览器若登录了其他账号，先切换。
4. 成功后回到主机总览。同一浏览器访问 `/api/auth/me` 应返回 **200** 和 `provider: "github"`。
5. 再用自己授权的 SSH 主机验证终端、文件等功能。无需提供 SSH 主机即可完成 GitHub 登录验收。

每次失败都回首页重新点击登录，不能刷新或收藏带 `code`、`state` 的旧 callback 链接。占位 URL 必须在此时被替换。

登录仅读取公开身份，不申请仓库、组织或私人邮箱权限。首次部署将用户名解析为数字 ID 并固定到 D1，改名不会更换管理员；仅修改 `GH_ADMIN` 不会转让已有实例。确需更换管理员时设置新的 `GH_ADMIN_ID` 并部署，旧会话会被撤销。

## 已有用户修复

2026-10-07 修复前，登录签名把标准 Base64 `ENCRYPTION_KEY` 当作 Base64URL 读取。随机密钥含 `+` 或 `/` 时，点击登录会抛异常并返回通用 500；原固定测试密钥不含这些字符，因此没发现问题。

1. 在 Fork 执行 **Sync fork → Update branch**，确认包含修复提交 **`e0eaa36`**（`fix(auth): decode GitHub signing keys as standard Base64`）或其后续版本。Fork 有自定义改动时先处理合并，不要盲目覆盖。
2. 保留原 Worker、D1、OAuth App、Client Secret 和 **ENCRYPTION_KEY**。不要清库、删除资源或反复换密钥。
3. 对更新后的 **main** 运行 **Deploy**，通过第 4 步的入口验收。重新运行旧提交的任务不会获得新代码。
4. 从正式首页重新登录，不再使用旧 callback 链接。

新代码使用原密钥字节，既有资料无需迁移或重新加密。**GitHub 模式不需要因此新增 Access 权限。** 若更新后仍失败，按下表判断；同一句 500 也可能来自其他异常。

## 按失败阶段排查

| 阶段/提示 | 检查与处理 |
| --- | --- |
| Deploy 配置校验说缺少变量 | 检查名称、Variables/Secrets 页签，以及是否误建在 Environment 中 |
| Deploy 的 Cloudflare API 返回 403 | 检查 Token 权限、实际账户范围、有效期；不是更换 ENCRYPTION_KEY |
| Deploy 入口验收无法连接 | 检查正式域名 DNS/证书和所属账户，待就绪后重跑，不跳过验收 |
| 点击登录、跳 GitHub 前返回 500 | 先确认部署的是含 Base64 修复的 main，保留原密钥 |
| “请从配置的正式入口登录” | 使用 Deploy 摘要入口，检查旧 CUSTOM_DOMAIN Secret 是否覆盖 Variable，不手改 APP_ORIGIN |
| GitHub 提示回调地址错误 | 回填完整 `https://正式入口/auth/callback`，不要只填首页 |
| 回调提示授权码无效/过期 | Client ID/Secret 要来自同一 OAuth App，Secret 必须未撤销；从首页重新授权 |
| 回调 403“不是管理员” | 切换到固定管理员账号；这不是 Cloudflare API 权限问题 |
| 授权返回后仍为 500/502 | 检查失败路径是否 `/auth/callback`、GitHub 连通性、OAuth 配置和 D1 状态；按下面方式提供脱敏反馈 |
| 仍出现 Cloudflare Access 页面 | 解除旧域名网关，见[切换登录方式](/deploy/switch-login) |

反馈只需提供：失败步骤、正式入口（如可公开）、失败请求的**路径与 HTTP 状态码**、Deploy 的 SHA/运行链接和脱敏截图。**不要发送** API Token、Client Secret、ENCRYPTION_KEY、Cookie、callback 查询参数或未经脱敏的整份网络日志。

第三方服务故障、账户限制与 DNS 尚未生效时，需要先解决对应问题；自动检查不能替代真实授权验收，也不能保证外部服务永远可用。

## 会话与退出

- state 与当前浏览器绑定，PKCE 使用 S256，临时 Cookie 有效期 10 分钟。
- 正式 Cookie 与签名令牌有效期均为 **30 天**，使用 Secure、HttpOnly、SameSite=Lax。
- 退出会撤销本实例已有管理员会话并清除本浏览器 Cookie，不退出 GitHub 网站账号。
- GitHub access token 只用于当前身份验证，不写数据库、Cookie 或日志。
- 更新 OAuth Client Secret 后，在 Actions 更新 `GH_CLIENT_SECRET` 并重跑 Deploy；不要更换 `ENCRYPTION_KEY`。
