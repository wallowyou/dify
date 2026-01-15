# React Query (TanStack Query) 使用指南

`@tanstack/react-query` 是 Dify 前端项目中用于管理服务器状态（Server State）的核心库。它负责数据的获取、缓存、同步和更新，极大简化了异步数据管理的复杂性。

## 核心概念

1.  **Query (查询)**: 用于获取数据 (GET)。每个查询由唯一的 `queryKey` 标识。
2.  **Mutation (变更)**: 用于修改数据 (POST, PUT, DELETE)。
3.  **QueryClient**: 管理缓存和配置的全局实例。

---

## 基础用法

### 1. 获取数据 (`useQuery`)

最常用的 Hook，用于从 API 获取数据并自动缓存。

```typescript
import { useQuery } from '@tanstack/react-query'
import { fetchAppList } from '@/service/apps'

export const useAppList = (params) => {
  return useQuery({
    // 1. Query Key: 唯一标识，类似依赖数组。当 params 变化时，会自动重新请求
    queryKey: ['apps', params], 
    
    // 2. Query Function: 必须返回一个 Promise
    queryFn: () => fetchAppList(params),
    
    // 3. Options (可选配置)
    enabled: !!params.userId, // 仅当 userId 存在时才请求
    staleTime: 60 * 1000,     // 数据 60秒内被认为是新鲜的，不会重新请求
  })
}
```

**组件中使用:**

```tsx
const { data, isLoading, isError, error } = useAppList({ page: 1 })

if (isLoading) return <Loading />
if (isError) return <div>Error: {error.message}</div>

return (
  <ul>
    {data.data.map(app => <li key={app.id}>{app.name}</li>)}
  </ul>
)
```

#### 常用参数 (Options)

| 参数 | 类型 | 描述 |
| :--- | :--- | :--- |
| `queryKey` | `Array` | **必填**。用于缓存管理的唯一标识数组。依赖项变化时会自动触发重新请求。 |
| `queryFn` | `Function` | **必填**。返回 Promise 的异步函数，用于获取数据。 |
| `enabled` | `boolean` | 默认为 `true`。设为 `false` 时不会自动发起请求（常用于依赖其他数据就绪时）。 |
| `staleTime` | `number` | 数据保持“新鲜”的时间（毫秒）。在此期间读取缓存不会触发后台重新请求。 |
| `gcTime` | `number` | 闲置缓存数据的保留时间（毫秒）。默认 5 分钟。 (v5+ 名称，旧版为 `cacheTime`) |
| `refetchOnWindowFocus` | `boolean` | 窗口重新获得焦点时是否重新请求。 |
| `select` | `Function` | 用于对返回数据进行转换或筛选。 |
| `retry` | `boolean/number` | 请求失败时的重试次数。默认为 3 次。 |

#### 常用返回值 (Result)

| 属性 | 类型 | 描述 |
| :--- | :--- | :--- |
| `data` | `TData` | 异步请求返回的数据。默认为 `undefined`。 |
| `error` | `Error` | 请求失败时的错误对象。 |
| `isLoading` | `boolean` | 是否正在进行**首次**加载且暂无缓存数据。 |
| `isFetching` | `boolean` | 是否正在进行**任意**请求（包括后台更新）。 |
| `isError` | `boolean` | 请求是否失败。 |
| `isSuccess` | `boolean` | 请求是否成功且有数据。 |
| `refetch` | `Function` | 手动触发查询的函数。 |

### 2. 修改数据 (`useMutation`)

用于执行创建、更新或删除操作。

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { createApp } from '@/service/apps'

export const useCreateApp = () => {
  const queryClient = useQueryClient()

  return useMutation({
    // Mutation Function
    mutationFn: (newApp) => createApp(newApp),
    
    // 成功回调
    onSuccess: () => {
      // 关键：让 'apps' 缓存失效，触发列表自动刷新
      queryClient.invalidateQueries({ queryKey: ['apps'] })
    },
    
    onError: (error) => {
      console.error('Failed to create app:', error)
    }
  })
}
```

**组件中使用:**

```tsx
const { mutate, isPending } = useCreateApp()

