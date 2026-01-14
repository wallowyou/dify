# Ky 技术文档

> Ky 是一个基于 [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch) 的微型且优雅的 HTTP 客户端。

[![Coverage Status](https://codecov.io/gh/sindresorhus/ky/branch/main/graph/badge.svg)](https://codecov.io/gh/sindresorhus/ky)
[![Bundle Size](https://badgen.net/bundlephobia/minzip/ky)](https://bundlephobia.com/result?p=ky)

- **平台支持**：现代浏览器、Node.js、Bun 和 Deno
- **核心特点**：零依赖、体积小巧、功能强大

---

## 🌟 为什么选择 Ky？

相比原生 `fetch`，Ky 提供了更人性化的 API 和更多内置功能：

- ✅ **更简单的 API**
- ✅ **方法快捷方式** (`ky.post()`, `ky.get()` 等)
- ✅ **自动错误处理**：将非 2xx 状态码视为错误
- ✅ **自动重试**：网络失败或特定状态码自动重试
- ✅ **JSON 增强**：自动序列化与解析
- ✅ **超时支持**：内置请求超时控制
- ✅ **URL 前缀**：方便处理 API 路径
- ✅ **钩子机制 (Hooks)**：请求/响应拦截
- ✅ **TypeScript 友好**：泛型支持与更好的类型推断

---

## 📦 安装

### Node.js / Webpack

```bash
npm install ky
```

### CDN / Deno

- **jsdelivr**: `https://cdn.jsdelivr.net/npm/ky`
- **unpkg**: `https://unpkg.com/ky`
- **esm.sh**: `https://esm.sh/ky`

**Deno 示例**:
```typescript
import ky from 'https://esm.sh/ky';
```

---

## 🚀 快速上手

### 基础用法

```javascript
import ky from 'ky';

const json = await ky.post('https://example.com', {
  json: { foo: true }
}).json();

console.log(json);
//=> {data: '🦄'}
```

### 对比原生 fetch

<details>
<summary>点击查看原生 fetch 实现对比</summary>

```javascript
class HTTPError extends Error {}

const response = await fetch('https://example.com', {
  method: 'POST',
  body: JSON.stringify({ foo: true }),
  headers: {
    'content-type': 'application/json'
  }
});

if (!response.ok) {
  throw new HTTPError(`Fetch error: ${response.statusText}`);
}

const json = await response.json();
console.log(json);
```
</details>

---

## 📚 API 参考

### `ky(input, options?)`

核心函数，行为类似 `fetch` 但功能更强。

- **返回值**: `Response` 对象（扩展了快捷 Body 方法）。
- **快捷 Body 方法**: `.json()`, `.text()`, `.formData()`, `.arrayBuffer()`, `.blob()`, `.bytes()`。

> **注意**: 
> - 如果响应状态码不在 200-299 之间，Body 方法会抛出 `HTTPError`。
> - 如果 Body 为空或状态码为 204，`.json()` 返回空字符串而非抛错。

#### 示例

```typescript
// JavaScript
const user = await ky('/api/user').json();

// TypeScript (支持泛型)
const user = await ky('/api/users/1').json<User>();
```

### HTTP 方法快捷方式

- `ky.get(input, options?)`
- `ky.post(input, options?)`
- `ky.put(input, options?)`
- `ky.patch(input, options?)`
- `ky.head(input, options?)`
- `ky.delete(input, options?)`

---

## ⚙️ 选项 (Options)

`ky` 支持所有 `fetch` 的选项，并额外支持以下配置：

### `method`
- **类型**: `string`
- **默认值**: `'get'`
- **说明**: HTTP 方法，自动转为大写。

### `json`
- **类型**: `object` | `any`
- **说明**: 发送 JSON 数据的快捷方式。会自动设置 `Content-Type: application/json` 并序列化。代替 `body` 使用。

### `searchParams`
- **类型**: `string` | `object` | `URLSearchParams`
- **默认值**: `''`
- **说明**: URL 查询参数。`undefined` 值会被过滤。

### `prefixUrl`
- **类型**: `string` | `URL`
- **说明**: URL 前缀。常用于创建 API 实例。
- **注意**: 使用此选项时，`input` 不能以 `/` 开头。

```javascript
const api = ky.create({ prefixUrl: 'https://example.com/api' });
await api.get('users/123'); // => https://example.com/api/users/123
```

### `retry`
- **类型**: `object` | `number`
- **默认值**: 
  ```javascript
  {
    limit: 2,
    methods: ['get', 'put', 'head', 'delete', 'options', 'trace'],
    statusCodes: [408, 413, 429, 500, 502, 503, 504],
    afterStatusCodes: [413, 429, 503],
    maxRetryAfter: undefined,
    backoffLimit: undefined,
    retryOnTimeout: false
  }
  ```
- **配置项**:
  - `limit`: 最大重试次数。
  - `methods`: 允许重试的方法。
  - `statusCodes`: 触发重试的状态码。
  - `retryOnTimeout`: 是否在超时时重试。
  - `jitter`: 是否启用随机抖动防止惊群效应。
  - `shouldRetry`: 自定义重试逻辑函数。

### `timeout`
- **类型**: `number` | `false`
- **默认值**: `10000` (10秒)
- **说明**: 请求超时时间（毫秒）。设为 `false` 禁用。

### `hooks`
- **类型**: `object`
- **默认值**: `{ beforeRequest: [], beforeRetry: [], afterResponse: [], beforeError: [] }`
- **说明**: 生命周期钩子。

#### 1. `beforeRequest`
在请求发送前修改请求。
```javascript
hooks: {
  beforeRequest: [
    (request) => {
      request.headers.set('X-Requested-With', 'ky');
    }
  ]
}
```

#### 2. `beforeRetry`
在重试发生前触发。
```javascript
hooks: {
  beforeRetry: [
    async ({ request, options, error, retryCount }) => {
      const token = await refreshToken();
      request.headers.set('Authorization', `Bearer ${token}`);
    }
  ]
}
```

#### 3. `afterResponse`
响应返回后触发。可用于重试策略或日志。
```javascript
hooks: {
  afterResponse: [
    (_request, _options, response) => {
      if (response.status === 403) {
        console.log('Forbidden');
      }
    }
  ]
}
```

#### 4. `beforeError`
在抛出 `HTTPError` 前修改错误对象。

---

## 🛠 高级功能

### 进度监听
支持 `onDownloadProgress` 和 `onUploadProgress`。

```javascript
const response = await ky('https://example.com', {
  onDownloadProgress: (progress, chunk) => {
    console.log(`${progress.percent * 100}% - ${progress.transferredBytes} bytes`);
  }
});
```

### 自定义实例 `ky.create()`
创建一个带有默认配置的新 Ky 实例。

```javascript
const api = ky.create({
  prefixUrl: 'https://api.example.com',
  headers: {
    'Authorization': 'Bearer my-token'
  }
});
```

### 扩展实例 `ky.extend()`
基于现有实例继承并修改配置。

```javascript
const newApi = api.extend({
  headers: {
    'Custom-Header': 'baz'
  }
});
```

### 取消请求
使用标准的 `AbortController`。

```javascript
const controller = new AbortController();
const { signal } = controller;

setTimeout(() => controller.abort(), 5000);

await ky(url, { signal });
```

### Node.js 代理支持
支持 `undici` 的 `ProxyAgent` 或环境变量代理。

```javascript
import { ProxyAgent } from 'undici';

const response = await ky('https://example.com', {
  dispatcher: new ProxyAgent('http://proxy.example.com:8080')
});
```

---

## 📘 TypeScript 支持

Ky 的类型定义使用类型别名而非接口，以避免全局污染。如果需要扩展错误类型，建议使用包装类型。

```typescript
import ky, { HTTPError } from 'ky';

interface CustomError extends HTTPError {
  customProperty: unknown;
}

const api = ky.extend({
  hooks: {
    beforeError: [
      async error => {
        (error as CustomError).customProperty = 'value';
        return error;
      }
    ]
  }
});
```

---

## ❓ FAQ

**Q: 如何在 Node.js 中使用？**
A: Node.js 18+ 原生支持 fetch，可直接使用 Ky。

**Q: 浏览器支持情况？**
A: 支持最新版本的 Chrome, Firefox, Safari。

**Q: 与 Axios 的区别？**
A: Ky 基于标准 Fetch API 构建，体积更小，语法更现代。

---

> **维护者**: Sindre Sorhus, Seth Holladay, Szymon Marczak
