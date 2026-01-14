# Dify 登录模块分析

## 目录结构

```
app/signin/
├── page.tsx                    # 入口页面（路由到 NormalForm 或 OneMoreStep）
├── layout.tsx                  # 布局包装器
├── _header.tsx                 # 头部（Logo、语言选择、主题切换）
├── normal-form.tsx             # 主登录表单组件
├── one-more-step.tsx           # 登录后设置（邀请码、语言、时区）
├── split.tsx                   # 分隔线组件
├── page.module.css             # 样式文件
├── components/
│   ├── mail-and-code-auth.tsx      # 邮箱验证码登录
│   ├── mail-and-password-auth.tsx  # 邮箱密码登录
│   ├── social-auth.tsx             # OAuth（GitHub、Google）
│   └── sso-auth.tsx                # SSO（SAML、OIDC、OAuth2）
├── check-code/
│   └── page.tsx                # 验证码输入页面
├── invite-settings/
│   └── page.tsx                # 邀请接受设置页面
└── utils/
    └── post-login-redirect.ts  # 登录后重定向逻辑
```

---

## 核心组件功能

### NormalForm (`normal-form.tsx`)

主登录页面编排器，负责：
- 检查用户是否已登录 → 重定向到 `/apps`
- 验证系统功能和许可证状态
- 处理邀请链接（通过 `invite_token` 参数）
- 根据系统功能条件渲染不同的认证方式
- 显示许可证状态警告（LOST、EXPIRED、INACTIVE）
- 如果所有认证方式都禁用，显示"无登录方式"错误

状态管理：
- `authType`: 'code' | 'password'（切换邮箱验证码和密码认证）
- `showORLine`: 显示 OAuth 和邮箱认证之间的分隔线
- `allMethodsAreDisabled`: 如果没有启用任何认证方式则禁用所有方法
- `workspaceName`: 用于邀请链接

### MailAndPasswordAuth (`mail-and-password-auth.tsx`)

传统邮箱密码登录，功能包括：
- 邮箱验证（正则检查）
- 密码可见性切换（👀/😝 表情按钮）
- "忘记密码"链接（如果邮箱未设置则禁用）
- 发送前密码加密
- 支持邀请链接（禁用邮箱字段）
- 通过 Amplitude 跟踪登录事件

API 调用：`login()` 端点
响应：通过 `setWebAppAccessToken()` 设置访问令牌
重定向：
- 如果是邀请：`/signin/invite-settings`
- 否则：使用 `resolvePostLoginRedirect()` 或 `/apps`

### MailAndCodeAuth (`mail-and-code-auth.tsx`)

邮箱验证码登录流程：
1. 用户输入邮箱
2. 点击"验证邮箱" → 通过 `sendEMailLoginCode()` 发送验证码
3. 重定向到 `/signin/check-code`，带邮箱和令牌参数
4. 在 localStorage 中存储倒计时计时器

### CheckCode (`check-code/page.tsx`)

验证码输入页面，功能包括：
- 6位数字验证码输入（maxLength=6）
- 重新发送倒计时计时器
- 重新发送验证码功能
- 通过 Amplitude 跟踪登录事件

API 调用：`emailLoginWithCode()`，带加密验证码
重定向：与密码认证相同（invite-settings 或 /apps）

### SocialAuth (`social-auth.tsx`)

OAuth 登录按钮，支持的提供商：
- GitHub
- Google

实现：直接链接到后端 OAuth 端点
- `/oauth/login/github`
- `/oauth/login/google`

如果存在邀请令牌，会在查询参数中保留

### SSOAuth (`sso-auth.tsx`)

企业 SSO 登录，支持的协议：
- **SAML**：调用 `getUserSAMLSSOUrl()`
- **OIDC**：调用 `getUserOIDCSSOUrl()`，在 cookie 中存储状态
- **OAuth2**：调用 `getUserOAuth2SSOUrl()`，在 cookie 中存储状态

状态管理：在 cookie 中存储协议状态用于验证

### OneMoreStep (`one-more-step.tsx`)

新用户登录后设置，字段包括：
- 邀请码（可选）
- 界面语言选择
- 时区选择

状态：使用 `useReducer` 进行表单状态管理
API：调用 `useOneMoreStep()` hook 提交设置

### InviteSettings (`invite-settings/page.tsx`)

接受工作区邀请，功能包括：
- 通过 `useInvitationCheck()` 验证邀请令牌
- 用户设置姓名、语言、时区
- 调用 `activateMember()` 激活账户
- 令牌由后端存储在 cookie 中

---

## 登录流程实现

### 标准邮箱密码流程

```
NormalForm（检查登录状态）
  ↓
MailAndPasswordAuth（邮箱 + 密码）
  ↓
login() API 调用（加密密码）
  ↓
setWebAppAccessToken()（存储令牌）
  ↓
resolvePostLoginRedirect()（确定重定向）
  ↓
/apps 或自定义重定向 URL
```

### 邮箱验证码流程

```
NormalForm
  ↓
MailAndCodeAuth（邮箱输入）
  ↓
sendEMailLoginCode() API 调用
  ↓
重定向到 /signin/check-code（带邮箱和令牌）
  ↓
CheckCode（6位数字验证码输入）
  ↓
emailLoginWithCode() API 调用（加密验证码）
  ↓
/apps 或自定义重定向
```

