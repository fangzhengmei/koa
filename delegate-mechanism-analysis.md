# Koa Delegate 机制深度分析

## 1. 概述

Koa 框架中，`ctx.url`、`ctx.status` 等属性并非直接挂载在 Context 对象上，而是通过 **delegate（委托）机制** 代理到底层的 Request 和 Response 对象。本文将深入分析这套委托机制的实现原理、getter/setter 的方向对称性，以及 `ctx.status` 与直接操作底层对象的差异。

## 2. 核心架构

### 2.1 对象关系图

在 Koa 中，每次请求都会创建一个 Context 对象，该对象内部包含 Request 和 Response 对象：

```
┌─────────────────────────────────────────────────────────┐
│                      Context (ctx)                        │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────┐          ┌──────────────┐            │
│  │   Request    │◄────────►│   Response   │            │
│  │ ctx.request  │          │ ctx.response │            │
│  └──────┬───────┘          └──────┬───────┘            │
│         │                          │                      │
│         ▼                          ▼                      │
│  ┌──────────────┐          ┌──────────────┐            │
│  │  Incoming    │          │ServerResponse│            │
│  │   Message    │          │  (Node.js)   │            │
│  │  (Node.js)   │          │   ctx.res    │            │
│  │    ctx.req   │          └──────────────┘            │
│  └──────────────┘                                        │
└─────────────────────────────────────────────────────────┘
```

### 2.2 对象创建流程

从 `application.js` 的 `createContext` 方法可以看到对象的创建和关联过程：

```javascript
// lib/application.js:213-229
createContext (req, res) {
  const context = Object.create(this.context)
  const request = (context.request = Object.create(this.request))
  const response = (context.response = Object.create(this.response))
  
  // 建立相互引用
  context.app = request.app = response.app = this
  context.req = request.req = response.req = req
  context.res = request.res = response.res = res
  request.ctx = response.ctx = context
  request.response = response
  response.request = request
  context.originalUrl = request.originalUrl = req.url
  context.state = {}
  return context
}
```

**关键点**：
- Context、Request、Response 对象都通过 `Object.create()` 从原型创建
- 所有对象共享同一个 `req`（IncomingMessage）和 `res`（ServerResponse）
- 对象之间建立了双向引用：`request.ctx = context` 且 `context.request = request`

## 3. Delegate 机制实现原理

### 3.1 delegates 库简介

