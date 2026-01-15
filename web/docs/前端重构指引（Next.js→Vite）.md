# 前端重构指引（Next.js→Vite）

本文档面向“将 `dify/web` 前端从 Next.js（App Router）重构为纯前后端分离的 SPA（Vite + React + TypeScript）”这一目标，给出一份可落地的整体架构方案与分阶段迁移步骤。

重构范围以 `web/` 当前业务为准，包含 Console（控制台）与 WebApp（分享/嵌入式）两类形态；后端保持现有 API（`/console/api`、`/api`、`/marketplace`）不变。

---

## 1. 目标与原则

### 1.1 目标

- 移除 Next.js：不再依赖 App Router、RSC、Next middleware、Next build/standalone。
- 前后端完全分离：前端构建产物为纯静态资源（HTML/CSS/JS），可由 Nginx/CDN 托管。
- 保留现有业务能力：登录与鉴权、Console/WebApp 两套路由与布局、i18n、请求封装、React Query 缓存、主题、埋点/监控等。

### 1.2 非目标（建议明确）

- 不做后端 API 结构大改（除必要的 CORS/Cookie/CSRF 配置）。
- 不追求 SSR/SEO（Console 通常不需要；WebApp 若需要 SEO 另行评估）。

### 1.3 迁移策略选型

- **策略 A：双轨迁移（推荐）**：新建 Vite SPA，与现有 Next.js 并行运行；逐模块迁移与灰度，最终切换入口并下线 Next。
- **策略 B：一次性替换**：在一个迭代内完成全量替换。适合业务冻结期，但风险高、回滚成本大。

本文默认以“策略 A：双轨迁移”为主线设计步骤，便于控风险与回滚。

---

## 2. 现状关键依赖点（需要被替换/承接）

### 2.1 路由与布局（Next App Router）

- `app/` 目录承载全部路由与布局（Layout/Route Group/动态段）。
- 广泛使用 `next/navigation`（`useRouter/useParams/usePathname/useSearchParams`）与 `next/link`。
- 路由组示例：
  - `app/(commonLayout)`：Console 形态布局
  - `app/(shareLayout)`：WebApp 形态布局

### 2.2 运行时配置注入（env + body dataset）

