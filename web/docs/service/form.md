# 表单状态与校验（web）

本文总结 Dify `/web` 前端中表单的状态管理、校验方式与相关技术栈，覆盖 Console 常规页面表单、配置面板类表单、以及工作流节点/Schema 编辑场景。

---

## 1. 技术栈与分层

### 1.1 核心库

- **TanStack React Form (`@tanstack/react-form`)**：核心表单状态管理与字段级/表单级校验框架。
  - 统一入口：[index.tsx](../../app/components/base/form/index.tsx)
- **Zod (`zod`)**：声明式 Schema 校验（常用于提交时校验、或配置驱动表单的 onChange 校验）。
  - 提交校验适配器：[zod-submit-validator.ts](../../app/components/base/form/utils/zod-submit-validator.ts)
- **jsonschema (`jsonschema`)**：用于 JSON Schema（Draft-07）相关校验（主要出现在工作流节点 Schema 编辑/校验，不等同于常规输入表单校验）。
  - Draft-07 校验器：[validators.ts](../../utils/validators.ts)

### 1.2 典型形态（从“表单构建方式”看）

- **AppForm（组件式）**：`useAppForm` + `form.AppField` 渲染字段组件并管理状态。
  - 入口：[index.tsx](../../app/components/base/form/index.tsx)
  - 示例：[installForm.tsx](../../app/install/installForm.tsx), [ForgotPasswordForm.tsx](../../app/forgot-password/ForgotPasswordForm.tsx)
- **BaseForm（Schema 驱动）**：传入 `FormSchema[]` 动态渲染字段（`BaseForm` + `BaseField`）。
  - 入口：[base-form.tsx](../../app/components/base/form/components/base/base-form.tsx), [base-field.tsx](../../app/components/base/form/components/base/base-field.tsx)
- **Scenario Forms（场景封装）**：为特定场景（如 Node Panel、RAG input-field editor）提供更高层的“配置 → 表单”封装。
  - `form-scenarios/base`：[base/index.tsx](../../app/components/base/form/form-scenarios/base/index.tsx)
  - `form-scenarios/node-panel`：[field.tsx](../../app/components/base/form/form-scenarios/node-panel/field.tsx)
- **非表单库的轻量表单**：直接 `useState` 管理输入，校验失败用 Toast 提示（常见于登录/验证码等流程）。
  - 示例：[mail-and-password-auth.tsx](../../app/signin/components/mail-and-password-auth.tsx)

---

## 2. TanStack React Form 模式详解：Hook vs Factory

Dify 项目中主要存在两种 TanStack Form 的使用模式：**基础 Hook 模式 (`useForm`)** 和 **高级工厂模式 (`createFormHook` + `createFormHookContexts`)**。

后者（工厂模式）是 Dify 的**核心表单方案**，用于构建统一、强类型的表单体系；而前者（基础模式）通常仅用于极为简单的独立场景。

### 2.1 模式对比

| 特性 | **基础 Hook 模式** (`useForm`) | **高级工厂模式** (`createFormHook`) |
| :--- | :--- | :--- |
| **生成方式** | 直接 import `useForm` | 通过 `createFormHook` 生成定制 Hook (如 `useAppForm`) |
| **Context** | 需手动配置 `<form.Provider>` | **自动集成** Context (`createFormHookContexts`) |
| **UI 组件绑定** | **无** (需手动渲染 input) | **强绑定** (预注册组件，如 `field.TextField`) |
| **字段组件** | `<form.Field>` | `<form.AppField>` (感知注册组件) |
| **代码量** | 每个 Field 需写完整渲染逻辑 | **极简**，Field 内部逻辑已封装 |
| **类型提示** | 仅基础表单状态提示 | **强类型**，提示可用的注册组件及 Props |
| **适用场景** | 简单、一次性的独立表单 | **大型项目**，有统一 UI 库的表单系统 |

---

### 2.2 高级工厂模式 (`createFormHook` + `createFormHookContexts`)

这是 Dify 项目中**90% 以上表单**采用的模式。它将 TanStack Form 的逻辑层与 Dify 的 UI 组件层（`components/base/*`）进行了深度绑定。

#### 核心实现原理
1.  **基础设施 (`createFormHookContexts`)**：创建 React Context，用于跨组件传递表单状态和字段状态。
2.  **工厂组装 (`createFormHook`)**：将 Context、UI 组件（`TextField` 等）和 TanStack Form 核心逻辑组装，生成 `useAppForm`。

