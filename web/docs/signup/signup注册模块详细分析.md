# Signup 注册模块详细技术分析

## 1. 模块概述

### 1.1 功能定位
`signup` 模块是 Dify Web 应用的用户注册系统，采用基于邮箱验证的多步骤注册流程，确保用户邮箱的真实性和有效性。

### 1.2 注册流程
```
用户输入邮箱 → 发送验证码 → 验证码校验 → 设置密码 → 注册完成 → 跳转应用
     ↓              ↓              ↓            ↓           ↓
  page.tsx    check-code/    check-code/  set-password/   /apps
              page.tsx       page.tsx     page.tsx
```

### 1.3 技术栈
- **框架**: Next.js 14+ (App Router)
- **状态管理**: React Hooks + URL Query Parameters
- **UI**: Tailwind CSS + 自定义组件库
- **国际化**: react-i18next
- **HTTP**: React Query (TanStack Query)
- **图标**: Remix Icon

## 2. 文件结构与职责

```
web/app/signup/
├── layout.tsx                      # 注册流程统一布局
├── page.tsx                        # 步骤1: 邮箱输入页面
├── check-code/
│   └── page.tsx                    # 步骤2: 验证码校验页面
├── set-password/
│   └── page.tsx                    # 步骤3: 密码设置页面
└── components/
    ├── input-mail.tsx              # 邮箱输入表单组件
    └── input-mail.spec.tsx         # 邮箱组件单元测试
```

## 3. 核心组件详细分析

### 3.1 Layout 组件 (`layout.tsx`)

**职责**: 为所有注册页面提供统一的视觉框架和布局结构。

**关键特性**:
- 响应式设计 (移动端/桌面端自适应)
- 居中卡片布局
- 品牌化控制 (可通过 `systemFeatures.branding.enabled` 控制版权信息显示)

**依赖**:
```typescript
- useGlobalPublicStore: 获取系统配置
- useDocumentTitle: 设置页面标题
- Header: 复用 signin 模块的头部组件
```

**样式特点**:
- 背景: `bg-background-default-burn` (烧焦背景色)
- 卡片: 圆角 2xl, 边框高光效果
- 内边距: 移动端 6px, 桌面端 108px

---

### 3.2 步骤1: 邮箱输入 (`page.tsx` + `components/input-mail.tsx`)

#### 3.2.1 主页面组件 (`page.tsx`)

**核心逻辑**:
```typescript
const handleInputMailSubmitted = useCallback((email: string, result: string) => {
  const params = new URLSearchParams(searchParams)
  params.set('token', encodeURIComponent(result))
  params.set('email', encodeURIComponent(email))
  router.push(`/signup/check-code?${params.toString()}`)
}, [router, searchParams])
```

**数据流**:
1. 用户提交邮箱
2. `MailForm` 组件调用 API
3. API 返回临时 token
4. 通过回调函数传递 email 和 token
5. 路由跳转到验证码页面

#### 3.2.2 邮箱表单组件 (`input-mail.tsx`)

**Props 接口**:
```typescript
type Props = {
  onSuccess: (email: string, payload: string) => void
}
```

**状态管理**:
```typescript
const [email, setEmail] = useState('')
const { mutateAsync: submitMail, isPending } = useSendMail()
```

**验证规则**:
1. **非空校验**: 邮箱不能为空
2. **格式校验**: 使用 `emailRegex` 正则表达式验证
   ```typescript
   // 来自 @/config
   emailRegex.test(email)
   ```

**API 调用**:
```typescript
const res = await submitMail({ email, language: locale })
if ((res as MailSendResponse).result === 'success')
  onSuccess(email, (res as MailSendResponse).data)
```

**UI 特性**:
- 自动完成: `autoComplete="email"`
- Tab 索引: 优化键盘导航
- 禁用状态: 提交中或邮箱为空时禁用按钮
- 链接到登录: 已有账号可直接跳转 `/signin`
- 服务条款: 显示 TOS 和隐私政策链接 (可通过品牌配置控制)

---

### 3.3 步骤2: 验证码校验 (`check-code/page.tsx`)