当前根布局通过给 `<body>` 注入 dataset（`DatasetAttr`）实现“运行时可变配置”（见 [layout.tsx](file:///c%3A/work/dify/web/app/layout.tsx#L36-L129) 与 [config/index.ts](file:///c%3A/work/dify/web/config/index.ts#L56-L90)）。

迁移到静态托管后，需要选择“配置注入方案”（见第 4 章）。

### 2.3 安全头与嵌入策略（Next middleware）

- CSP、`X-Frame-Options` 由 [middleware.ts](file:///c%3A/work/dify/web/middleware.ts) 注入。
- SPA 后需要将该能力迁移到网关/Nginx/CDN 层（见第 9.5 节）。

### 2.4 i18n（Server/Client 分层）

- 目前存在 server/client 两套入口（见 [i18n-config](file:///c%3A/work/dify/web/i18n-config)）。
- SPA 仅保留 client 侧加载与语言切换逻辑，去掉 server 侧读取 locale 的流程（见第 6.3 节）。

### 2.5 React Query（含 Server 端 QueryClient）

- Next 下存在 server/client 两套 QueryClient 初始化（见 `context/query-client*.ts`）。
- SPA 只保留 client QueryClient（server 侧逻辑不再需要）。

### 2.6 Next 构建能力（rewrites/redirects/images/mdx/basePath）

来自 [next.config.js](file:///c%3A/work/dify/web/next.config.js)：

- 开发代理 rewrites（`REMOTE_API_URL`）
- `/ -> /apps` redirect
- `images.remotePatterns`（Next Image）
- `basePath`
- MD/MDX pageExtensions

这些能力需要在 Vite 与部署侧分别承接（见第 7、9 章）。

---

## 3. 目标架构（Vite + React + TS）

### 3.1 推荐技术栈（与现有保持一致/低改动优先）

- 构建：Vite + TypeScript
- 路由：React Router v6（或 TanStack Router；推荐 React Router 便于团队接手）
- 请求层：沿用现有 `service/`（ky + fetch hooks + base.ts 拦截）
- 服务端状态：TanStack Query（沿用）
- 全局状态：保留现有 Zustand/Jotai（逐步收敛到一种也可，但属于二期优化）
- i18n：react-i18next（沿用）
- 样式：Tailwind CSS（沿用配置）
- UI 组件体系：shadcn/ui（基于 Radix UI）+ 少量领域自定义组件
- 测试：Vitest + Testing Library（沿用）
- 监控：Sentry（沿用）；埋点：Amplitude（沿用）

### 3.2 目录结构建议

如果计划引入 shadcn/ui 且希望治理组件数量、并逐步走向多应用拆分，推荐直接采用 pnpm workspace（monorepo）承接新工程，避免把组件/请求层/i18n 等公共能力在多个工程中复制粘贴。

示例（仅为建议，可按团队习惯调整）：

```
web/
  apps/
    console/                 # Vite SPA：Console（控制台）
      index.html
      vite.config.ts
      src/
        main.tsx
        app.tsx
        routes/
        layouts/
        pages/
        styles/
    webapp/                  # Vite SPA：WebApp（可选；也可先单 SPA）
      index.html
      vite.config.ts
      src/...
  packages/
    ui/                      # shadcn/ui 组件与基础 UI（按钮、弹窗、表单等）
      src/
        components/
        hooks/
        styles/
    service/                 # API 封装（沿用并收敛 web/service）
      src/...
    shared/                  # 通用工具、hooks、常量、types（跨应用）
      src/...
    i18n/                    # i18n 初始化与资源加载封装
      src/...
    config/                  # 运行时配置读取与导出（API_PREFIX 等）
      src/...
    eslint-config/           # 统一 ESLint 规则（可选）
    tsconfig/                # 统一 TS 配置（可选）
```

迁移期可让 Next.js 与 `apps/console` 并行部署在不同路径或不同域名，逐步切流，最终下线 Next。

### 3.3 应用拆分：单 SPA vs 双 SPA

你需要在“Console + WebApp”的承载方式上做选择：

- **单 SPA**：一套路由树，`/apps`、`/signin`、`/chat/...`、`/webapp-signin` 等全部在同一应用中，以不同 Layout 区分。
- **双 SPA**：Console 与 WebApp 独立构建与发布（两个入口 HTML），隔离依赖体积与权限边界；但部署更复杂。

推荐先做“单 SPA”，待稳定后再评估是否拆成“双 SPA”。

### 3.4 组件体系（shadcn/ui + 自定义组件治理）

如果你认为当前 `components`（尤其是 `app/components`）自定义组件过多，建议把“组件治理”作为重构的一等目标，而不是迁移结束后的二期工作，否则迁移会把历史包袱原样复制到新工程。

建议将组件分为三层，并把落盘位置与依赖边界固定下来：

- **基础 UI（UI primitives）**
  - 目标：统一交互、样式、可访问性；减少“重复的按钮/弹窗/表单组件”。
  - 承载：`packages/ui`
  - 选型：shadcn/ui（Radix UI）为基础，配合 Tailwind 主题变量统一风格。
- **可复用业务组件（domain components）**
  - 目标：和业务域强相关，但会跨页面复用（例如 AppCard、模型选择器、数据集选择器等）。
  - 承载：优先放在各 app 内（`apps/console/src/components`），稳定后再按业务域抽到 `packages/*`。
- **页面组件（page components）**
  - 目标：只服务单页或单流程，不跨域复用。
  - 承载：`apps/*/src/pages/**/components`

建议的迁移策略：

- 先把“基础 UI”收敛到 shadcn/ui：按钮/输入框/下拉/弹窗/抽屉/Popover/Tooltip/Tabs 等，统一交互与样式手感。
- 对现有自定义组件做一次分类清点：
  - 只是 shadcn 组件的薄封装：迁移到 `packages/ui` 并统一 API
  - 领域组件：留在 app 内，迁移时只做必要的 Next 替换
  - 页面组件：不要提升为共享组件，避免共享导致耦合
- 在迁移期设置一条规则：新增基础 UI 一律走 `packages/ui`，不再在 app 内“新造轮子”。

---

## 4. 配置与环境变量（前后端分离的关键）

### 4.1 现状问题

当前大量配置来自 `process.env.NEXT_PUBLIC_*`，并通过 `<body data-*>` 做运行时兜底（见 [config/index.ts](file:///c%3A/work/dify/web/config/index.ts#L56-L90)）。

在静态托管 SPA 中：

- 构建时环境变量（`.env`）只能在构建期注入，构建后不可变。
- 运行时可变配置需要额外方案（比如 `config.json` 或 HTML 注入）。

### 4.2 推荐方案（优先级从高到低）

1. **运行时配置文件（推荐）**
   - 部署时提供 `public/config.json`（或 `/config.json`），前端启动时拉取并写入全局配置。
   - 好处：同一份构建产物可部署到不同环境；符合“前后端完全分离”。
2. **HTML dataset 注入（兼容现有模式）**
   - 在 `index.html` 的 `<body>` 上注入 `data-api-prefix` 等字段，继续复用现有 `getStringConfig/getBooleanConfig` 逻辑。
   - 好处：改动小；坏处：需要部署侧模板化 HTML。
3. **纯 Vite 构建期注入**
   - 使用 `VITE_*`，通过 `import.meta.env` 读取。
   - 好处：最简单；坏处：环境切换必须重新构建，不利于多环境交付。

### 4.3 环境变量命名建议

建议不要在业务代码里直接散落 `import.meta.env`，而是统一在一处做兼容层，例如：

- `config/runtime.ts`：负责“运行时配置文件/HTML dataset/构建期 env”的统一读取
- 业务侧仍通过 `config/index.ts` 导出常量

迁移期可以先保持 `NEXT_PUBLIC_*` 名称不变，内部实现改为读取 runtime config（这样对业务侵入最小）。

---

## 5. 鉴权与安全（前后端分离最大风险点）

### 5.1 Console（Cookie + CSRF）

现状（见 [Dify Web 前端登录与鉴权.md](file:///c%3A/work/dify/web/docs/Dify%20Web%20%E5%89%8D%E7%AB%AF%E7%99%BB%E5%BD%95%E4%B8%8E%E9%89%B4%E6%9D%83.md) 与 [service/fetch.ts](file:///c%3A/work/dify/web/service/fetch.ts)）：

- Console API 默认依赖 Cookie，会开启 `credentials: 'include'`
- CSRF Header 从 Cookie 读取并注入（`X-CSRF-Token`）
- 401 时触发刷新令牌并重试，失败跳转 `/signin`（见 [service/base.ts](file:///c%3A/work/dify/web/service/base.ts)）

前后端分离后需要满足：

- **CORS 允许携带 Cookie**
  - `Access-Control-Allow-Origin` 不能是 `*`
  - 必须返回 `Access-Control-Allow-Credentials: true`
- **Cookie SameSite/Domain/Secure 设置正确**
  - 前端与后端不同站点时，Cookie 通常需要 `SameSite=None; Secure`
  - `NEXT_PUBLIC_COOKIE_DOMAIN` 的策略需要重新验证（见 [config/index.ts](file:///c%3A/work/dify/web/config/index.ts#L163-L180)）
- **CSRF Cookie 可读性**
  - 如果 CSRF token 存在 `HttpOnly`，前端无法读取，就需要后端改为通过响应头/接口返回 token

建议的部署拓扑（可二选一）：

- **同域反向代理（推荐）**：`https://console.example.com` 上同时代理静态资源与 `/console/api`，尽量规避 CORS 与 SameSite 问题。
- **跨域直连**：`https://web.example.com` 调 `https://api.example.com`，需要完整的 CORS + Cookie 策略配合。

### 5.2 WebApp（AccessToken/ShareCode/Passport）

WebApp 主要依赖本地 token + 分享码/Passport 头（见 [service/webapp-auth.ts](file:///c%3A/work/dify/web/service/webapp-auth.ts)）。

迁移时重点：

- 保持请求头注入逻辑不变
- 保持 WebApp 的“未授权/不可用”页面与跳转逻辑不变（现有在 `app/(shareLayout)`）

### 5.3 路由守卫模型（SPA）

Next 下常见“守卫”由请求层（401/403）与 Layout/Splash 组件共同完成。

SPA 推荐组合：

- **路由级守卫**：React Router 的 layout route 中做登录检查（例如读取 `useIsLogin` + 显示 Splash）
- **请求级兜底**：保留 `service/base.ts` 的 401/403 处理（刷新/登出/跳转）

---

## 6. 迁移路线图（分阶段步骤）

### 阶段 0：准备与边界锁定（1～3 天）

- 输出“页面与路由清单”：以 `app/` 为基准，梳理所有路由（含动态段）与对应 Layout。
- 标注“必须先迁移”的链路：
  - 登录/注册（`/signin`、`/signup/...`）
  - Console 主框架（`/apps`、应用详情/工作流等）
  - WebApp 入口（`/chat`、`/webapp-signin`）
- 确定部署拓扑（同域代理或跨域直连）与配置注入方案（第 4 章）。

交付物：一份路由映射表（可写在本文档附录，或单独维护表格）。

### 阶段 1：搭建 pnpm workspace + Vite 骨架（1～3 天）

在 `web/` 下落地 pnpm workspace（monorepo）结构，并完成以下能力：

- TS + React 入口（`src/main.tsx`）
- Tailwind（迁移 [tailwind.config.js](file:///c%3A/work/dify/web/tailwind.config.js) 的 content 路径）
- ESLint/TypeCheck/Test 与现有保持一致（pnpm scripts 可复用）
- 路由（React Router）与基础 Layout 框架（CommonLayout/ShareLayout）
- `packages/ui` 建立组件基线（引入 shadcn/ui 并确定主题/样式变量策略）

### 阶段 2：打通“配置 + 请求层 + 登录态”（2～5 天）

目标：能在 SPA 中完成真实登录，并在 Console 页面加载用户态与列表数据。

- 配置读取：实现第 4 章方案之一，导出 `API_PREFIX/PUBLIC_API_PREFIX/...`
- 复用请求层：
  - 优先直接复用 `web/service/`（拷贝或软链接到 `spa/src/service`，迁移期建议“复制一份”避免互相干扰）
  - 确保 `credentials/include`、CSRF 注入、401 刷新逻辑可用
- 复用 React Query：
  - 用 client 侧 QueryClient 初始化替代 Next 的 server/client 分离
  - 将 QueryProvider 置于 `main.tsx` 顶层

验收：`/signin` 登录成功 -> 跳转 `/apps` -> 列表接口返回并渲染。

### 阶段 3：迁移全局 Providers（1～3 天）

对照 [layout.tsx](file:///c%3A/work/dify/web/app/layout.tsx#L80-L127)，在 SPA 入口组装等价 Provider：

- Theme（替换/沿用 `next-themes` 的能力）
- JotaiProvider（若仍在用）
- React Query Provider
- i18n Provider（仅保留 client 侧）
- Toast/Modal/Global stores
- Sentry/Amplitude 初始化

验收：主题切换、语言切换、Toast 弹出、Sentry 上报均正常。

### 阶段 4：迁移路由树与布局（持续迭代）

将 Next 路由模型映射到 React Router 的 nested routes：

- Next `layout.tsx` -> React Router 的 `element: <Layout/>` + `Outlet`
- Next 路由组 `(commonLayout)/(shareLayout)` -> 两套顶层 Layout route
- Next 动态段 `[id]` -> `:id`
- Next `redirects()`（`/ -> /apps`）-> SPA 中用 Navigate 实现

建议先落地这两个顶层路由骨架：

- Console：`/signin`、`/signup/*`、`/apps`、`/app/:appId/*`
- WebApp：`/webapp-signin`、`/chat/:token?`、`/completion/*`、`/workflow/*`（按现有规则）

### 阶段 5：按业务域迁移页面（2～6 周，取决于规模）

建议顺序：

1. 登录/注册（依赖最少，且是全站入口）
2. Apps 列表/创建/概览（业务主链路）
3. Workflow/Chat（复杂度高、交互密集）
4. Datasets/Plugins/Settings 等后台模块
5. WebApp（share/iframe/扫码等入口）

每迁移一个模块，要求：

- 页面路由可达、数据可拉取、核心交互可用
- 单测（若原模块已有）能跑通或完成等价覆盖

### 阶段 6：替换 Next 专属能力（穿插进行）

见第 7 章的“替换清单”，逐项清理 `next/*` 依赖，保证 SPA 纯净。

### 阶段 7：部署切换与下线 Next（1～3 天）

- 在网关上将 `/{basePath?}` 指向 SPA 静态资源
- 配置 history fallback（`/apps` 刷新不 404）
- 验证 CSP/X-Frame-Options、cookie、跨域、缓存策略
- 保留一段时间的 Next 回滚路径（例如 `/_next_app` 前缀或独立域名）

---

## 7. Next.js 能力替换清单（对照表）

| 现有能力 | Next.js 实现 | SPA 推荐替代 |
| :--- | :--- | :--- |
| 路由导航 | `next/navigation` | `react-router-dom`：`useNavigate/useParams/useLocation/useSearchParams` |
| 链接组件 | `next/link` | `react-router-dom`：`<Link/>` |
| 图片优化 | `next/image` + `images.remotePatterns` | 普通 `<img/>` 或引入图片组件；远程域白名单转移到 CSP/代理层 |
| Layout/路由组 | `app/(group)/layout.tsx` | React Router nested routes + Layout 组件 |
| 重定向 | `redirects()` | 路由 `<Navigate/>` 或在入口做一次性跳转 |
| 开发代理 | `rewrites()` | `vite.config.ts` 的 `server.proxy` |
| CSP/XFO | `middleware.ts` | Nginx/CDN/网关注入响应头 |
| basePath | `next.config.basePath` | `vite base` + Router basename（或部署侧统一前缀） |
| MDX 页面 | `@next/mdx` + pageExtensions | Vite MDX 插件（如 `@mdx-js/rollup`）或将文档从运行时移除 |
| next/font | `next/font/google` | 直接使用 CSS `@import` / 自托管字体文件 |
| Service Worker | Serwist + Next | `vite-plugin-pwa`（或自维护 sw.js） |
| nuqs adapter | `nuqs/adapters/next/app` | 切换为 nuqs 的非 Next adapter，或用 Router 的 `useSearchParams` |

---

## 8. 工程化与质量保障

### 8.1 TypeScript 与路径别名

当前 `tsconfig.json` 使用 `@/* -> ./*`（见 [tsconfig.json](file:///c%3A/work/dify/web/tsconfig.json#L13-L20)）。

SPA 建议保持相同别名，避免大规模改 import 路径；在 Vite 侧用 tsconfig paths 插件或手动配置 alias。

在 monorepo 下，建议：

- app 内使用 `@/` 指向各自的 `src`
- 共享包用 `@dify/*`（例如 `@dify/ui`、`@dify/service`、`@dify/shared`、`@dify/i18n`、`@dify/config`）

### 8.2 Lint / Type Check / Test

参考现有前端流程（见 [AGENTS.md](file:///c%3A/work/dify/AGENTS.md)）：

- `pnpm lint:fix`
- `pnpm type-check:tsgo`
- `pnpm test`

迁移期建议保持脚本一致，减少 CI/团队习惯迁移成本。

### 8.3 回归策略

- 单元测试：优先保障“service 层与核心组件”
- E2E（建议引入）：登录 -> 进入 apps -> 创建/打开 app -> 进入 workflow/chat 的主链路
- 关键指标：首屏、路由切换、长列表性能、编辑器/画布类页面稳定性

### 8.4 组件治理与依赖策略

引入 shadcn/ui 后，建议明确依赖边界，避免组件层持续膨胀：

- `packages/ui` 不依赖 `packages/service`（UI 不直连请求），最多依赖 `packages/shared` 的纯工具与类型
- `apps/*` 可以依赖 `packages/*`，但不要反向依赖
- 领域组件如果抽到 `packages/*`，优先按业务域拆包（例如 `packages/billing`、`packages/datasets`），而不是把所有业务组件塞进 `ui`

---

## 9. 部署与运行（静态托管 + API）

### 9.1 静态资源托管

建议产物：

- `dist/`（Vite build 输出）
- 通过 Nginx/CDN 托管

必须配置：

- history fallback：所有非静态资源请求返回 `index.html`
- 缓存策略：`index.html` 短缓存，静态资源（hash 文件）长缓存

### 9.2 API 访问与代理

推荐拓扑：同域反代（最大限度避免 Cookie/CORS/CSRF 问题）：

- `https://console.example.com/` -> 静态资源
- `https://console.example.com/console/api` -> 后端
- `https://console.example.com/api` -> 后端（WebApp）

开发态：用 Vite proxy 对齐 Next rewrites（参考 [next.config.js](file:///c%3A/work/dify/web/next.config.js#L77-L93) 的能力）。

### 9.3 basePath（可选）

如果历史上依赖 `NEXT_PUBLIC_BASE_PATH`，SPA 需要：

- 构建：Vite `base`
- 路由：Router `basename`
- 静态资源引用：全部走相对路径或由 `base` 统一处理

### 9.4 Service Worker / PWA（可选）

当前 [layout.tsx](file:///c%3A/work/dify/web/app/layout.tsx#L43-L45) 动态拼了 `swUrl`。

SPA 建议：

- 将 sw 注册逻辑封装为可开关能力（配置控制）
- 统一在部署域名下提供 `/sw.js`（或 plugin 自动注入）

### 9.5 CSP 与 X-Frame-Options

将 [middleware.ts](file:///c%3A/work/dify/web/middleware.ts) 的规则迁移到部署层：

- **X-Frame-Options**
  - 默认 `DENY`
  - 对 `/chat`、`/workflow`、`/completion`、`/webapp-signin` 放开（保持现有行为）
- **CSP**
  - 白名单由部署环境配置注入（类似 `NEXT_PUBLIC_CSP_WHITELIST`）
  - 建议由网关生成 nonce/策略（如果需要严格脚本控制）

---

## 10. 附录：迁移验收清单（建议每阶段过一遍）

- 路由：
  - 刷新任意业务路由不 404
  - 动态段与 query 参数行为一致
- 鉴权：
  - Console：Cookie/CSRF 正常；401 刷新与跳转正常
  - WebApp：token/passport/share-code 逻辑一致
- 配置：
  - API_PREFIX 等在不同环境可正确注入
  - `NEXT_PUBLIC_COOKIE_DOMAIN`/SameSite 策略在目标拓扑下可用
- 体验：
  - Loading/Error/Toast 行为一致
  - 主题/i18n 切换一致
- 安全：
  - CSP/XFO 生效，iframe 嵌入策略符合预期
- 质量：
  - `pnpm lint:fix`、`pnpm type-check:tsgo`、`pnpm test` 通过
