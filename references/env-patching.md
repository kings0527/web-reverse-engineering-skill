# 环境修补实战：从Proxy监测到知乎__zse_ck完整复现

> 本文提供Node.js环境修补的完整代码模板，包括Proxy监测法、BOM/DOM伪造清单、反检测修补、以及知乎__zse_ck的端到端实战案例。

## 1. Proxy监测法完整代码模板

Proxy监测法的核心思想是用递归Proxy包裹空对象，自动捕获目标代码所有属性访问，记录缺失的API并逐步补全：

```javascript
/**
 * 递归Proxy监测器：包裹目标对象，记录所有属性访问和缺失API
 * 用法：const monitoredGlobal = createMonitoringProxy({}, 'globalThis');
 */
function createMonitoringProxy(target, name = 'globalThis') {
    const accessed = new Set();
    const missing = new Set();

    return new Proxy(target, {
        get(obj, prop) {
            // Symbol属性特殊处理
            if (typeof prop === 'symbol') return obj[prop];

            const key = `${name}.${String(prop)}`;
            if (!accessed.has(key)) {
                accessed.add(key);
                console.log(`[ENV-ACCESS] ${key}`);
            }

            if (prop in obj) {
                const val = obj[prop];
                // 递归代理：返回的对象也包装Proxy，实现深层监测
                if (typeof val === 'object' && val !== null && !val.__isProxy) {
                    const subProxy = createMonitoringProxy(val, key);
                    Object.defineProperty(subProxy, '__isProxy', { value: true });
                    return subProxy;
                }
                // 函数绑定：确保this指向正确
                if (typeof val === 'function') {
                    return val.bind(obj);
                }
                return val;
            }

            // 未定义的属性 → 记录并返回空函数/undefined
            if (!missing.has(key)) {
                missing.add(key);
                console.log(`[ENV-MISS] ${key} → needs implementation`);
            }
            // 返回空函数（大多数情况下目标代码在调用方法）
            return function() { return undefined; };
        },

        has(obj, prop) {
            // 让 'prop' in obj 始终返回true，避免in检测失败
            return true;
        },

        // 拦截Object.keys等枚举操作
        ownKeys(obj) {
            return Reflect.ownKeys(obj);
        },

        // 拦截Object.getOwnPropertyDescriptor
        getOwnPropertyDescriptor(obj, prop) {
            if (prop in obj) {
                return Object.getOwnPropertyDescriptor(obj, prop);
            }
            // 对缺失属性返回一个看起来自然的描述符
            return { value: undefined, writable: true, enumerable: true, configurable: true };
        }
    });
}

// 使用流程：
// 1. 创建监测代理：const env = createMonitoringProxy(baseEnv, 'globalThis');
// 2. 运行目标代码：vm.runInContext(targetCode, env);
// 3. 查看日志：[ENV-MISS] globalThis.document.cookie → 需要实现
// 4. 补充缺失API → 重复步骤2-3直到无MISS
```

**迭代策略**：每次运行后查看`[ENV-MISS]`日志，补充对应API，然后重新运行。通常3-5轮迭代后就能覆盖所有依赖。

## 2. BOM/DOM伪造清单

### 2.1 必需对象与属性

每个对象的必需属性和推荐伪造值：

```javascript
const baseEnv = {
    // === globalThis ===
    // 关键：Object.prototype.toString.call(window) 必须返回 "[object Window]"
    [Symbol.toStringTag]: 'Window',

    // === window（self-reference） ===
    get self() { return this; },
    get top() { return this; },
    get parent() { return this; },
    frames: {},
    length: 0,
    name: '',
    closed: false,
    atob: (str) => Buffer.from(str, 'base64').toString('binary'),
    btoa: (str) => Buffer.from(str, 'binary').toString('base64'),

    // === document ===
    document: {
        cookie: '',           // 初始cookie（从预请求获取）
        title: '',
        readyState: 'complete',
        domain: 'www.zhihu.com',
        referrer: '',
        createElement: function(tag) {
            return {
                tagName: tag.toUpperCase(),
                style: {},
                setAttribute: function(k, v) { this[k] = v; },
                getAttribute: function(k) { return this[k] || null; },
                appendChild: function() {},
                removeChild: function() {},
                addEventListener: function() {},
                childNodes: [],
                firstChild: null,
                ownerDocument: null, // 避免循环引用，运行时绑定
            };
        },
        getElementById: function() { return null; },
        querySelector: function() { return null; },
        querySelectorAll: function() { return []; },
        head: { appendChild: function() {} },
        body: { appendChild: function() {} },
    },

    // === navigator ===
    navigator: {
        userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...',
        platform: 'Win32',
        language: 'zh-CN',
        languages: ['zh-CN', 'zh', 'en'],
        hardwareConcurrency: 8,
        maxTouchPoints: 0,
        webdriver: false,     // 关键：必须为false或undefined
        plugins: { length: 0 },
        mimeTypes: { length: 0 },
        cookieEnabled: true,
        onLine: true,
    },

    // === location ===
    location: {
        href: 'https://www.zhihu.com/',
        hostname: 'www.zhihu.com',
        pathname: '/',
        protocol: 'https:',
        port: '',
        search: '',
        hash: '',
        origin: 'https://www.zhihu.com',
    },

    // === screen ===
    screen: {
        width: 1920, height: 1080,
        availWidth: 1920, availHeight: 1040,
        colorDepth: 24, pixelDepth: 24,
    },

    // === history ===
    history: { length: 2, state: null, pushState: function(){}, replaceState: function(){} },

    // === storage ===
    localStorage: createStorageMock(),
    sessionStorage: createStorageMock(),
};

function createStorageMock() {
    const store = new Map();
    return {
        getItem: (k) => store.get(k) ?? null,
        setItem: (k, v) => store.set(k, String(v)),
        removeItem: (k) => store.delete(k),
        clear: () => store.clear(),
        get length() { return store.size; },
    };
}
```