**URL 参数**:
```typescript
const email = decodeURIComponent(searchParams.get('email') as string)
const token = decodeURIComponent(searchParams.get('token') as string)
```

**状态管理**:
```typescript
const [code, setVerifyCode] = useState('')
const [loading, setIsLoading] = useState(false)
const [token, setToken] = useState(initialToken)
```

#### 3.3.1 验证码校验逻辑

**验证流程**:
```typescript
const verify = async () => {
  // 1. 非空校验
  if (!code.trim()) {
    Toast.notify({ type: 'error', message: t('checkCode.emptyCode') })
    return
  }

  // 2. 格式校验 (6位数字)
  if (!/\d{6}/.test(code)) {
    Toast.notify({ type: 'error', message: t('checkCode.invalidCode') })
    return
  }

  // 3. API 验证
  setIsLoading(true)
  const res = await verifyCode({ email, code, token })

  // 4. 处理结果
  if ((res as MailValidityResponse).is_valid) {
    // 更新 token 并跳转
    const params = new URLSearchParams(searchParams)
    params.set('token', encodeURIComponent((res as MailValidityResponse).token))
    router.push(`/signup/set-password?${params.toString()}`)
  } else {
    Toast.notify({ type: 'error', message: t('checkCode.invalidCode') })
  }
}
```

#### 3.3.2 重发验证码逻辑

**重发流程**:
```typescript
const resendCode = async () => {
  const res = await submitMail({ email, language: locale })
  if ((res as MailSendResponse).result === 'success') {
    const newToken = (res as MailSendResponse)?.data
    // 更新本地 token 状态
    setToken(newToken)
    // 更新 URL 参数
    const params = new URLSearchParams(searchParams)
    params.set('token', encodeURIComponent(newToken))
    router.replace(`/signup/check-code?${params.toString()}`)
  }
}
```

**UI 组件**:
- **邮件图标**: `RiMailSendFill` 带阴影的圆角容器
- **倒计时组件**: `<Countdown onResend={resendCode} />` 控制重发间隔
- **返回按钮**: 使用 `router.back()` 返回上一步

---

### 3.4 步骤3: 设置密码 (`set-password/page.tsx`)

**URL 参数**:
```typescript
const token = decodeURIComponent(searchParams.get('token') || '')
```

**状态管理**:
```typescript
const [password, setPassword] = useState('')
const [confirmPassword, setConfirmPassword] = useState('')
const { mutateAsync: register, isPending } = useMailRegister()
```

#### 3.4.1 密码验证逻辑

**验证规则**:
```typescript
const valid = useCallback(() => {
  // 1. 非空校验
  if (!password.trim()) {
    showErrorMessage(t('error.passwordEmpty'))
    return false
  }

  // 2. 复杂度校验
  if (!validPassword.test(password)) {
    showErrorMessage(t('error.passwordInvalid'))
    return false
  }

  // 3. 一致性校验
  if (password !== confirmPassword) {
    showErrorMessage(t('account.notEqual'))
    return false
  }

  return true
}, [password, confirmPassword])
```

**密码复杂度要求** (来自 `@/config`):
```typescript
// validPassword 正则表达式通常要求:
// - 最小长度 8 位
// - 包含大小写字母、数字或特殊字符
```

#### 3.4.2 注册提交逻辑

**提交流程**:
```typescript
const handleSubmit = useCallback(async () => {
  if (!valid()) return

  // 1. 调用注册 API
  const res = await register({
    token,
    new_password: password,
    password_confirm: confirmPassword,
  })

  // 2. 处理成功响应
  if ((res as MailRegisterResponse).result === 'success') {
    // 3. 读取 UTM 参数
    const utmInfo = parseUtmInfo()

    // 4. 发送分析事件
    trackEvent(
      utmInfo ? 'user_registration_success_with_utm' : 'user_registration_success',
      { method: 'email', ...utmInfo }
    )
    sendGAEvent(
      utmInfo ? 'user_registration_success_with_utm' : 'user_registration_success',
      { method: 'email', ...utmInfo }
    )

    // 5. 清理 Cookie
    Cookies.remove('utm_info')

    // 6. 显示成功提示
    Toast.notify({ type: 'success', message: t('api.actionSuccess') })

    // 7. 跳转到应用主页
    router.replace('/apps')
  }
}, [password, token, valid, confirmPassword, register])
```

