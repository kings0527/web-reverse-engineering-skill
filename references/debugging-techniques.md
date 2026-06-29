# Chrome DevTools 高级调试技巧

> 浏览器调试是 Web 逆向的基本功。本文档超越基础断点，
> 覆盖条件断点、LogPoint、内存分析等高级技巧。

---

## 目录

- [一、断点进阶](#一断点进阶)
- [二、代码搜索与导航](#二代码搜索与导航)
- [三、Console高级用法](#三console高级用法)
- [四、性能分析在逆向中的应用](#四性能分析在逆向中的应用)
- [五、Memory Inspector](#五memory-inspector)
- [六、Network面板高级用法](#六network面板高级用法)
- [七、Source Map与格式化](#七source-map与格式化)
- [八、反调试对抗](#八反调试对抗)

---

## 一、断点进阶

### 1.1 条件断点（Conditional Breakpoint）

**操作路径**：Sources面板 → 行号右键 → `Add conditional breakpoint...`

条件断点只在表达式返回 `true` 时触发中断，适合追踪特定条件下的执行。

```javascript
// 条件表达式示例

// 当加密数据长度超过100时中断
x.length > 100

// 当请求URL包含特定路径时中断
url.includes('/api/sign')

// 当函数被调用超过10次时中断（利用全局计数器）
(window.__count = (window.__count || 0) + 1) > 10

// 当参数包含特定字段时中断
JSON.stringify(params).includes('token')

// 当时间戳在特定范围时中断（追踪时序问题）
Date.now() > 1700000000000
```

**实战应用**：在混淆后的加密函数中，参数类型多变，设置 `typeof arg === 'string' && arg.length === 32` 可以精确捕获传入32位密钥的时刻。

### 1.2 LogPoint（日志断点）

**操作路径**：Sources面板 → 行号右键 → `Add logpoint...`

LogPoint 不中断执行，只在Console输出日志。表达式会被eval，结果输出到console。

```javascript
// LogPoint表达式示例

// 记录函数调用参数
'encrypt called:', arguments[0], arguments[1]

// 记录调用次数
'Call #' + (window.__lp = (window.__lp || 0) + 1), 'args:', key, data

// 输出调用栈（不中断）
'Stack:', new Error().stack

// JSON格式化输出对象
'Request:', JSON.stringify({url, params, headers})
```

**性能优势**：比手动插入 `console.log` 零侵入，不修改源代码，不触发代码完整性校验。

**实战应用**：追踪JSVMP派发循环中的opcode序列：在解释器 `switch` 语句处设置 LogPoint `'OP:', opcode, 'stack:', stack.slice(-3)`，记录完整执行路径。

### 1.3 XHR/Fetch Breakpoint

**操作路径**：Sources面板 → 右侧 `XHR/fetch Breakpoints` → 点击 `+`

输入URL中包含的字符串，当匹配的请求发出时自动中断。

```
# 触发条件示例
/api/sign          → 包含 /api/sign 的请求
token              → URL中任何位置包含 token
encrypt            → 捕获加密相关接口
```

**实战应用**：
1. 输入签名接口路径 `/api/v2/sign`
2. 触发中断后，查看Call Stack
3. 从 `XMLHttpRequest.send` 向上回溯
4. 找到签名参数的生成位置

### 1.4 Event Listener Breakpoint

**操作路径**：Sources面板 → 右侧 `Event Listener Breakpoints` → 展开分类勾选

**常用分类**：
- **Mouse** → `mousedown`, `click`：追踪点击触发的逻辑
- **Keyboard** → `keydown`：捕获输入相关的加密
- **Timer** → `setTimeout`, `setInterval`：追踪定时刷新的token
- **Script** → `Script First Statement`：在每个脚本首行中断
- **XHR** → `readystatechange`：精确追踪请求状态变化

**实战应用**：某些网站在 `mousedown` 事件中收集鼠标轨迹作为指纹。勾选 Mouse → mousedown，操作页面后中断，在Call Stack中找到指纹收集的入口函数。

### 1.5 DOM Breakpoint

**操作路径**：Elements面板 → 右键目标元素 → `Break on...`

三种触发条件：
- **Subtree modifications**：子元素增删改时中断
- **Attribute modifications**：属性变化时中断
- **Node removal**：节点被删除时中断

**实战应用**：
1. 在Elements中找到 `<body>` 或可疑容器元素
2. 设置 `Subtree modifications` 断点
3. 等待页面动态插入反爬脚本时自动中断
4. 在Call Stack中定位插入逻辑

---

## 二、代码搜索与导航

### 2.1 全局搜索（Ctrl+Shift+F）

**操作路径**：DevTools → `Ctrl+Shift+F`（或 Sources面板顶部搜索框）

搜索所有已加载的JS源码，支持正则表达式。

```
# 搜索技巧

# 搜索加密关键词
encrypt|decrypt|cipher|crypto

# 搜索签名相关
sign|signature|hmac|md5|sha256

# 搜索特定参数名（来自Network面板的观察）
X-Sign|x-token|_signature

# 搜索可疑的编码操作
btoa|atob|encodeURIComponent|fromCharCode

# 搜索反调试关键词
debugger|devtool|console\.(log|warn|error)\s*=
```

**技巧**：先在Network面板观察请求Header中的签名参数名（如 `X-Bogus`），然后全局搜索该名称，定位生成逻辑。

### 2.2 Call Stack分析

断点触发后，右侧 `Call Stack` 面板显示完整调用链。

**阅读方法**：
- 最顶部是当前暂停位置
- 向下是调用者（越往下越接近入口）
- 灰色帧是异步边界（Promise、setTimeout等）

**异步调用栈**：
- DevTools默认启用 Async stack traces
- 可以穿透 `Promise.then`、`setTimeout`、`requestAnimationFrame`
- 设置：DevTools Settings → Experiments → 确保 async stack 已启用

**实战应用**：
1. 在 `XMLHttpRequest.prototype.send` 设置断点
2. 触发目标请求
3. 中断后查看 Call Stack
4. 跳过框架代码（如axios内部），找到业务代码中调用加密函数的位置
5. 点击对应栈帧直接跳转到源码

### 2.3 Scope变量检查

断点暂停时，右侧 `Scope` 面板显示当前作用域的所有变量。

**作用域层级**：
- **Local**：当前函数内的局部变量
- **Closure**：闭包捕获的外层变量（最重要！加密密钥常在这里）
- **Script**：模块级变量
- **Global**：window对象属性

**Watch表达式**：
- 右侧 `Watch` 面板 → 点击 `+` 添加表达式
- 每次断点触发时自动求值
- 适合监控：`JSON.stringify(params)`, `key.toString(16)`, `buffer.byteLength`

**实战应用**：在加密函数断点处，展开 Closure 作用域，常能找到：
- 加密密钥（`key`, `secret`, `salt`）
- 初始化向量（`iv`）
- 中间计算结果

---

## 三、Console高级用法

### 3.1 monitor / unmonitor

```javascript
// 监控函数调用（每次调用时输出函数名和参数）
monitor(window.encrypt)
// 输出示例：function encrypt called with arguments: "hello", "key123"

// 取消监控
unmonitor(window.encrypt)

// 监控对象方法
monitor(CryptoJS.AES.encrypt)
```

**实战应用**：不确定某个函数何时被调用，用 `monitor` 快速确认调用时机和参数。

### 3.2 debug / undebug

```javascript
// 在函数入口自动设置断点（等效于在源码中设断点）
debug(window.generateSign)
// 下次 generateSign 被调用时自动中断

// 取消
undebug(window.generateSign)

// 适用于混淆代码中难以手动定位的函数
debug(window._0x3f2a)
```

**实战应用**：通过全局搜索找到可疑函数名后，直接 `debug(fn)` 而无需在海量代码中找到精确位置。

### 3.3 getEventListeners

```javascript
// 获取元素上绑定的所有事件监听器
getEventListeners(document)
// 返回：{click: [...], mousedown: [...], keydown: [...]}

// 获取特定元素的监听器
getEventListeners(document.getElementById('loginBtn'))

// 查看具体handler的源码位置
getEventListeners(document).click[0].listener
// 点击输出的函数引用可以跳转到源码
```

**实战应用**：某些网站通过事件监听器收集行为指纹。用 `getEventListeners(document)` 列出所有全局监听器，分析哪些是指纹收集相关的。

### 3.4 copy()

```javascript
// 将对象复制到系统剪贴板
copy(document.cookie)
copy(JSON.stringify(largeObject, null, 2))
copy(performance.getEntries())

// 复制加密结果
copy(window.__hookResults)
```

**实战应用**：断点暂停时，在Console中 `copy(encryptedData)` 直接复制加密结果，粘贴到本地脚本验证。

### 3.5 queryObjects

```javascript
// 查找堆中所有特定类型的实例
queryObjects(CryptoKey)
// 返回所有 CryptoKey 实例

queryObjects(Map)
// 返回所有 Map 实例（可能包含缓存的密钥对）

queryObjects(Promise)
// 查看所有未resolve的Promise
```

**实战应用**：`queryObjects(CryptoKey)` 可以找到页面中所有加密密钥实例，即使它们存储在闭包中难以直接访问。

---

## 四、性能分析在逆向中的应用

### 4.1 Performance面板

**操作路径**：DevTools → Performance → 点击录制按钮 → 操作页面 → 停止录制

**火焰图阅读方法**：
- 横轴是时间
- 纵轴是调用深度（越深越具体）
- 宽度代表执行时间（越宽越耗时）
- 颜色区分脚本来源

**实战应用**：
1. 开始录制
2. 触发一次API请求（如点击登录）
3. 停止录制
4. 在火焰图中找到最"宽"（耗时最长）的函数调用
5. 这些通常就是加密/签名函数（计算密集型）
6. 点击直接跳转到源码

### 4.2 CPU Profile

**操作路径**：Performance面板 → 录制 → Bottom-Up / Call Tree 视图

```
# 按Self Time排序，找到最耗CPU的函数
# 混淆代码中，加密函数通常具有：
# - 高Self Time（大量数学运算）
# - 被调用次数少但单次耗时长
# - 名字通常被混淆为 _0x????
```

**实战应用**：在完全混淆的代码中，不知道哪个函数是加密入口：
1. 录制包含加密请求的操作
2. 按 Self Time 降序排列
3. 排除浏览器内部函数后，Top 3-5 的函数大概率是加密相关
4. 点击函数名跳转到源码，设置断点进一步分析

---

## 五、Memory Inspector

### 5.1 Heap Snapshot

**操作路径**：DevTools → Memory → Heap snapshot → Take snapshot

**分析方法**：
1. 按 `Retained Size` 降序排列
2. 展开大对象，查找包含密钥/证书的数据
3. 搜索特定构造函数名（如 `CryptoKey`, `ArrayBuffer`）

```
# 搜索技巧（在Heap Snapshot的Filter中）
CryptoKey          → 查找加密密钥实例
Uint8Array         → 查找字节数组（可能是密钥/IV）
Map                → 查找键值对集合（配置/缓存）
```

**实战应用**：
1. 在页面加载完成后抓取堆快照
2. 搜索 `CryptoKey` 或相关类名
3. 展开实例，查看 `algorithm`、`extractable`、`type` 属性
4. 通过 Retainers 查看是谁持有这个密钥 → 找到密钥的存储位置

### 5.2 Allocation Timeline

**操作路径**：Memory → Allocation instrumentation on timeline → Start

追踪内存分配过程，识别频繁创建的对象：
- 蓝色柱代表新分配的内存
- 操作页面触发加密时，观察新创建的对象
- 点击蓝色柱查看那个时刻创建了什么对象

**实战应用**：触发一次签名请求，在Timeline中找到对应时刻新创建的 `ArrayBuffer` / `Uint8Array`，它们很可能是加密过程的输入/输出缓冲区。

---

## 六、Network面板高级用法

### 6.1 请求发起者（Initiator）

**操作路径**：Network面板 → 点击请求 → `Initiator` 标签

显示请求的完整调用链（类似精简版Call Stack）：
- 展示从入口到最终发起请求的函数链
- 点击任意一行跳转到对应源码位置

**实战应用**：
1. 在Network面板找到携带签名的API请求
2. 查看 Initiator
3. 跳过框架代码（axios/fetch wrapper）
4. 找到业务代码中构造请求参数的位置 → 即签名生成位置

### 6.2 Copy as cURL / fetch

**操作路径**：Network面板 → 右键请求 → `Copy` → `Copy as cURL` / `Copy as fetch`

```bash
# 复制为cURL后，可以在终端直接重放
curl 'https://api.target.com/data?sign=abc123&t=1700000000' \
  -H 'Cookie: session=xyz' \
  -H 'X-Token: token123'

# 修改参数重放，验证签名逻辑
# 如果修改参数后请求失败 → 确认存在签名校验
# 如果修改sign后请求失败 → 确认sign是必要的
```

### 6.3 请求阻断（Block request URL）

**操作路径**：Network面板 → 右键请求 → `Block request URL` 或 `Block request domain`

也可以通过：Network面板底部 → `Request blocking` 标签 → 添加模式

```
# 阻断模式示例
*.js?anti-debug*     → 阻断反调试脚本
*fingerprint*        → 阻断指纹收集请求
cdn.target.com/*.js  → 阻断特定CDN的所有JS
```

**实战应用**：
1. 观察页面加载了多个JS文件
2. 逐一阻断，观察哪个文件被阻断后页面行为异常
3. 异常的那个文件大概率包含核心逻辑（反爬/加密）
4. 重点分析该文件

---

## 七、Source Map与格式化

### 7.1 Pretty Print

**操作路径**：Sources面板 → 打开压缩代码 → 点击左下角 `{}` 图标

格式化后：
- 代码自动缩进，每行一条语句
- 断点可以设在格式化后的代码中
- 变量名仍是混淆的，但结构清晰

**技巧**：格式化后使用 `Ctrl+G` 跳转到指定行号。

### 7.2 Source Map覆盖 & Local Override

**Workspace功能**（已弃用，推荐Override）：

**Override功能**（推荐）：
1. Sources面板 → `Overrides` 标签
2. 点击 `Select folder for overrides` 选择本地目录
3. 浏览器请求授权后，可以编辑任何远程JS文件
4. 修改保存后自动覆盖远程资源

**操作步骤**：
1. Network面板找到目标JS文件
2. 右键 → `Override content`（或在Sources中右键 → `Save for overrides`）
3. 在本地版本中添加 `console.log` 或修改逻辑
4. 刷新页面，浏览器加载本地修改版

**实战应用**：
```javascript
// 在混淆代码的关键位置注入日志
// 原始代码（格式化后）：
function _0x3f2a(_0x1b, _0x2c) {
    var _0x3d = _0x1b + _0x2c;
    return _0x4e(_0x3d);
}

// Override版本（添加日志）：
function _0x3f2a(_0x1b, _0x2c) {
    console.log('[DEBUG] _0x3f2a input:', _0x1b, _0x2c);
    var _0x3d = _0x1b + _0x2c;
    var result = _0x4e(_0x3d);
    console.log('[DEBUG] _0x3f2a output:', result);
    return result;
}
```

---

## 八、反调试对抗

### 8.1 常见反调试手段

**1. debugger语句陷阱**
```javascript
// 无限循环debugger
setInterval(function() { debugger; }, 100);

// 或隐藏在eval中
eval("debugger");
(function(){return false;})["constructor"]("debugger")();
```

**2. console.log覆写检测**
```javascript
// 检测console是否被Hook
if (console.log.toString() !== 'function log() { [native code] }') {
    // 检测到调试工具
}
```

**3. DevTools打开检测**
```javascript
// 窗口尺寸差异检测
setInterval(() => {
    if (window.outerWidth - window.innerWidth > 160 ||
        window.outerHeight - window.innerHeight > 160) {
        // DevTools已打开（侧边或底部面板）
        document.body.innerHTML = '';
    }
}, 1000);
```

**4. 时间差检测**
```javascript
// debugger暂停会增加时间
const start = performance.now();
debugger;
if (performance.now() - start > 100) {
    // 检测到调试器暂停
    window.location.href = 'about:blank';
}
```

### 8.2 绕过方法

**绕过 debugger 陷阱**：

方法一：Deactivate All Breakpoints
- Sources面板 → 点击 `Deactivate breakpoints` 按钮（`Ctrl+F8`）
- 所有断点和debugger语句都不会触发

方法二：Never Pause Here
- 在 `debugger` 行右键 → `Never pause here`
- 仅跳过这一行的中断

方法三：条件断点返回false
- 在 `debugger` 行添加条件断点，表达式设为 `false`
- debugger语句被"覆盖"，不再触发

**绕过 console 覆写检测**：
```javascript
// 在页面最早期注入（通过Extension或Override）
// 保存原始console引用
Object.defineProperty(window, '__console', {
    value: { ...console },
    writable: false,
    configurable: false
});

// 让 toString 检测通过
const nativeToString = Function.prototype.toString;
Function.prototype.toString = function() {
    if (this === console.log) return 'function log() { [native code] }';
    return nativeToString.call(this);
};
```

**绕过窗口尺寸检测**：
```javascript
// Hook outerWidth/outerHeight
Object.defineProperty(window, 'outerWidth', {
    get: () => window.innerWidth
});
Object.defineProperty(window, 'outerHeight', {
    get: () => window.innerHeight + 80 // 正常标签栏/地址栏高度
});
```

**绕过时间差检测**：
```javascript
// Hook performance.now 和 Date.now
const originalNow = performance.now.bind(performance);
const originalDateNow = Date.now;
let offset = 0;

performance.now = function() {
    return originalNow() - offset;
};

Date.now = function() {
    return originalDateNow() - offset;
};

// 每次debugger暂停后自动补偿时间差（通过Extension在resume时更新offset）
```

**综合反反调试脚本**（通过 Local Override 或 Extension 注入）：
```javascript
// anti-anti-debug.js — 页面加载最早期注入
(function() {
    // 1. 禁止 debugger 语句执行
    const handler = setInterval(() => {}, 0);
    clearInterval(handler);
    
    // 重写Function构造函数，过滤debugger
    const OriginalFunction = Function;
    Function = function(...args) {
        if (args.length > 0) {
            args[args.length - 1] = args[args.length - 1].replace(/debugger/g, '');
        }
        return OriginalFunction(...args);
    };
    Function.prototype = OriginalFunction.prototype;
    Object.defineProperty(Function, 'prototype', { writable: false });
    
    // 2. 禁止 eval 中的 debugger
    const originalEval = window.eval;
    window.eval = function(code) {
        return originalEval(code.replace(/debugger/g, ''));
    };
    
    // 3. 固定窗口尺寸
    Object.defineProperty(window, 'outerWidth', { get: () => window.innerWidth });
    Object.defineProperty(window, 'outerHeight', { get: () => window.innerHeight + 85 });
    
    // 4. 时间保护
    const _perf = performance.now.bind(performance);
    const _date = Date.now;
    performance.now = function() { return _perf(); };
    Date.now = function() { return _date(); };
    
    console.log('[Anti-Debug] Protection active');
})();
```