- **定义入口**：[index.tsx](../../app/components/base/form/index.tsx)
  - 注册了 `fieldComponents`（如 `TextField`、`SelectField`）
  - 注册了 `formComponents`（如 `Actions`）
  - 导出了 `useAppForm`、`withForm`、`formContext`、`fieldContext`

#### 使用示例 (`useAppForm`)

```tsx
import { useAppForm } from '@/app/components/base/form'

const MyForm = () => {
  // 1. 使用定制 Hook 初始化表单
  const form = useAppForm({
    defaultValues: { email: '' },
    onSubmit: async ({ value }) => { /* ... */ }
  })

  return (
    // 2. 使用 form.AppField
    <form.AppField name="email">
      {(field) => (
        // 3. 直接使用注册好的 UI 组件 (field.TextField)
        // 自动绑定了 value, onChange, onBlur, error 等逻辑
        <field.TextField label="Email" />
      )}
    </form.AppField>
  )
}
```

#### 优势
- **极简开发**：开发者无需关心 `value` 绑定和 `onChange` 事件，`field.TextField` 内部已通过 Context 自动连接。
- **统一规范**：强制使用设计系统组件，避免样式不统一。
- **AppField**：`form.AppField` 是增强版的字段容器，它将注册的组件注入到 `children` 回调参数中。

---

### 2.3 基础 Hook 模式 (`useForm`)

这是 TanStack Form 的原始用法，适用于不依赖项目统一组件库、需要高度定制 DOM 结构的场景。

#### 使用示例

```tsx
import { useForm } from '@tanstack/react-form'

const SimpleForm = () => {
  const form = useForm({
    defaultValues: { name: '' },
    onSubmit: async ({ value }) => { /* ... */ }
  })

  return (
    // 使用基础 form.Field
    <form.Field name="name">
      {(field) => (
        // 必须手动处理渲染逻辑
        <div>
          <label>Name</label>
          <input
            value={field.state.value}
            onChange={(e) => field.handleChange(e.target.value)}
            onBlur={field.handleBlur}
          />
          {field.state.meta.errors && (
            <span>{field.state.meta.errors.join(',')}</span>
          )}
        </div>
      )}
    </form.Field>
  )
}
```

#### 局限性
- 代码冗余：每个字段都要重复写 label、input、error message 的渲染逻辑。
- 维护成本高：如果设计规范变更（如错误提示样式），需要修改所有散落的 `useForm` 代码。

---

### 2.4 选型建议

在 Dify 项目开发中：

1.  **首选 `useAppForm` (工厂模式)**：
    *   构建常规业务表单（登录、设置、创建应用等）。
    *   需要使用标准 UI 组件（Input, Select, Checkbox）。
    *   需要统一的错误处理和校验反馈。

2.  **慎用 `useForm` (基础模式)**：
    *   仅在极少数需要完全自定义 DOM 结构、且无法复用现有 UI 组件的边缘场景使用。
    *   或者在编写不需要 UI 渲染的纯逻辑表单测试时使用。

3.  **`useState` + Toast (轻量模式)**：
    *   仅限登录/注册等极简交互（如 [mail-and-password-auth.tsx](../../app/signin/components/mail-and-password-auth.tsx)）。
    *   不建议在复杂表单中扩展此模式。

---

## 3. 校验实现方式

### 3.1 字段级校验：`FormSchema.validators` + required 自动补全

Schema 驱动的表单里，每个字段可在 `FormSchema.validators` 中直接提供 TanStack 的字段校验器（`FieldValidators`），并通过 `BaseForm` 渲染时传入：

- `BaseForm` 渲染时注入 `validators`：[base-form.tsx](../../app/components/base/form/components/base/base-form.tsx)
- required 自动校验合并逻辑：[use-get-validators.ts](../../app/components/base/form/hooks/use-get-validators.ts)

`useGetValidators` 的关键策略：

- 当 `required=true` 且字段未提供 `validators` 时，自动生成 `onMount/onChange/onBlur` 三类校验。
- 错误消息使用 i18n：`t('errorMsg.fieldRequired', { ns: 'common', field })`。

这使得“必填校验”在 Schema 表单中具备一致性，并避免每个字段都手写校验器。