#### 3.4.3 UTM 参数处理

**UTM 信息解析**:
```typescript
const parseUtmInfo = () => {
  const utmInfoStr = Cookies.get('utm_info')
  if (!utmInfoStr) return null
  try {
    return JSON.parse(utmInfoStr)
  } catch (e) {
    console.error('Failed to parse utm_info cookie:', e)
    return null
  }
}
```

**用途**:
- 追踪用户来源渠道
- 分析营销活动效果
- 区分有 UTM 和无 UTM 的注册事件

---

## 4. API 接口规范

### 4.1 发送验证码 API

**Hook**: `useSendMail()`

**请求**:
```typescript
POST /register/email-code
{
  email: string,
  language: string  // 当前语言环境
}
```

**响应**:
```typescript
type MailSendResponse = {
  result: 'success' | 'fail',
  data: string  // 临时 token
}
```

**使用场景**:
- 首次发送验证码 (步骤1)
- 重新发送验证码 (步骤2)

---

### 4.2 验证码校验 API

**Hook**: `useMailValidity()`

**请求**:
```typescript
POST /register/email-code-validity
{
  email: string,
  code: string,    // 6位数字验证码
  token: string    // 上一步返回的临时 token
}
```

**响应**:
```typescript
type MailValidityResponse = {
  is_valid: boolean,
  token: string  // 新的 token (用于下一步)
}
```

**验证规则**:
- 验证码必须是 6 位数字
- 验证码有时效性 (通常 5-10 分钟)
- token 必须与发送时的 token 匹配

---

### 4.3 完成注册 API

**Hook**: `useMailRegister()`

**请求**:
```typescript
POST /register
{
  token: string,
  new_password: string,
  password_confirm: string
}
```

**响应**:
```typescript
type MailRegisterResponse = {
  result: 'success' | 'fail'
}
```

**成功后行为**:
- 自动登录 (后端设置 session/cookie)
- 跳转到 `/apps` 页面

---

## 5. 状态管理与数据流

### 5.1 跨页面状态传递

**传递方式**: URL Query Parameters

**传递的数据**:
```typescript
// 步骤 1 → 步骤 2
/signup/check-code?email=xxx&token=yyy

// 步骤 2 → 步骤 3
/signup/set-password?token=zzz&email=xxx
```

**优点**:
- 无需全局状态管理
- 支持页面刷新
- 支持浏览器前进/后退
- URL 可分享 (虽然 token 会过期)

**安全考虑**:
- Token 有时效性
- Token 只能使用一次或有限次数
- 敏感信息 (密码) 不通过 URL 传递

### 5.2 本地状态管理

**React Hooks**:
```typescript
// 表单输入状态
const [email, setEmail] = useState('')
const [code, setVerifyCode] = useState('')
const [password, setPassword] = useState('')
const [confirmPassword, setConfirmPassword] = useState('')

// 加载状态
const [loading, setIsLoading] = useState(false)
const { isPending } = useSendMail()

// Token 状态 (check-code 页面)
const [token, setToken] = useState(initialToken)
```

### 5.3 全局状态

**使用场景**: 系统配置

```typescript
const { systemFeatures } = useGlobalPublicStore()

// 使用示例
systemFeatures.branding.enabled  // 控制品牌信息显示
```

---

## 6. 错误处理

### 6.1 客户端验证错误

**处理方式**: Toast 提示

```typescript
Toast.notify({
  type: 'error',
  message: t('error.emailEmpty')
})
```

**错误类型**:
- 邮箱为空: `error.emailEmpty`
- 邮箱格式错误: `error.emailInValid`
- 验证码为空: `checkCode.emptyCode`
- 验证码格式错误: `checkCode.invalidCode`
- 密码为空: `error.passwordEmpty`
- 密码复杂度不足: `error.passwordInvalid`
- 两次密码不一致: `account.notEqual`

### 6.2 API 错误处理

**错误捕获**:
```typescript
try {
  const res = await verifyCode({ email, code, token })
  // 处理响应
} catch (error) {
  console.error(error)
  // React Query 会自动处理 HTTP 错误
}
```

