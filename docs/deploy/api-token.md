# 创建 Cloudflare API Token

部署 Token 只负责让 GitHub Actions 管理 Worker、D1 与自定义域名。它不是 EdgeSSH 的登录凭据，也不应作为 Worker 运行时变量。

## 创建入口

1. 登录 Cloudflare 控制面板（Dashboard）。
2. 打开右上角个人资料。
3. 进入 **我的个人资料（My Profile）→ API 令牌（API Tokens）**。
4. 选择 **创建令牌（Create Token）**。
5. 以 **编辑 Cloudflare Workers（Edit Cloudflare Workers）** 模板作为起点。

Cloudflare 中文界面仍可能显示部分英文产品名或权限名，按括号中的英文原名定位即可。控制台的编辑（Edit）/读取（Read）对应 API 文档的 Write/Read。

## 权限建议

以模板创建后，按下表保留或补齐权限。范围、权限名和级别均同时列出中英文：

| 范围 | 权限 | 级别 | 何时需要 | 覆盖能力 |
| --- | --- | --- | --- | --- |
| 账户（Account） | Workers 脚本（Workers Scripts） | 编辑（Edit） | **始终需要** | 部署主/预览 Worker、Durable Object、变量和 Secret；管理 `workers.dev` 子域；绑定 Workers 自定义域名（Custom Domains） |
| 账户（Account） | D1（D1） | 编辑（Edit） | **始终需要** | 查找/创建 D1、检查旧数据、查询工作区状态和执行 migration |
| 账户（Account） | 账户设置（Account Settings） | 读取（Read） | 未配置 `CLOUDFLARE_ACCOUNT_ID` 时需要 | 通过 `/accounts` 自动发现唯一账户；显式配置账户 ID 后可省略 |
| 账户（Account） | Access：应用和策略（Access: Apps and Policies） | 编辑（Edit） | **仅 Cloudflare 登录模式**的首次启用、切回或配置修复 | 查找/创建 Access 应用，读取及更新邮箱策略 |
| 账户（Account） | Access：组织、身份提供程序和组（Access: Organizations, Identity Providers, and Groups） | 编辑（Edit） | **仅 Cloudflare 登录模式**的首次启用、切回或配置修复 | 读取 Zero Trust 团队域，查找身份提供程序，缺少时创建 OTP |

账户资源（Account Resources）只选择实际部署账户。GitHub 登录模式完全不调用 Access API，因此不需要两项 Access 权限。

EdgeSSH 的 `CUSTOM_DOMAIN` 使用账户级 **Workers 自定义域名（Workers Custom Domains）** API，由 **Workers 脚本（Workers Scripts）：编辑（Edit）** 覆盖；项目不使用普通 Workers 路由。因此，无论使用 `workers.dev` 还是 `CUSTOM_DOMAIN`，都不需要模板自带的 **区域（Zone）> Workers 路由（Workers Routes）：编辑（Edit）** 或 **区域（Zone）：读取（Read）**。只有自行把 `wrangler` 配置改成普通 route pattern 时，才需要把这两项 Zone 权限加回并限定到目标 Zone。

EdgeSSH 不使用 **Workers KV 存储（Workers KV Storage）** 或 **R2 存储（Workers R2 Storage）**。可以移除模板自带的 KV 权限，不要额外授予 R2。

<ScreenshotPlaceholder
  title="API Token 权限与资源范围"
  description="请截取权限表和 Account Resources 选择结果。只有自行使用普通 route pattern 时才展示 Zone 权限。必须遮盖 Token 值、账户邮箱和与教程无关的域名。"
  filename="02-cloudflare-api-token-permissions.png"
  src="/screenshots/02-cloudflare-api-token-permissions.png"
  alt="Cloudflare API Token 模板选择页面"
  caption="Cloudflare Token 模板入口示例；最终权限以本页最小权限表为准，可移除模板自带的 KV 和 Zone 权限。"
/>

## 保存到 GitHub

Token 创建完成后，只会完整显示一次：

1. 复制 Token。
2. 打开 Fork 的 **设置（Settings）→ 机密和变量（Secrets and variables）→ Actions**。
3. 在 **机密（Secrets）** 中创建 `CLOUDFLARE_API_TOKEN`。
4. 粘贴并保存。

不要把 Token 存为普通 Variable。也不要把 Token 写入 `.env`、`.dev.vars`、`wrangler.toml`、README 或截图。

GitHub 模式不需要任何 Access/IdP 权限。创建完成后回到[部署流程](/deploy/actions)，只填写所选模式的参数。自定义域名和 `workers.dev` 都由账户级 Workers 权限覆盖。

## 验证与排错

Action 报 `Authentication error` 或 `Invalid API Token` 时：

- 确认复制的是 API Token，不是 Global API Key。
- 确认 Secret 名称精确为 `CLOUDFLARE_API_TOKEN`。
- 确认 Token 尚未被撤销或过期。
- 确认 Account Resources 包含 `CLOUDFLARE_ACCOUNT_ID` 对应账户。

Action 能部署 Worker、但无法创建 D1 时，通常是缺少 **账户（Account）> D1（D1）：编辑（Edit）**。自定义域名绑定失败时，先确认域名所属 Zone 与 Worker 位于同一账户，并检查 **账户（Account）> Workers 脚本（Workers Scripts）：编辑（Edit）**；项目不依赖 Workers Routes 权限。

## 文档站 Pages 权限

维护者手工发布文档站时还需要 **账户（Account）> Cloudflare Pages（Cloudflare Pages）：编辑（Edit）**；普通 EdgeSSH 部署用户不需要此权限。文档源码单独维护，不在应用 `main`，也没有主分支 `Deploy docs` 工作流。