### 3.2 提交时校验（Zod）：`zodSubmitValidator(schema)`

在账号设置、安装、忘记密码等“提交即校验”的表单中，常用 Zod 声明一个对象 Schema，然后将其适配为 TanStack 的 `validators.onSubmit`：

- 适配器实现：[zod-submit-validator.ts](../../app/components/base/form/utils/zod-submit-validator.ts)
- 示例用法：[installForm.tsx](../../app/install/installForm.tsx), [ForgotPasswordForm.tsx](../../app/forgot-password/ForgotPasswordForm.tsx)

适配器策略：

- `schema.safeParse(value)` 失败时，将 `issues` 中每条错误按 `issue.path[0]` 映射到 `fields` 字典（仅保留每个字段第一个错误）。
- 返回 `{ fields: Record<string, string> }`，让 TanStack 在对应字段上记录错误。

### 3.3 表单级校验（Zod / 自定义）

部分“配置驱动表单”会在 `validators.onChange` 直接对整个表单值做一次校验，并返回首条错误信息：

- 例子：`form-scenarios/base` 的 `BaseForm`：[base/index.tsx](../../app/components/base/form/form-scenarios/base/index.tsx)
- Schema 生成器：根据 `BaseConfiguration[]` 生成 Zod schema：[utils.ts](../../app/components/base/form/form-scenarios/base/utils.ts)

这种写法适合字段集合随配置变化的场景，但粒度更粗（通常只拿首条错误）——更像“表单整体是否可提交”的守门逻辑。

### 3.4 手写校验 + Toast（不使用表单库）

登录等流程中，有些页面直接用 `useState` 管理输入并做同步校验，失败时使用 Toast 提示：

- 代表：[mail-and-password-auth.tsx](../../app/signin/components/mail-and-password-auth.tsx)
  - 校验规则：邮箱为空/格式、密码为空等
  - 错误展示：`Toast.notify({ type: 'error', message: ... })`

该模式实现成本低，适合字段少、交互简单的表单，但缺少统一的字段级 meta（如 touched/dirty/errors）。

---

## 4. Schema 驱动的 `BaseForm` / `BaseField`

### 4.1 目标：用 `FormSchema[]` 统一描述字段

`FormSchema` 定义了字段类型、label、placeholder、options、show_on、validators 等信息：

- 类型定义：[types.ts](../../app/components/base/form/types.ts)
- 表单渲染容器：[base-form.tsx](../../app/components/base/form/components/base/base-form.tsx)
- 单字段渲染与 UI 适配：[base-field.tsx](../../app/components/base/form/components/base/base-field.tsx)

`BaseForm` 的核心行为：

- 生成默认值：优先使用 props `defaultValues`，否则从 `formSchemas[].default` 汇总。
- 支持条件展示：`show_on` 条件满足才渲染对应字段（渲染层面的隐藏）。
- 将校验器（required 合并后的 validators）传给 `<form.Field name validators>`。
- 提供 `ref` 能力：`getForm/getFormValues/setFields`（见 4.3）。

### 4.2 字段 UI 与交互：`BaseField`

`BaseField` 负责把 `FormSchema.type` 映射到具体输入组件，并处理：

- i18n：label/placeholder/tooltip/description/help 支持对象化多语言（`useRenderI18nObject`）。
- 选项依赖：`options[].show_on` 支持“选项级联显示”（并不会丢失字段状态，只是筛选可选项）。
- 动态选项：`dynamic-select` 通过 `useTriggerPluginDynamicOptions` 异步拉取 options，并处理 loading/empty/error 提示。
- 错误展示：依赖 `fieldState.validateStatus` 与 `fieldState.errors/warnings`，样式映射见 `VALIDATE_STATUS_STYLE_MAP`。

对应实现参考：[base-field.tsx](../../app/components/base/form/components/base/base-field.tsx)。

### 4.3 `getFormValues`：提交前校验门禁 + Secret 字段保护

Schema 表单常需要“从表单实例读值并提交”。项目提供了一个带门禁的 `getFormValues`：

- 入口：[use-get-form-values.ts](../../app/components/base/form/hooks/use-get-form-values.ts)
- 校验门禁（会 Toast 首条错误）：[use-check-validated.ts](../../app/components/base/form/hooks/use-check-validated.ts)

行为要点：