const handleSubmit = () => {
  mutate({ name: 'My New App' })
}
```

#### 常用参数 (Options)

| 参数 | 类型 | 描述 |
| :--- | :--- | :--- |
| `mutationFn` | `Function` | **必填**。执行异步操作的函数（如 POST/PUT 请求）。 |
| `onSuccess` | `Function` | Mutation 成功后的回调。常用于 invalidation 或更新 UI。 |
| `onError` | `Function` | Mutation 失败后的回调。 |
| `onSettled` | `Function` | 无论成功或失败都会执行的回调。 |
| `onMutate` | `Function` | Mutation 执行前触发。常用于乐观更新 (Optimistic UI)。 |

#### 常用返回值 (Result)

| 属性 | 类型 | 描述 |
| :--- | :--- | :--- |
| `mutate` | `Function` | 用于触发 mutation 的同步函数。参数将传递给 `mutationFn`。 |
| `mutateAsync` | `Function` | 与 `mutate` 类似，但返回 Promise，可使用 `await` 等待结果。 |
| `data` | `TData` | Mutation 成功后的返回数据。 |
| `isPending` | `boolean` | 是否正在执行 Mutation。 (v5+ 名称，旧版为 `isLoading`) |
| `isError` | `boolean` | 是否执行失败。 |
| `isSuccess` | `boolean` | 是否执行成功。 |
| `reset` | `Function` | 重置 mutation 状态（清除 `data` 和 `error`）。 |

---

## 推荐写法与最佳实践

### 1. 封装 Custom Hooks (推荐)
不要直接在组件中调用 `useQuery`，而是将其封装在 `service/use-*.ts` 文件中。这样可以复用逻辑并保持组件整洁。

**❌ 不推荐:**
```tsx
// Component.tsx
const { data } = useQuery({ queryKey: ['apps'], queryFn: fetchApps })
```

**✅ 推荐:**
```typescript
// service/use-apps.ts
export const useApps = () => useQuery({ queryKey: ['apps'], queryFn: fetchApps })

// Component.tsx
const { data } = useApps()
```

### 2. Query Key 的管理
Query Key 是缓存的基石。建议遵循从通用到具体的原则，并包含所有依赖变量。

*   **列表**: `['entity-list', { filter, page }]`
*   **详情**: `['entity-detail', id]`
*   **关联**: `['entity-comments', entityId]`

### 3. 利用 `select` 转换数据
如果只需要后端返回数据的一部分，或者需要转换格式，使用 `select` 选项。这具有记忆化 (Memoization) 效果，性能更好。

```typescript
useQuery({
  queryKey: ['user', id],
  queryFn: fetchUser,
  select: (data) => ({
    fullName: `${data.firstName} ${data.lastName}`,
    isAdmin: data.role === 'admin'
  })
})
```

### 4. 乐观更新 (Optimistic Updates)
对于用户体验要求高的操作（如点赞、改名），可以在服务器返回前先更新 UI。

```typescript
useMutation({
  mutationFn: updateName,
  onMutate: async (newName) => {
    // 1. 取消正在进行的查询
    await queryClient.cancelQueries({ queryKey: ['user'] })
    // 2. 保存旧数据快照
    const previousUser = queryClient.getQueryData(['user'])
    // 3. 乐观更新缓存
    queryClient.setQueryData(['user'], (old) => ({ ...old, name: newName }))
    return { previousUser }
  },
  onError: (err, newName, context) => {
    // 4. 出错时回滚
    queryClient.setQueryData(['user'], context.previousUser)
  },
  onSettled: () => {
    // 5. 完成后重新拉取最新数据
    queryClient.invalidateQueries({ queryKey: ['user'] })
  }
})
```

## Dify 项目中的配置

Dify 在 `web/context/query-client.tsx` (客户端) 和 `web/context/query-client-server.ts` (服务端) 中配置了全局默认值：

*   **staleTime**: 默认为 0（或服务端渲染时的 30分钟），意味着数据默认被认为是过期的，切换窗口时会重新获取。
*   **refetchOnWindowFocus**: 默认为 `false` (在某些配置中)，避免频繁切换导致的网络请求。

## 常见问题排查

1.  **数据一直 Loading**: 检查 `queryFn` 是否正确返回了 Promise，且 Promise 最终 resolve 了。
2.  **无限重复请求**: 检查 `useEffect` 或组件渲染逻辑，或者 `queryFn` 抛出了错误导致不断重试（默认重试 3 次）。
3.  **缓存不更新**: 检查 `queryKey` 是否包含了所有变化的参数，或者 Mutation 后是否忘记调用 `invalidateQueries`。
