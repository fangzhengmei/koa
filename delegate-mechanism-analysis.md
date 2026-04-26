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

Koa 使用了 [delegates](https://www.npmjs.com/package/delegates) 库（版本 1.0.0）来实现委托机制。该库由 TJ Holowaychuk 开发，提供了优雅的 API 来在原型对象之间构建方法、getter、setter 的委托。

### 3.2 delegates 库完整源码分析

以下是 delegates 1.0.0 的完整源码（来自 [unpkg.com](https://unpkg.com/delegates@1.0.0/index.js)）：

```javascript
/**
 * Expose `Delegator`.
 */

module.exports = Delegator;

/**
 * Initialize a delegator.
 *
 * @param {Object} proto
 * @param {String} target
 * @api public
 */

function Delegator(proto, target) {
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  this.proto = proto;
  this.target = target;
  this.methods = [];
  this.getters = [];
  this.setters = [];
  this.fluents = [];
}

/**
 * Delegate method `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.method = function(name){
  var proto = this.proto;
  var target = this.target;
  this.methods.push(name);

  proto[name] = function(){
    return this[target][name].apply(this[target], arguments);
  };

  return this;
};

/**
 * Delegator accessor `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.access = function(name){
  return this.getter(name).setter(name);
};

/**
 * Delegator getter `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.getter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);

  proto.__defineGetter__(name, function(){
    return this[target][name];
  });

  return this;
};

/**
 * Delegator setter `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.setter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.setters.push(name);

  proto.__defineSetter__(name, function(val){
    return this[target][name] = val;
  });

  return this;
};

/**
 * Delegator fluent accessor
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.fluent = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.fluents.push(name);

  proto[name] = function(val){
    if ('undefined' != typeof val) {
      this[target][name] = val;
      return this;
    } else {
      return this[target][name];
    }
  };

  return this;
};
```

### 3.3 Context 中的委托配置

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

### 3.4 委托类型详解

delegates 库提供了四种委托方式：

| 委托方式 | 说明 | 示例 |
|---------|------|------|
| `method(name)` | 委托方法调用 | `ctx.redirect('/home')` → `ctx.response.redirect('/home')` |
| `access(name)` | 同时委托 getter 和 setter（双向） | `ctx.status = 200` → `ctx.response.status = 200` |
| `getter(name)` | 只委托 getter（只读） | `ctx.ip` → 读取 `ctx.request.ip`，不可写入 |
| `fluent(name)` | 流畅 API 委托（getter/setter 二合一，支持链式） | Koa 中未使用 |

### 3.5 委托实现原理（真实源码分析）

#### 3.5.1 Delegator 构造函数

```javascript
function Delegator(proto, target) {
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  this.proto = proto;        // 要添加委托的原型对象（如 Context.prototype）
  this.target = target;      // 目标属性名（如 'request' 或 'response'）
  this.methods = [];         // 跟踪已委托的方法名
  this.getters = [];         // 跟踪已委托的 getter
  this.setters = [];         // 跟踪已委托的 setter
  this.fluents = [];         // 跟踪已委托的 fluent 属性
}
```

**关键点**：
- 支持**工厂模式调用**：不使用 `new` 关键字也能创建实例
- `proto`：要添加委托属性的原型对象（在 Koa 中是 Context 的 prototype）
- `target`：目标属性名（在 Koa 中是 `'request'` 或 `'response'`）

#### 3.5.2 `method` 委托 - 方法委托

**真实实现**：
```javascript
Delegator.prototype.method = function(name){
  var proto = this.proto;
  var target = this.target;
  this.methods.push(name);  // 记录到 methods 数组

  // 在原型上创建一个包装函数
  proto[name] = function(){
    // 关键：使用 apply 绑定 this 到目标对象
    return this[target][name].apply(this[target], arguments);
  };

  return this;  // 支持链式调用
};
```

**与伪代码的差异**：

| 方面 | 伪代码示意 | 真实实现 |
|------|-----------|---------|
| 函数定义 | `function(...args)` | `function()` 使用 `arguments` |
| this 绑定 | 隐式绑定 | 显式使用 `apply(this[target], arguments)` |
| 参数传递 | 扩展运算符 | `arguments` 对象 |

**关键技术点 - `apply(this[target], arguments)`**：

这是 `method` 委托最重要的设计。让我们分析为什么这样做：

```javascript
// 当调用 ctx.redirect('/home') 时：
// 1. this = ctx（Context 实例）
// 2. this[target] = ctx['response'] = ctx.response
// 3. this[target][name] = ctx.response.redirect
// 4. apply(ctx.response, ['/home']) 执行函数

// 效果等价于：
ctx.response.redirect.apply(ctx.response, ['/home']);
// 即：
ctx.response.redirect('/home');
```

**为什么要用 `apply` 绑定 `this`？**

考虑 `response.js` 中的 `redirect` 方法：

```javascript
// lib/response.js:302-323
redirect (url) {
  // 方法内部使用了 this
  if (/^https?:\/\//i.test(url)) {
    url = new URL(url).toString()
  }
  this.set('Location', encodeUrl(url))  // 这里的 this 必须是 Response 对象
  
  if (!statuses.redirect[this.status]) this.status = 302  // 这里也是
  
  if (this.ctx.accepts('html')) {  // 这里也是
    // ...
  }
}
```

如果不使用 `apply` 绑定 `this`：

```javascript
// 假设实现是这样的：
proto[name] = function(...args) {
  return this[target][name](...args);
}

// 当调用 ctx.redirect('/home') 时：
// this[target][name] = ctx.response.redirect
// 直接调用 ctx.response.redirect(...args)
// 此时 redirect 内部的 this 是什么？

// 在 JavaScript 中，函数的 this 取决于调用方式：
// obj.method()  -> this = obj
// method()      -> this = undefined (严格模式) 或 global

// 但这里是：
// var fn = ctx.response.redirect;
// fn(...args);  // 这是一次独立的函数调用！
// 此时 this 不是 ctx.response，而是 undefined 或 global！
```

**`apply` 的作用就是确保方法内部的 `this` 正确指向目标对象**（如 `ctx.response`）。

#### 3.5.3 `access` 委托 - 双向访问器委托

**真实实现**：
```javascript
Delegator.prototype.access = function(name){
  return this.getter(name).setter(name);
};
```

**分析**：
- `access` 只是简单地**链式调用** `getter` 和 `setter`
- 利用了 `getter()` 和 `setter()` 都返回 `this` 的特性
- 因此 `access(name)` 等价于同时配置 getter 和 setter

#### 3.5.4 `getter` 委托 - 只读属性委托

**真实实现**：
```javascript
Delegator.prototype.getter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);  // 记录到 getters 数组

  // ⚠️ 注意：使用的是 __defineGetter__，不是 Object.defineProperty
  proto.__defineGetter__(name, function(){
    return this[target][name];
  });

  return this;  // 支持链式调用
};
```

**与伪代码的重大差异**：

| 方面 | 伪代码示意 | 真实实现 |
|------|-----------|---------|
| API | `Object.defineProperty` | `__defineGetter__` |
| 标准性 | ES5 标准 API | **非标准，已废弃** |
| 兼容性 | 现代环境都支持 | 虽已废弃但仍被支持 |

**关于 `__defineGetter__`**：

```javascript
// __defineGetter__ 是 Object.prototype 上的方法，语法：
obj.__defineGetter__(propName, getterFunction);

// 等价于（但不完全相同）：
Object.defineProperty(obj, propName, {
  get: getterFunction,
  configurable: true,
  enumerable: true
});
```

**为什么说 `__defineGetter__` 已废弃？**

根据 [MDN 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__)：

> `__defineGetter__` 方法可以将一个函数绑定在当前对象的指定属性上，当那个属性被读取时，那个绑定的函数就会被调用。
> 
> **该特性是非标准的**，请尽量不要在生产环境中使用它！
> 
> 该方法已被弃用，建议使用 `Object.defineProperty` 方法来定义 getter。

**Koa 使用 `__defineGetter__` 的影响**：

虽然 `__defineGetter__` 已废弃，但：
1. 现代浏览器和 Node.js 仍**完全支持**（为了向后兼容）
2. 功能上与 `Object.defineProperty` 的 getter 基本一致
3. 但这是一个**技术债务**，未来可能被移除

#### 3.5.5 `setter` 委托 - 只写属性委托

**真实实现**：
```javascript
Delegator.prototype.setter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.setters.push(name);  // 记录到 setters 数组

  // ⚠️ 注意：使用的是 __defineSetter__，不是 Object.defineProperty
  proto.__defineSetter__(name, function(val){
    return this[target][name] = val;
  });

  return this;  // 支持链式调用
};
```

**关键点**：
- 同样使用**已废弃的 `__defineSetter__`**
- setter 函数返回赋值表达式的值（`return this[target][name] = val`）
- 这符合 JavaScript 赋值表达式的行为（赋值表达式返回被赋的值）

#### 3.5.6 `fluent` 委托 - 流畅 API 委托

**真实实现**：
```javascript
Delegator.prototype.fluent = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.fluents.push(name);  // 记录到 fluents 数组

  proto[name] = function(val){
    if ('undefined' != typeof val) {
      // Setter 模式：设置值并返回 this（支持链式调用）
      this[target][name] = val;
      return this;  // ← 链式调用的关键！
    } else {
      // Getter 模式：无参数时返回值
      return this[target][name];
    }
  };

  return this;  // 支持配置阶段的链式调用
};
```

**Koa 中未使用 `fluent` 委托**，但让我们理解其工作原理：

```javascript
// 假设有这样的配置：
delegate(obj, 'settings').fluent('env');

// 使用方式：
obj.env();           // Getter 模式：返回 obj.settings.env
obj.env('production'); // Setter 模式：设置并返回 obj（支持链式）

// 链式调用示例：
obj
  .env('production')
  .env('test')  // 可以继续链式设置
  .env();        // 最后读取，返回 'test'
```

**与 `access` 的区别**：

| 特性 | `access` | `fluent` |
|------|----------|----------|
| 实现方式 | `__defineGetter__` + `__defineSetter__` | 单个函数判断参数 |
| Getter 调用 | `obj.prop` | `obj.prop()` |
| Setter 调用 | `obj.prop = value` | `obj.prop(value)` |
| 链式支持 | 不支持 | Setter 后返回 `this`，支持 |
| Koa 中使用 | ✅ 大量使用 | ❌ 未使用 |

### 3.6 链式调用的实现原理

所有委托方法（`method`、`getter`、`setter`、`access`、`fluent`）都返回 `this`，这使得流畅的链式 API 成为可能：

```javascript
// 这样的链式调用：
delegate(proto, 'response')
  .method('redirect')
  .method('set')
  .access('status')
  .access('body')
  .getter('headerSent');

// 等价于（但更易读）：
var d = delegate(proto, 'response');
d.method('redirect');
d.method('set');
d.access('status');
d.access('body');
d.getter('headerSent');
```

**链式调用的执行流程**：

```
1. delegate(proto, 'response') 
   → 返回 Delegator 实例 { proto, target: 'response', methods: [], ... }

2. .method('redirect')
   → 在 proto 上定义 redirect 函数
   → this.methods.push('redirect')
   → 返回 this（同一个 Delegator 实例）

3. .access('status')
   → 调用 this.getter('status').setter('status')
   → 在 proto 上定义 status 的 getter 和 setter
   → this.getters.push('status'), this.setters.push('status')
   → 返回 this（同一个 Delegator 实例）

4. .getter('headerSent')
   → 在 proto 上定义 headerSent 的 getter
   → this.getters.push('headerSent')
   → 返回 this（同一个 Delegator 实例）
```

## 4. Getter 和 Setter 的方向对称性分析

### 4.1 对称性定义

在委托机制中，**方向对称性**指的是：
- **Getter 方向**：`ctx.prop` → `ctx.request.prop` 或 `ctx.response.prop`
- **Setter 方向**：`ctx.prop = value` → `ctx.request.prop = value` 或 `ctx.response.prop = value`

### 4.2 不同委托类型的对称性

#### 4.2.1 `access` 委托 - 完全对称

使用 `access` 委托的属性，getter 和 setter 是完全对称的：

**执行流程分析**：

```
读取 ctx.status：
┌─────────────────────────────────────────────────────────────┐
│  1. ctx.status (属性访问)                                     │
│     ↓                                                         │
│  2. 触发原型上的 __defineGetter__ 定义的 getter 函数          │
│     ↓                                                         │
│  3. getter 函数执行：return this['response']['status']        │
│     ↓                                                         │
│  4. this['response'] = ctx.response                           │
│     ↓                                                         │
│  5. 访问 ctx.response.status（Response.prototype 上的 getter）  │
│     ↓                                                         │
│  6. Response getter 执行：return this.res.statusCode          │
│     ↓                                                         │
│  7. 返回 ctx.res.statusCode（Node.js 原始响应对象的状态码）    │
└─────────────────────────────────────────────────────────────┘

写入 ctx.status = 200：
┌─────────────────────────────────────────────────────────────┐
│  1. ctx.status = 200 (赋值操作)                               │
│     ↓                                                         │
│  2. 触发原型上的 __defineSetter__ 定义的 setter 函数          │
│     ↓                                                         │
│  3. setter 函数执行：return this['response']['status'] = 200  │
│     ↓                                                         │
│  4. this['response'] = ctx.response                           │
│     ↓                                                         │
│  5. 赋值 ctx.response.status = 200                            │
│     ↓                                                         │
│  6. 触发 Response.prototype 上的 setter                        │
│     ↓                                                         │
│  7. Response setter 执行完整逻辑（验证、设置 _explicitStatus 等）│
└─────────────────────────────────────────────────────────────┘
```

**示例验证**：
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

**执行流程分析**：

```
读取 ctx.ip：
┌─────────────────────────────────────────────────────────────┐
│  1. ctx.ip (属性访问)                                         │
│     ↓                                                         │
│  2. 触发原型上的 __defineGetter__ 定义的 getter 函数          │
│     ↓                                                         │
│  3. getter 函数执行：return this['request']['ip']             │
│     ↓                                                         │
│  4. 返回 ctx.request.ip（Request 对象计算得到的 IP）           │
└─────────────────────────────────────────────────────────────┘

写入 ctx.ip = '127.0.0.1'：
┌─────────────────────────────────────────────────────────────┐
│  1. ctx.ip = '127.0.0.1' (赋值操作)                          │
│     ↓                                                         │
│  2. 查找原型链上是否有 setter：                                │
│     - 只有 __defineGetter__，没有 __defineSetter__           │
│     - Object.getOwnPropertyDescriptor 返回 { set: undefined } │
│     ↓                                                         │
│  3. JavaScript 引擎行为：                                      │
│     - 如果原型上有 getter 但没有 setter                       │
│     - 在严格模式下：抛出 TypeError                             │
│     - 在非严格模式下：在实例上创建自有属性，不触发 setter      │
│     ↓                                                         │
│  4. 结果：                                                     │
│     - ctx 上新增自有属性 ip = '127.0.0.1'                    │
│     - 原型上的 getter 仍然存在                                │
│     - 下次读取 ctx.ip 时，原型 getter 优先级更高？              │
│       实际上：自有属性会遮蔽原型属性？                         │
│       需要验证！                                               │
└─────────────────────────────────────────────────────────────┘
```

**关于属性遮蔽的关键问题**：

当原型上有 getter，而实例上创建了同名自有属性时，会发生什么？

```javascript
// 测试代码
const proto = {};
proto.__defineGetter__('ip', function() {
  return 'from-prototype';
});

const obj = Object.create(proto);

console.log(obj.ip);  // 'from-prototype'

// 尝试赋值
obj.ip = 'from-instance';

console.log(obj.ip);  // 输出什么？
```

**答案**：

在 JavaScript 中：
- 如果原型上的属性是 **数据属性**（不是 getter/setter），实例的自有属性会**遮蔽**原型属性
- 如果原型上的属性是 **访问器属性**（只有 getter，没有 setter）：
  - **严格模式**：赋值会抛出 `TypeError: Cannot set property ip of #<Object> which has only a getter`
  - **非严格模式**：赋值会被**静默忽略**，不会创建自有属性

**验证**：
```javascript
// 非严格模式
'use strict';  // 注释掉这行进入非严格模式

const proto = {};
proto.__defineGetter__('ip', function() {
  return 'from-prototype';
});

const obj = Object.create(proto);

console.log(obj.ip);  // 'from-prototype'

// 非严格模式下：
obj.ip = 'from-instance';  // 静默忽略

console.log(obj.ip);  // 仍然是 'from-prototype'！
console.log(obj.hasOwnProperty('ip'));  // false，没有创建自有属性

// 严格模式下：
// obj.ip = 'from-instance';  // 抛出 TypeError！
```

**Koa 中的实际行为**：

Koa 使用 CommonJS 模块，Node.js 默认在非严格模式下执行模块代码。但 `__defineGetter__` 创建的访问器属性在赋值时的行为：

根据 delegates 库的源码注释和实际使用情况，Koa 开发者应该知道 `getter` 委托的属性是只读的，不会尝试去修改它们。

#### 4.2.3 `setter` 委托 - 不对称（只写）

Koa 中**没有单独使用 `setter` 委托**的属性，所有可写属性都通过 `access` 委托（同时有 getter 和 setter）。

但让我们理解单独 `setter` 委托的行为：

```javascript
// 假设有这样的配置（Koa 中没有）
delegate(proto, 'response').setter('body');

// 行为：
ctx.body = 'Hello';  // ✅ 正常工作，委托到 ctx.response.body
const body = ctx.body;  // ❌ 不工作！因为只有 setter，没有 getter
```

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
  ctx.ip = '192.168.1.1';  // 非严格模式下静默忽略
  
  // 下次读取
  console.log(ctx.ip);  // 仍返回真实的客户端 IP，不是 '192.168.1.1'
  
  // 检查自有属性
  console.log(ctx.hasOwnProperty('ip'));  // false，没有创建自有属性
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

## 6. delegates 库技术债务分析

### 6.1 使用已废弃 API 的风险

delegates 1.0.0 使用了 `__defineGetter__` 和 `__defineSetter__`，这两个 API 存在以下问题：

| 问题 | 说明 |
|------|------|
| 非标准 | 不属于 ECMAScript 标准 |
| 已废弃 | MDN 明确标记为 deprecated |
| 未来风险 | 未来的 JavaScript 引擎可能移除支持 |
| 功能限制 | 相比 `Object.defineProperty`，可配置性更低 |

### 6.2 与 `Object.defineProperty` 的对比

```javascript
// delegates 当前使用的方式（已废弃）
proto.__defineGetter__(name, function(){
  return this[target][name];
});

proto.__defineSetter__(name, function(val){
  return this[target][name] = val;
});

// 推荐的标准方式
Object.defineProperty(proto, name, {
  get: function() {
    return this[target][name];
  },
  set: function(val) {
    this[target][name] = val;
  },
  configurable: true,
  enumerable: true
});
```

### 6.3 实际影响

目前（2024 年），`__defineGetter__` 和 `__defineSetter__` 在所有主流浏览器和 Node.js 版本中**仍然完全支持**。这主要是因为：

1. **向后兼容**：大量旧代码依赖这些 API
2. **实现成本低**：JavaScript 引擎维护这些 API 的成本不高

但长期来看，存在以下风险：
- 新的 JavaScript 环境可能选择不实现
- 严格模式下行为可能改变
- 调试工具可能给出警告

## 7. 最佳实践建议

### 7.1 始终通过 Context 委托访问

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

### 7.2 了解属性的可写性

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

### 7.3 调试技巧

如果遇到属性行为异常，可以检查：

```javascript
// 检查属性是自有属性还是委托属性
console.log(ctx.hasOwnProperty('status'));  // false - 委托属性
console.log(ctx.hasOwnProperty('ip'));      // false - 委托属性

// 查看原型链上的属性描述符
const proto = Object.getPrototypeOf(ctx);
console.log(Object.getOwnPropertyDescriptor(proto, 'ip'));
// { get: [Function], set: undefined, enumerable: true, configurable: true }
```

### 7.4 扩展 Context 时的注意事项

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

## 8. 总结

### 8.1 核心要点

1. **委托机制**：Koa 使用 `delegates` 库将 Context 的属性和方法委托给内部的 Request 和 Response 对象。

2. **delegates 库真实实现**：
   - 使用**已废弃的 `__defineGetter__` 和 `__defineSetter__`**，而非 `Object.defineProperty`
   - `method` 委托使用 `apply(this[target], arguments)` 确保 `this` 正确绑定
   - 所有方法返回 `this`，支持流畅的链式 API 设计
   - `access` 只是 `getter` + `setter` 的链式调用

3. **三种委托类型**：
   - `method()`：委托方法调用，正确绑定 `this`
   - `access()`：双向委托 getter 和 setter（对称）
   - `getter()`：只委托 getter（不对称，只读）

4. **方向对称性**：
   - `access` 委托的属性读写完全对称
   - `getter` 委托的属性只有读操作被委托，写操作在严格模式下会报错，非严格模式下静默忽略

5. **`ctx.status` vs 直接操作**：
   - `ctx.status` 提供完整的验证、状态管理和副作用处理
   - 直接操作 `ctx.res.statusCode` 跳过所有安全检查和逻辑，可能导致难以调试的问题

### 8.2 设计哲学

Koa 的委托机制体现了以下设计哲学：

1. **关注点分离**：Request 负责请求处理，Response 负责响应处理，Context 作为统一入口。

2. **便捷性与安全性平衡**：
   - 通过委托提供便捷的 API（`ctx.status` 而非 `ctx.response.status`）
   - 在底层实现中添加严格的验证和逻辑，确保安全性

3. **渐进式暴露**：
   - 常用操作通过 Context 直接暴露
   - 高级操作仍可通过 `ctx.request`、`ctx.response`、`ctx.req`、`ctx.res` 访问底层对象

### 8.3 关键警示

⚠️ **重要警示**：

1. **不要对 `getter` 委托的属性赋值**：严格模式下会报错，非严格模式下静默忽略。

2. **除非必要，不要直接操作 `ctx.req` 和 `ctx.res`**：Koa 的 Request/Response 封装了大量安全逻辑和便捷方法，直接操作底层对象会绕过这些保护。

3. **理解委托链**：`ctx.status` → `ctx.response.status` → `ctx.res.statusCode`，每一层都有其存在的意义。

4. **技术债务意识**：delegates 库使用了已废弃的 `__defineGetter__` 和 `__defineSetter__`，虽然目前仍能工作，但长期存在风险。

## 9. 参考资料

- [Koa 源码](https://github.com/koajs/koa)
- [delegates 库 (npm)](https://www.npmjs.com/package/delegates)
- [delegates 库源码 (unpkg)](https://unpkg.com/delegates@1.0.0/index.js)
- [Object.prototype.__defineGetter__ - MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__)
- [Node.js HTTP 文档](https://nodejs.org/api/http.html)

---

*分析基于 Koa 3.2.0 和 delegates 1.0.0 版本源码*

*更新日期：2026-04-26 - 替换伪代码为 delegates 库真实实现分析*