**React Query 错误处理**:
- 自动重试机制
- 错误状态管理
- 全局错误处理器 (可在 QueryClient 配置)

### 6.3 边界情况

**处理的边界情况**:
1. **URL 参数缺失**: 使用 `decodeURIComponent(searchParams.get('xxx') || '')`
2. **UTM Cookie 解析失败**: try-catch 包裹 JSON.parse
3. **API 响应异常**: 类型断言 + 条件判断

---

## 7. 安全性分析

### 7.1 Token 机制

**Token 流转**:
```
发送验证码 → token1
验证验证码 → token2 (新 token)
完成注册   → 使用 token2
```

**安全特性**:
- Token 有时效性
- Token 在每个步骤更新
- Token 与邮箱绑定
- Token 不可预测 (后端生成)

### 7.2 输入验证

**前端验证**:
- 邮箱格式: 正则表达式
- 验证码格式: 6 位数字
- 密码复杂度: 正则表达式

**后端验证** (推测):
- 邮箱唯一性检查
- 验证码有效性检查
- Token 有效性检查
- 密码强度检查

### 7.3 XSS 防护

**React 内置防护**:
- 自动转义用户输入
- 使用 `dangerouslySetInnerHTML` 需谨慎

**当前代码**: 未发现 XSS 风险点

### 7.4 CSRF 防护

**Next.js 默认防护**:
- SameSite Cookie 策略
- 可能使用 CSRF Token (需查看后端实现)

---

## 8. 国际化 (i18n)

### 8.1 命名空间

**使用的命名空间**:
- `login`: 注册/登录相关文案
- `common`: 通用文案 (成功/错误提示)

### 8.2 翻译键

**注册模块使用的键**:
```typescript
// 步骤 1
t('signup.createAccount')
t('signup.welcome')
t('signup.verifyMail')
t('signup.haveAccount')
t('signup.signIn')

// 步骤 2
t('checkCode.checkYourEmail')
t('checkCode.tipsPrefix')
t('checkCode.validTime')
t('checkCode.verificationCode')
t('checkCode.verificationCodePlaceholder')
t('checkCode.verify')
t('checkCode.emptyCode')
t('checkCode.invalidCode')

// 步骤 3
t('changePassword')
t('changePasswordTip')
t('changePasswordBtn')
t('account.newPassword')
t('account.confirmPassword')

// 通用
t('email')
t('emailPlaceholder')
t('passwordPlaceholder')
t('confirmPasswordPlaceholder')
t('back')
t('tos')
t('pp')
t('tosDesc')
```

### 8.3 语言切换

**语言获取**:
```typescript
const locale = useLocale()  // 从 i18n context 获取
```

**API 调用时传递语言**:
```typescript
submitMail({ email, language: locale })
```

---

## 9. 分析与埋点

### 9.1 分析工具

**集成的工具**:
1. **Amplitude**: `trackEvent()`
2. **Google Analytics**: `sendGAEvent()`

### 9.2 埋点事件

**注册成功事件**:
```typescript
// 无 UTM 参数
trackEvent('user_registration_success', { method: 'email' })

// 有 UTM 参数
trackEvent('user_registration_success_with_utm', {
  method: 'email',
  ...utmInfo  // utm_source, utm_medium, utm_campaign 等
})
```

### 9.3 UTM 参数追踪

**Cookie 存储**:
- Cookie 名称: `utm_info`
- 存储格式: JSON 字符串
- 清理时机: 注册成功后

**UTM 参数示例**:
```json
{
  "utm_source": "google",
  "utm_medium": "cpc",
  "utm_campaign": "spring_sale",
  "utm_term": "ai_platform",
  "utm_content": "banner_ad"
}
```

---

## 10. 性能优化

### 10.1 代码分割

**Next.js App Router 自动分割**:
- 每个 `page.tsx` 自动成为独立的代码块
- 按需加载各个步骤的代码

### 10.2 React Query 优化

**特性**:
- 自动缓存
- 请求去重
- 后台重新验证
- 乐观更新 (如需要)