- `needCheckValidatedValues=true` 时，先执行 `checkValidated()`：
  - 通过 `form.getAllErrors()` 汇总错误
  - 对“被 show_on 隐藏的字段”忽略其错误（避免隐藏字段阻断提交）
  - 若有错误：Toast 首条错误并返回 `isCheckValidated=false`
- `needTransformWhenSecretFieldIsPristine=true` 时：
  - 将 `secret-input` 且 `fieldMeta.isPristine=true` 的字段值替换为 `"[__HIDDEN__]"`，避免“未修改的密钥”在提交时被当作明文重复提交
  - 实现：[secret-input/index.ts](../../app/components/base/form/utils/secret-input/index.ts)

### 4.4 `setFields`：从外部注入错误/警告态

`BaseForm` 维护了一份 `fieldStates`，并通过 `ref.setFields([{ name, value, errors, warnings, validateStatus, help }])` 允许外部：

- 批量设置字段值（`form.setFieldValue`）
- 注入 errors/warnings/help 以及 `validateStatus`（若未显式给定，按 errors/warnings 自动推导）

实现参考：[base-form.tsx](../../app/components/base/form/components/base/base-form.tsx)。

---

## 5. `form-scenarios`：面向场景的“配置 → 表单”

### 5.1 `form-scenarios/base`：`BaseConfiguration[]` 生成 Zod schema

该场景以 `BaseConfiguration[]` 描述字段（类型、required、max/min 等），通过 `generateZodSchema` 生成一个整体 Zod schema，然后在 `validators.onChange` 中进行全量校验：

- 字段配置结构：[types.ts](../../app/components/base/form/form-scenarios/base/types.ts)
- schema 生成器：[utils.ts](../../app/components/base/form/form-scenarios/base/utils.ts)
- 组装与渲染：[base/index.tsx](../../app/components/base/form/form-scenarios/base/index.tsx)

特点：

- 适合“字段配置来自后端/编辑器”的场景
- 校验逻辑集中但粒度偏粗（首条错误为主）

### 5.2 `form-scenarios/node-panel`：`withForm` + 条件展示

Node Panel 的字段渲染将“展示条件”定义为 `showConditions`，并通过 `useStore(form.store, selector)` 实时判断是否展示：

- 实现：[field.tsx](../../app/components/base/form/form-scenarios/node-panel/field.tsx)

它与 `BaseForm.show_on` 的差异在于：

- `BaseForm.show_on` 属于 `FormSchema[]` 的字段级条件（Schema 驱动表单通用能力）
- Node Panel 的 `showConditions` 是场景类型（InputFieldConfiguration）的展示条件，且通常与工作流节点配置数据结构绑定

---

## 6. JSON Schema（Draft-07）校验（工作流相关）

这部分不是“输入表单校验”，但属于 `/web` 中的“结构化配置校验”能力，常见于工作流节点的 Schema 编辑/推导/校验。

- Draft-07 validator：[validators.ts](../../utils/validators.ts)
  - `draft07Validator(schema)`：基于 `jsonschema.Validator` + 本地 `draft-07.json`
  - `forbidBooleanProperties(schema)`：项目自定义规则，禁止 boolean schema（避免旧逻辑不兼容）
- 使用方示例：`validateSchemaAgainstDraft7` 等工具函数位于 [utils.ts](../../app/components/workflow/nodes/llm/utils.ts)

---

## 7. 选型建议（在 Dify 现有模式下）

- 需要“复用字段组件、字段级 meta、提交/校验一致性”时：优先使用 `useAppForm` 体系（见 [index.tsx](../../app/components/base/form/index.tsx)）。
- 需要“根据 schema 动态渲染字段、支持 show_on、动态选项、secret 字段保护、外部注入错误”时：使用 `BaseForm` + `FormSchema[]`（见 [base-form.tsx](../../app/components/base/form/components/base/base-form.tsx)）。
- 需要“配置驱动的一次性表单（字段集合随配置变化明显）”时：可以使用 `form-scenarios/*`（见 [base/index.tsx](../../app/components/base/form/form-scenarios/base/index.tsx)）。
- 仅少量字段、轻量交互：可用 `useState + Toast`（见 [mail-and-password-auth.tsx](../../app/signin/components/mail-and-password-auth.tsx)），但不建议在复杂表单中扩展该模式。