Koa 使用了 [delegates](https://www.npmjs.com/package/delegates) 库来实现委托机制。该库由 TJ Holowaychuk 开发，提供了优雅的 API 来在原型对象之间构建方法、getter、setter 的委托。

### 3.2 Context 中的委托配置

在 `lib/context.js` 中，通过 delegates 库配置了详细的委托规则：

```javascript
// lib/context.js:194-248

/**
 * Response delegation.
 */
delegate(proto, 'response')
  .method('attachment')
  .method('redirect')
  .method('remove')
  .method('vary')
  .method('has')
  .method('set')
  .method('append')
  .method('flushHeaders')
  .method('back')
  .access('status')
  .access('message')
  .access('body')
  .access('length')
  .access('type')
  .access('lastModified')
  .access('etag')
  .getter('headerSent')
  .getter('writable')

/**
 * Request delegation.
 */
delegate(proto, 'request')
  .method('acceptsLanguages')
  .method('acceptsEncodings')
  .method('acceptsCharsets')
  .method('accepts')
  .method('get')
  .method('is')
  .access('querystring')
  .access('idempotent')
  .access('socket')
  .access('search')
  .access('method')
  .access('query')
  .access('path')
  .access('url')
  .access('accept')
  .getter('origin')
  .getter('href')
  .getter('subdomains')
  .getter('protocol')
  .getter('host')
  .getter('hostname')
  .getter('URL')
  .getter('header')
  .getter('headers')
  .getter('secure')
  .getter('stale')
  .getter('fresh')
  .getter('ips')
  .getter('ip')
```

### 3.3 委托类型详解

delegates 库提供了三种主要的委托方式：

| 委托方式 | 说明 | 示例 |
|---------|------|------|
| `method(name)` | 委托方法调用 | `ctx.redirect('/home')` → `ctx.response.redirect('/home')` |
| `access(name)` | 同时委托 getter 和 setter（双向） | `ctx.status = 200` → `ctx.response.status = 200` |
| `getter(name)` | 只委托 getter（只读） | `ctx.ip` → 读取 `ctx.request.ip`，不可写入 |

### 3.4 委托实现原理

delegates 库的核心实现原理是通过 `Object.defineProperty` 在原型上定义属性描述符：

**对于 `method` 委托**：
```javascript
// 伪代码实现
proto[name] = function(...args) {
  return this[targetProp][name](...args);
}
```

**对于 `access` 委托**：
```javascript
// 伪代码实现
Object.defineProperty(proto, name, {
  get() {
    return this[targetProp][name];
  },
  set(val) {
    this[targetProp][name] = val;
  },
  configurable: true,
  enumerable: true
});
```

**对于 `getter` 委托**：
```javascript
// 伪代码实现
Object.defineProperty(proto, name, {
  get() {
    return this[targetProp][name];
  },
  configurable: true,
  enumerable: true
});
```

## 4. Getter 和 Setter 的方向对称性分析

### 4.1 对称性定义

在委托机制中，**方向对称性**指的是：
- **Getter 方向**：`ctx.prop` → `ctx.request.prop` 或 `ctx.response.prop`
- **Setter 方向**：`ctx.prop = value` → `ctx.request.prop = value` 或 `ctx.response.prop = value`

### 4.2 不同委托类型的对称性

#### 4.2.1 `access` 委托 - 完全对称

使用 `access` 委托的属性，getter 和 setter 是完全对称的：

**示例：`ctx.status`**
```javascript
// 读取
const code = ctx.status;
// 等价于：
const code = ctx.response.status;
// 最终读取：ctx.res.statusCode

// 写入
ctx.status = 200;
// 等价于：
ctx.response.status = 200;
// 最终写入会触发 response.js 中的 setter 逻辑
```

#### 4.2.2 `getter` 委托 - 不对称（只读）

使用 `getter` 委托的属性，**只有 getter 被委托，setter 不存在**：

**示例：`ctx.ip`**
```javascript
// 读取 - 正常工作
const ip = ctx.ip;
// 等价于：
const ip = ctx.request.ip;

// 写入 - 不会报错，但也不会生效！
ctx.ip = '127.0.0.1';
// 这会在 ctx 对象上创建一个新属性 'ip'，
// 而不是委托到 ctx.request.ip
// 下次读取 ctx.ip 时，仍会返回 ctx.request.ip 的值
```

**重要陷阱**：对 `getter` 委托的属性赋值不会报错，但会在 Context 对象上创建一个新的自有属性，遮蔽了委托的 getter。

### 4.3 Koa 中的委托对称性统计

#### Response 委托（`delegate(proto, 'response')`）

| 属性/方法 | 委托类型 | 可读取 | 可写入 | 对称性 |
|----------|---------|--------|--------|--------|
| attachment, redirect, remove, vary, has, set, append, flushHeaders, back | method | ✅ | - | - |
| status, message, body, length, type, lastModified, etag | access | ✅ | ✅ | 对称 |
| headerSent, writable | getter | ✅ | ❌ | 不对称 |

#### Request 委托（`delegate(proto, 'request')`）

| 属性/方法 | 委托类型 | 可读取 | 可写入 | 对称性 |
|----------|---------|--------|--------|--------|
| acceptsLanguages, acceptsEncodings, acceptsCharsets, accepts, get, is | method | ✅ | - | - |
| querystring, idempotent, socket, search, method, query, path, url, accept | access | ✅ | ✅ | 对称 |
| origin, href, subdomains, protocol, host, hostname, URL, header, headers, secure, stale, fresh, ips, ip | getter | ✅ | ❌ | 不对称 |

### 4.4 对称性问题的实际影响

**场景 1：错误地对只读属性赋值**
```javascript
// ⚠️ 这是一个常见错误
app.use(async (ctx) => {
  // 试图修改只读属性
  ctx.ip = '192.168.1.1';  // 不会报错，但也不会生效！
  
  // 下次读取
  console.log(ctx.ip);  // 仍返回真实的客户端 IP，不是 '192.168.1.1'
  
  // 但如果直接访问自有属性
  console.log(Object.getOwnPropertyDescriptor(ctx, 'ip')); 
  // 会发现 ctx 上确实有了一个 'ip' 属性，值为 '192.168.1.1'
  // 但读取时优先使用了原型上的 getter
});
```

**场景 2：正确的做法**
```javascript
// 对于需要修改的属性，确保使用 access 委托的属性
// 或者直接操作底层对象
app.use(async (ctx) => {
  // ✅ 正确：access 委托的属性可以正常读写
  ctx.status = 200;
  ctx.body = 'Hello';
  
  // ✅ 正确：直接操作 Request 对象
  ctx.request.method = 'POST';  // method 是 access 委托
  
  // ❌ 错误：ip 是 getter 委托，不可写入
  // ctx.ip = '...'
});
```

## 5. `ctx.status` 与直接操作底层对象的差异

### 5.1 `ctx.status` 的完整实现

让我们深入分析 `response.js` 中 `status` 属性的实现：

```javascript
// lib/response.js:73-93

/**
 * Get response status code.
 *
 * @return {Number}
 * @api public
 */

get status () {
  return this.res.statusCode
},

/**
 * Set response status code.
 *
 * @param {Number} code
 * @api public
 */

set status (code) {
  if (this.headerSent) return  // 🔒 检查：如果响应头已发送，直接返回

  assert(Number.isInteger(code), 'status code must be a number')  // ✅ 验证 1：必须是整数
  assert(code >= 100 && code <= 999, `invalid status code: ${code}`)  // ✅ 验证 2：必须在有效范围内
  
  this._explicitStatus = true  // 📝 标记：明确设置了状态码
  
  this.res.statusCode = code  // 🔧 实际设置：写入底层 ServerResponse
  
  // HTTP/1.x 兼容性：设置状态消息
  if (this.req.httpVersionMajor < 2) 
    this.res.statusMessage = statuses.message[code]
  
  // 🧹 清理：如果状态码表示"无内容"，自动清空 body
  if (this.body && statuses.empty[code]) 
    this.body = null
},
```

### 5.2 三种操作方式的对比

| 操作方式 | 代码示例 | 效果 | 安全性 | 推荐度 |
|---------|---------|------|--------|--------|
| **推荐方式**：`ctx.status = 204` | 完整逻辑 | ✅ 所有验证和副作用 | 最高 | ⭐⭐⭐ |
| **不推荐**：`ctx.response.status = 204` | 大部分逻辑 | ✅ 验证和副作用，但跳过委托层 | 中 | ⭐⭐ |
| **危险方式**：`ctx.res.statusCode = 204` | 仅直接赋值 | ❌ 无验证，无副作用 | 最低 | ⭐ |

### 5.3 差异详细分析

#### 差异 1：响应头已发送检查

```javascript
// ctx.status = 200 的行为：
// 如果响应头已发送，setter 会直接返回，不执行任何操作

// ctx.res.statusCode = 200 的行为：
// 无论响应头是否已发送，都会直接修改，可能导致不可预期的行为
```

#### 差异 2：状态码验证

```javascript
// ctx.status = 'invalid'
// 抛出 AssertionError: status code must be a number

// ctx.status = 9999
// 抛出 AssertionError: invalid status code: 9999

// ctx.res.statusCode = 'invalid'
// 静默失败，或者设置为 NaN/0，取决于 Node.js 版本

// ctx.res.statusCode = 9999
// 可能设置成功，但不符合 HTTP 规范
```

#### 差异 3：`_explicitStatus` 标记

```javascript
// ctx.status = 200 会设置 this._explicitStatus = true
// 这个标记会影响 body setter 的行为：

// lib/response.js:168
// set the status
if (!this._explicitStatus) this.status = 200
// 当设置 body 时，如果没有明确设置过 status，会自动设为 200

// 直接设置 ctx.res.statusCode 不会设置 _explicitStatus
// 可能导致 body setter 意外地修改状态码
```

#### 差异 4：HTTP/1.x 状态消息

```javascript
// ctx.status = 404
// HTTP/1.x 时会自动设置：ctx.res.statusMessage = 'Not Found'

// ctx.res.statusCode = 404
// 不会设置 statusMessage，可能保持默认或之前的值
```

#### 差异 5：空状态码的 body 自动清理

```javascript
// 204 (No Content), 304 (Not Modified) 等状态码不应有 body

// ctx.status = 204
// 如果之前有设置 body，会自动设为 null
// 同时会移除 Content-Type, Content-Length 等头部

// ctx.res.statusCode = 204
// body 保持不变，可能导致响应不符合 HTTP 规范
```

### 5.4 实际场景演示

**场景 1：设置 204 No Content**

```javascript
// ✅ 推荐：使用 ctx.status
app.use(async (ctx) => {
  ctx.body = { message: 'Hello' };  // 先设置 body
  ctx.status = 204;  // 设置 204，会自动清空 body
  
  // 结果：
  // ctx.body === null
  // Content-Type 被移除
  // Content-Length 被移除
  // 符合 HTTP 规范
});

// ❌ 危险：直接操作底层对象
app.use(async (ctx) => {
  ctx.body = { message: 'Hello' };
  ctx.res.statusCode = 204;  // 直接设置，不触发任何逻辑
  
  // 结果：
  // ctx.body 仍然是 { message: 'Hello' }
  // 响应仍然会发送 JSON body
  // 违反 HTTP 规范（204 不应有 body）
});
```

**场景 2：响应头已发送后尝试修改**

```javascript
// ✅ 使用 ctx.status - 安全
app.use(async (ctx) => {
  // 先发送响应头
  ctx.res.writeHead(200);
  ctx.res.flushHeaders();
  
  // 尝试修改状态码
  ctx.status = 500;  // 静默忽略，不报错，不修改
  
  // 结果：响应状态码仍然是 200
});

// ❌ 直接操作 - 可能导致错误
app.use(async (ctx) => {
  ctx.res.writeHead(200);
  ctx.res.flushHeaders();
  
  ctx.res.statusCode = 500;  // 行为不确定
  // 可能静默失败，可能导致内部状态不一致
  // 取决于 Node.js 版本和具体实现
});
```

**场景 3：状态码验证**

```javascript
// ✅ 使用 ctx.status - 早期报错
app.use(async (ctx) => {
  try {
    ctx.status = 'not a number';  // 立即抛出错误
  } catch (err) {
    // 可以捕获并处理
    console.error('Invalid status code:', err.message);
    ctx.status = 500;
  }
});

// ❌ 直接操作 - 延迟失败或静默失败
app.use(async (ctx) => {
  ctx.res.statusCode = 'not a number';  // 可能不报错
  // 但在发送响应时可能导致各种问题
  // 错误可能在完全不同的地方出现，难以调试
});
```

## 6. 最佳实践建议

### 6.1 始终通过 Context 委托访问

```javascript
// ✅ 推荐
ctx.status = 200;
ctx.body = 'Hello';
ctx.redirect('/home');

// ❌ 不推荐
ctx.response.status = 200;
ctx.response.body = 'Hello';
ctx.res.statusCode = 200;
```

### 6.2 了解属性的可写性

在使用前，了解哪些属性是只读的（getter 委托）：

```javascript
// 只读属性（getter 委托）- 不要尝试修改
ctx.ip, ctx.ips, ctx.protocol, ctx.secure, 
ctx.host, ctx.hostname, ctx.origin, ctx.href,
ctx.header, ctx.headers, ctx.fresh, ctx.stale,
ctx.subdomains, ctx.headerSent, ctx.writable

// 可读写属性（access 委托）- 可以正常使用
ctx.status, ctx.message, ctx.body, ctx.length, 
ctx.type, ctx.lastModified, ctx.etag,
ctx.method, ctx.url, ctx.path, ctx.query, 
ctx.querystring, ctx.search, ctx.accept, ctx.socket
```

### 6.3 调试技巧

如果遇到属性行为异常，可以检查：

```javascript
// 检查属性是自有属性还是委托属性
console.log(ctx.hasOwnProperty('status'));  // false - 委托属性
console.log(ctx.hasOwnProperty('ip'));      // false - 委托属性

// 检查是否意外创建了自有属性
ctx.ip = '127.0.0.1';
console.log(ctx.hasOwnProperty('ip'));      // true - 现在是自有属性！

// 查看原型链上的属性描述符
const proto = Object.getPrototypeOf(ctx);
console.log(Object.getOwnPropertyDescriptor(proto, 'ip'));
// { get: [Function], set: undefined, enumerable: true, configurable: true }
```

### 6.4 扩展 Context 时的注意事项

如果需要在应用中扩展 Context，避免与委托属性冲突：

```javascript
// ✅ 安全的扩展方式
app.context.myCustomProp = 'value';  // 添加到原型

// 或者在中间件中
app.use(async (ctx, next) => {
  // 使用 Symbol 避免命名冲突
  const MY_DATA = Symbol('myData');
  ctx[MY_DATA] = { ... };
  
  await next();
});

// ⚠️ 注意：不要覆盖委托的属性名
// app.context.status = ...  // 这会破坏委托机制！
```

## 7. 总结

### 7.1 核心要点

1. **委托机制**：Koa 使用 `delegates` 库将 Context 的属性和方法委托给内部的 Request 和 Response 对象。

2. **三种委托类型**：
   - `method()`：委托方法调用
   - `access()`：双向委托 getter 和 setter（对称）
   - `getter()`：只委托 getter（不对称，只读）

3. **方向对称性**：
   - `access` 委托的属性读写完全对称
   - `getter` 委托的属性只有读操作被委托，写操作会在 Context 上创建新属性，不会委托到底层

4. **`ctx.status` vs 直接操作**：
   - `ctx.status` 提供完整的验证、状态管理和副作用处理
   - 直接操作 `ctx.res.statusCode` 跳过所有安全检查和逻辑，可能导致难以调试的问题

### 7.2 设计哲学

Koa 的委托机制体现了以下设计哲学：

1. **关注点分离**：Request 负责请求处理，Response 负责响应处理，Context 作为统一入口。

2. **便捷性与安全性平衡**：
   - 通过委托提供便捷的 API（`ctx.status` 而非 `ctx.response.status`）
   - 在底层实现中添加严格的验证和逻辑，确保安全性

3. **渐进式暴露**：
   - 常用操作通过 Context 直接暴露
   - 高级操作仍可通过 `ctx.request`、`ctx.response`、`ctx.req`、`ctx.res` 访问底层对象

### 7.3 关键警示

⚠️ **重要警示**：

1. **不要对 `getter` 委托的属性赋值**：虽然不会报错，但会创建自有属性遮蔽委托，导致难以追踪的 bug。

2. **除非必要，不要直接操作 `ctx.req` 和 `ctx.res`**：Koa 的 Request/Response 封装了大量安全逻辑和便捷方法，直接操作底层对象会绕过这些保护。

3. **理解委托链**：`ctx.status` → `ctx.response.status` → `ctx.res.statusCode`，每一层都有其存在的意义。

## 8. 参考资料

- [Koa 源码](https://github.com/koajs/koa)
- [delegates 库](https://www.npmjs.com/package/delegates)
- [Node.js HTTP 文档](https://nodejs.org/api/http.html)

---

*分析基于 Koa 3.2.0 版本源码*