### 10.3 组件优化

**使用 useCallback**:
```typescript
const handleSubmit = useCallback(async () => {
  // ...
}, [dependencies])
```

**避免不必要的重渲染**:
- 合理使用 useCallback
- 避免内联函数作为 props

---

## 11. 测试策略

### 11.1 单元测试

**已有测试**:
- `input-mail.spec.tsx`: 邮箱输入组件测试

**建议测试覆盖**:
- 表单验证逻辑
- 错误处理
- 状态更新
- 路由跳转

### 11.2 集成测试

**建议测试场景**:
- 完整注册流程
- 验证码重发
- 错误恢复
- 浏览器前进/后退

### 11.3 E2E 测试

**关键路径**:
1. 输入邮箱 → 收到验证码
2. 输入验证码 → 验证通过
3. 设置密码 → 注册成功 → 跳转应用

---

## 12. 改进建议

### 12.1 安全性增强

1. **添加 Rate Limiting**: 限制验证码发送频率
2. **添加 Captcha**: 防止机器人注册
3. **密码强度指示器**: 实时显示密码强度

### 12.2 用户体验优化

1. **自动聚焦**: 页面加载后自动聚焦输入框
2. **Enter 键提交**: 支持键盘快捷操作
3. **进度指示器**: 显示当前步骤 (1/3, 2/3, 3/3)
4. **验证码自动填充**: 支持浏览器自动填充 OTP

### 12.3 错误处理改进

1. **更详细的错误信息**: 区分不同的 API 错误
2. **错误恢复指引**: 提供明确的下一步操作建议
3. **网络错误处理**: 离线状态提示

### 12.4 可访问性 (A11y)

1. **ARIA 标签**: 添加适当的 ARIA 属性
2. **键盘导航**: 确保所有功能可通过键盘操作
3. **屏幕阅读器支持**: 添加语义化标签

---

## 13. 依赖关系图

```
signup/
├── layout.tsx
│   ├── @/app/signin/_header
│   ├── @/context/global-public-context
│   └── @/hooks/use-document-title
│
├── page.tsx
│   └── components/input-mail.tsx
│       ├── @/app/components/base/button
│       ├── @/app/components/base/input
│       ├── @/app/components/base/toast
│       ├── @/app/signin/split
│       ├── @/config (emailRegex)
│       ├── @/context/global-public-context
│       ├── @/context/i18n
│       └── @/service/use-common (useSendMail)
│
├── check-code/page.tsx
│   ├── @/app/components/base/button
│   ├── @/app/components/base/input
│   ├── @/app/components/base/toast
│   ├── @/app/components/signin/countdown
│   ├── @/context/i18n
│   ├── @/service/use-common (useMailValidity, useSendMail)
│   └── @remixicon/react
│
└── set-password/page.tsx
    ├── @/app/components/base/amplitude
    ├── @/app/components/base/button
    ├── @/app/components/base/input
    ├── @/app/components/base/toast
    ├── @/config (validPassword)
    ├── @/service/use-common (useMailRegister)
    ├── @/utils/classnames
    ├── @/utils/gtag
    └── js-cookie
```

---

## 14. 总结

### 14.1 优点

1. **清晰的流程**: 三步注册流程简单明了
2. **良好的代码组织**: 文件结构清晰，职责分明
3. **完善的验证**: 前端验证 + 后端验证双重保障
4. **国际化支持**: 完整的 i18n 实现
5. **分析追踪**: 完善的埋点和 UTM 追踪
6. **响应式设计**: 适配移动端和桌面端

### 14.2 注意事项

1. **Token 安全**: Token 通过 URL 传递，需确保 HTTPS
2. **验证码时效**: 需要明确的过期提示
3. **错误处理**: 部分错误处理较简单，可以更详细
4. **测试覆盖**: 需要增加更多的单元测试和集成测试

### 14.3 技术亮点

1. **React Query**: 优雅的异步状态管理
2. **URL 状态管理**: 无需全局状态，支持页面刷新
3. **类型安全**: TypeScript 类型定义完善
4. **组件复用**: Header、Button、Input 等组件复用
5. **分析集成**: Amplitude + GA 双重追踪