### 2.2 常见遗漏属性

以下属性经常被忽略但会导致签名计算错误：

| 属性 | 说明 | 常见问题 |
|------|------|---------|
| `window.chrome` | Chrome浏览器特有对象 | `if (window.chrome)` 检测 |
| `document.all` | typeof为"undefined"但truthy | 特殊浏览器行为检测 |
| `window.performance` | 性能计时API | 时间戳计算依赖 |
| `navigator.connection` | 网络信息API | 部分签名包含网络类型 |
| `window.crypto` | Web Crypto API | 随机数生成依赖 |

## 3. 反检测代码模板

### 3.1 Function.toString修补

反爬系统会检查伪造函数的`toString()`是否返回`"[native code]"`：

```javascript
// 收集所有伪造的函数引用
const mockedFunctions = new WeakSet();

function createNativeLikeFunction(fn, name) {
    const wrapped = function(...args) { return fn.apply(this, args); };
    Object.defineProperty(wrapped, 'name', { value: name });
    Object.defineProperty(wrapped, 'toString', {
        value: function() { return `function ${name}() { [native code] }`; }
    });
    mockedFunctions.add(wrapped);
    return wrapped;
}

// 修补Function.prototype.toString（全局层面）
const originalToString = Function.prototype.toString;
Function.prototype.toString = function() {
    if (this === Function.prototype.toString) {
        return 'function toString() { [native code] }';
    }
    if (mockedFunctions.has(this)) {
        return `function ${this.name || ''}() { [native code] }`;
    }
    return originalToString.call(this);
};
// toString本身也需要修补
mockedFunctions.add(Function.prototype.toString);
```

### 3.2 getOwnPropertyDescriptor修补

确保属性描述符看起来自然（特别是`navigator.webdriver`）：

```javascript
// navigator.webdriver的描述符必须与真实浏览器一致
Object.defineProperty(navigator, 'webdriver', {
    get: () => false,
    enumerable: true,        // 真实浏览器中是enumerable
    configurable: true,
});

// 验证：Object.getOwnPropertyDescriptor(navigator, 'webdriver')
// 应该返回 { get: [Function], set: undefined, enumerable: true, configurable: true }
```

### 3.3 document.all特殊行为模拟

```javascript
// document.all的特殊行为：typeof为"undefined"但值为truthy
// 这是JavaScript中唯一具有此行为的值
// 使用Symbol.toPrimitive无法完全模拟，但可以用exotic object近似
const documentAll = new Proxy(function(){}, {
    get(target, prop) {
        if (prop === Symbol.toPrimitive) return () => undefined;
        if (prop === Symbol.toStringTag) return undefined;
        return undefined;
    },
    has() { return true; },
});
// 注意：typeof documentAll 返回 "function" 而非 "undefined"
// 完美模拟需要V8内部支持，大多数反爬不会检测这么深
```

## 4. 知乎__zse_ck实战案例

### 4.1 完整流程概览

```
预请求(收集cookie) → GET challenge页面 → 下载emo.js
→ Node.js子进程运行emo.js → 生成__zse_ck cookie → 带cookie请求目标
```

### 4.2 Step-by-Step实现

**Step 1：预请求收集初始cookie**