### OAuth 流程

```
NormalForm
  ↓
SocialAuth（GitHub/Google 按钮）
  ↓
重定向到 /oauth/login/{provider}
  ↓
后端处理 OAuth 回调
  ↓
重定向回应用
```

### SSO 流程

```
NormalForm
  ↓
SSOAuth（SSO 按钮）
  ↓
获取 SSO URL（SAML/OIDC/OAuth2）
  ↓
在 cookie 中存储状态
  ↓
重定向到 SSO 提供商
  ↓
后端处理回调
  ↓
重定向到应用
```

### 邀请流程

```
带 invite_token 的邀请链接
  ↓
NormalForm（检测 invite_token）
  ↓
invitationCheck() 验证令牌
  ↓
显示工作区名称
  ↓
登录（任意方式）
  ↓
/signin/invite-settings
  ↓
InviteSettings（设置姓名、语言、时区）
  ↓
activateMember() API 调用
  ↓
/apps
```

---

## 状态管理与 API 调用

### 状态管理

- **React Hooks**：`useState`、`useReducer` 用于本地表单状态
- **自定义 Hooks**：
  - `useIsLogin()`：检查用户是否已登录
  - `useOneMoreStep()`：提交登录后设置
  - `useInvitationCheck()`：验证邀请令牌
  - `useLocale()`：获取当前语言环境
  - `useGlobalPublicStore()`：访问系统功能和品牌设置

### API 调用

来自 `@/service/common`：
- `login()`：邮箱密码认证
- `sendEMailLoginCode()`：发送验证码
- `emailLoginWithCode()`：验证码登录
- `invitationCheck()`：验证邀请令牌
- `activateMember()`：接受邀请并设置账户

来自 `@/service/sso`：
- `getUserSAMLSSOUrl()`：获取 SAML 登录 URL
- `getUserOIDCSSOUrl()`：获取 OIDC 登录 URL
- `getUserOAuth2SSOUrl()`：获取 OAuth2 登录 URL

### 令牌管理

- `setWebAppAccessToken()`：存储访问令牌
- 令牌存储在 cookie 中（后端管理）
- SSO 验证状态存储在 cookie 中

---

## 认证方式实现详情

### 邮箱密码

- 验证：邮箱正则 + 非空密码
- 安全：密码通过 `encryptPassword()` 加密
- 错误处理：显示无效凭据的特定错误
- 记住我：始终启用（`remember_me: true`）

### 邮箱验证码

- 验证：6位数字验证码
- 安全：验证码通过 `encryptVerificationCode()` 加密
- 倒计时：60秒计时器存储在 localStorage
- 重发：可在倒计时期间重新发送验证码

### OAuth（GitHub/Google）

- 实现：直接链接到后端端点
- 保留：查询参数中的邀请令牌
- 后端处理：OAuth 回调和令牌交换

### SSO（SAML/OIDC/OAuth2）

- 状态管理：状态存储在 cookie 中用于 CSRF 防护
- 协议检测：从系统功能确定协议
- 错误处理：显示无效协议的错误

### 邀请

- 令牌验证：显示设置前检查
- 设置：用户提供姓名、语言、时区
- 激活：后端在 cookie 中存储令牌

---

## 关键功能与行为

| 功能 | 实现 |
|------|------|
| 许可证检查 | 显示 LOST、EXPIRED、INACTIVE 许可证警告 |
| 品牌定制 | 支持自定义 Logo 和品牌设置 |
| 多语言 | 头部语言选择器，通过 `setLocaleOnClient()` 持久化 |
| 主题支持 | 头部主题选择器（动态导入） |
| 重定向逻辑 | `resolvePostLoginRedirect()` 检查 localStorage 中的待处理重定向 |
| 邀请链接 | 禁用邮箱字段，验证令牌，显示工作区名称 |
| 错误消息 | i18n 翻译，带特定错误代码 |
| 分析 | Amplitude 跟踪登录成功事件 |
| 可访问性 | 正确的标签、tabIndex、aria-disabled 属性 |

---

## 安全考虑

- **密码加密**：密码在传输前加密
- **验证码加密**：验证码在传输前加密
- **CSRF 防护**：SSO 状态存储在 cookie 中
- **令牌存储**：访问令牌存储在 cookie 中（建议 HttpOnly）
- **邮箱验证**：API 调用前进行正则验证
- **邀请验证**：显示设置前验证令牌
- **禁用字段**：邀请链接时禁用邮箱字段

---

## 文件路径汇总

| 文件 | 路径 |
|------|------|
| 主入口 | `app/signin/page.tsx` |
| 布局 | `app/signin/layout.tsx` |
| 核心组件 | `app/signin/normal-form.tsx` |
| 邮箱验证码认证 | `app/signin/components/mail-and-code-auth.tsx` |
| 邮箱密码认证 | `app/signin/components/mail-and-password-auth.tsx` |
| OAuth 认证 | `app/signin/components/social-auth.tsx` |
| SSO 认证 | `app/signin/components/sso-auth.tsx` |
| 验证码页面 | `app/signin/check-code/page.tsx` |
| 邀请设置 | `app/signin/invite-settings/page.tsx` |
| 重定向工具 | `app/signin/utils/post-login-redirect.ts` |