```bash
# 获取初始Set-Cookie
curl -c cookies.txt -D headers.txt "https://www.zhihu.com"
# cookies.txt中可能包含：_zap, d_c0, KDSUK-B等初始cookie
```

**Step 2：获取challenge信息和emo.js URL**

```bash
# 请求captcha接口获取challenge
curl -b cookies.txt "https://www.zhihu.com/api/v4/oauth/captcha_v2"
# 响应中包含：_xsrf token 和 emo.js的URL
# 提取emo.js URL（通常在HTML的script标签中）
```

**Step 3：下载emo.js**

```bash
curl -o emo.js "https://static.zhihu.com/heifetz/emo.js"
# emo.js包含__zse_ck的生成逻辑，文件可能经过混淆
```

**Step 4：Node.js子进程运行环境模拟+emo.js**

```javascript
const vm = require('vm');
const fs = require('fs');

const emoJsCode = fs.readFileSync('emo.js', 'utf8');
const initialCookies = '_zap=xxx; d_c0=xxx'; // 从Step 1获取

// 构建最小环境（使用Proxy监测法迭代补全后的结果）
const env = {
    globalThis: {},
    window: {},
    document: {
        cookie: initialCookies,
        createElement: () => ({
            style: {}, setAttribute: () => {}, getAttribute: () => null,
            appendChild: () => {}, addEventListener: () => {},
        }),
        getElementById: () => null,
        querySelector: () => null,
        readyState: 'complete',
        title: '',
    },
    navigator: {
        userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...',
        platform: 'Win32', language: 'zh-CN',
        webdriver: false,
    },
    location: { href: 'https://www.zhihu.com/', hostname: 'www.zhihu.com' },
    screen: { width: 1920, height: 1080, colorDepth: 24 },
    setTimeout: setTimeout,
    setInterval: setInterval,
    clearTimeout: clearTimeout,
    clearInterval: clearInterval,
    atob: (s) => Buffer.from(s, 'base64').toString('binary'),
    btoa: (s) => Buffer.from(s, 'binary').toString('base64'),
    parseInt, parseFloat, isNaN, isFinite,
    encodeURIComponent, decodeURIComponent,
    Array, Object, String, Number, Boolean, Date, Math, JSON,
    RegExp, Error, TypeError, RangeError,
    Uint8Array, Uint32Array, ArrayBuffer,
    Map, Set, WeakMap, WeakSet, Promise, Symbol,
};
// 自引用
env.window = env.globalThis = env;

vm.createContext(env);
vm.runInContext(emoJsCode, env);

// 提取生成的__zse_ck
const zseCk = env.document.cookie.match(/__zse_ck=([^;]+)/)?.[1];
console.log('__zse_ck:', zseCk);
```

**Step 5：带完整cookie请求目标**

```bash
# 将__zse_ck添加到cookie链
curl -b "cookies.txt; __zse_ck=$ZSE_CK" \
     "https://www.zhihu.com/api/v4/answers/123456789"
```

### 4.3 Chrome DevTools深度调试辅助分析

当emo.js混淆程度高、静态分析困难时，使用Chrome DevTools LogPoint + 条件断点进行动态追踪：

```javascript
// 在Chrome DevTools Console中注入追踪Hook
// 追踪所有包含zse/cookie/encrypt关键词的函数调用
const originalCall = Function.prototype.call;
Function.prototype.call = function(...args) {
  const name = this.name || '';
  if (/zse|cookie|encrypt/i.test(name)) {
    console.log(`[TRACE] ${name}(`, args.slice(1), `)`);
  }
  return originalCall.apply(this, args);
};
```

通过LogPoint可以在不暂停执行的情况下记录关键函数的参数和返回值，帮助定位__zse_ck的生成入口。

## 5. 环境修补常见陷阱

| 陷阱 | 症状 | 解决方案 |
|------|------|---------|
| `setTimeout`回调不执行 | 代码运行后无cookie输出 | 用Node.js原生setTimeout或手动触发队列 |
| `Date.now()`与`performance.now()`不一致 | 时间校验失败 | 统一时间源（见skill文档5.5节） |
| `toString`检测失败 | "function not native"错误 | 修补Function.prototype.toString |
| `document.cookie`读写为空 | cookie设置了HttpOnly | 检查domain/path匹配 |
| `window.chrome`未定义 | "missing chrome object" | 伪造`chrome = { runtime: {}, loadTimes: ()=>{}, csi: ()=>{} }` |
| Canvas指纹不一致 | 每次运行签名不同 | 使用固定种子随机数 |
| 闭包变量被共享 | 多次调用结果相同 | 确保每次运行创建新的VM context |
