# Chrome Extension 逆向辅助工具开发

> Chrome Extension 可以在页面上下文中执行代码，拦截网络请求，
> 是 Web 逆向工程中轻量但强大的辅助工具。

---

## 目录

- [一、Extension在逆向中的用途](#一extension在逆向中的用途)
- [二、基础结构](#二基础结构)
- [三、核心功能实现](#三核心功能实现)
- [四、逆向专用Extension模板](#四逆向专用extension模板)
- [五、调试与发布](#五调试与发布)
- [六、注意事项](#六注意事项)

---

## 一、Extension在逆向中的用途

1. **注入Hook脚本** — 拦截 `fetch`、`XMLHttpRequest`、`crypto.subtle` 等原生API
2. **网络请求拦截与修改** — 修改请求Header、重定向URL、阻断特定资源
3. **页面DOM操作和数据提取** — 自动化提取页面中的动态数据
4. **本地存储中转** — 跨页面传递cookie、token等认证信息
5. **自动化操作辅助** — 模拟用户行为，触发特定逻辑路径

**实战场景**：目标网站通过 `crypto.subtle.sign()` 生成请求签名，使用Extension Hook此API可直接获取签名密钥和原始数据。

---

## 二、基础结构

### 2.1 manifest.json (Manifest V3)

```json
{
  "manifest_version": 3,
  "name": "Reverse Helper",
  "version": "1.0.0",
  "description": "Web逆向辅助工具",
  "permissions": [
    "webRequest",
    "scripting",
    "storage",
    "activeTab",
    "contextMenus",
    "cookies",
    "declarativeNetRequest"
  ],
  "host_permissions": [
    "<all_urls>"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["*://*.target.com/*"],
      "js": ["content.js"],
      "run_at": "document_start",
      "all_frames": true
    }
  ],
  "action": {
    "default_popup": "popup.html",
    "default_icon": "icon.png"
  },
  "web_accessible_resources": [
    {
      "resources": ["inject.js"],
      "matches": ["<all_urls>"]
    }
  ]
}
```

**关键权限说明**：
- `webRequest`：监控网络请求（只读，V3不支持阻断式修改）
- `scripting`：动态注入脚本到页面
- `storage`：本地持久化存储
- `declarativeNetRequest`：声明式请求修改（V3替代webRequestBlocking）

### 2.2 文件结构

```
my-reverse-helper/
├── manifest.json          ← 扩展配置
├── background.js          ← Service Worker（后台逻辑）
├── content.js             ← 注入到目标页面（隔离世界）
├── inject.js              ← 通过content.js注入到页面主世界
├── popup.html             ← 弹出面板UI
├── popup.js               ← 弹出面板逻辑
├── icon.png               ← 扩展图标
└── utils/
    ├── hook-templates.js  ← Hook模板库
    └── crypto-utils.js    ← 加密工具函数
```

---

## 三、核心功能实现

### 3.1 Hook注入（页面主世界执行）

**Content Script vs 主世界的区别**：
- Content Script 运行在隔离世界（isolated world），无法访问页面JS变量
- 主世界（main world）可以访问 `window`、覆写原生API，但不能使用 `chrome.*` API

**通过script标签注入到主世界**：

```javascript
// content.js — 负责将inject.js注入到页面主世界
(function() {
    const script = document.createElement('script');
    script.src = chrome.runtime.getURL('inject.js');
    script.onload = function() { this.remove(); };
    (document.head || document.documentElement).appendChild(script);
})();

// 监听来自inject.js的消息
window.addEventListener('message', (event) => {
    if (event.data && event.data.type === 'HOOK_DATA') {
        // 转发给background
        chrome.runtime.sendMessage({
            action: 'hookData',
            data: event.data.payload
        });
    }
});
```

```javascript
// inject.js — 在页面主世界中执行Hook
(function() {
    // Hook fetch
    const originalFetch = window.fetch;
    window.fetch = async function(...args) {
        const [url, options] = args;
        console.log('[Hook] fetch:', url, options);
        
        window.postMessage({
            type: 'HOOK_DATA',
            payload: { api: 'fetch', url: String(url), options }
        }, '*');
        
        const response = await originalFetch.apply(this, args);
        return response;
    };

    // Hook XMLHttpRequest
    const originalXHROpen = XMLHttpRequest.prototype.open;
    const originalXHRSend = XMLHttpRequest.prototype.send;
    
    XMLHttpRequest.prototype.open = function(method, url, ...rest) {
        this._hookInfo = { method, url };
        return originalXHROpen.call(this, method, url, ...rest);
    };
    
    XMLHttpRequest.prototype.send = function(body) {
        window.postMessage({
            type: 'HOOK_DATA',
            payload: { api: 'xhr', ...this._hookInfo, body }
        }, '*');
        return originalXHRSend.call(this, body);
    };

    // Hook WebSocket
    const OriginalWebSocket = window.WebSocket;
    window.WebSocket = function(url, protocols) {
        console.log('[Hook] WebSocket connect:', url);
        const ws = new OriginalWebSocket(url, protocols);
        
        const originalSend = ws.send.bind(ws);
        ws.send = function(data) {
            window.postMessage({
                type: 'HOOK_DATA',
                payload: { api: 'websocket', direction: 'send', data: String(data).slice(0, 500) }
            }, '*');
            return originalSend(data);
        };
        return ws;
    };
    window.WebSocket.prototype = OriginalWebSocket.prototype;
})();
```

### 3.2 网络请求拦截（declarativeNetRequest）

```javascript
// background.js — 动态添加请求修改规则
// 修改特定域名的请求Header
chrome.declarativeNetRequest.updateDynamicRules({
    removeRuleIds: [1, 2, 3],
    addRules: [
        {
            id: 1,
            priority: 1,
            action: {
                type: "modifyHeaders",
                requestHeaders: [
                    { header: "X-Custom-Auth", operation: "set", value: "my_token_123" },
                    { header: "Referer", operation: "set", value: "https://target.com/" }
                ]
            },
            condition: {
                urlFilter: "*://api.target.com/*",
                resourceTypes: ["xmlhttprequest", "main_frame", "sub_frame"]
            }
        },
        {
            id: 2,
            priority: 1,
            action: {
                type: "modifyHeaders",
                responseHeaders: [
                    { header: "Access-Control-Allow-Origin", operation: "set", value: "*" },
                    { header: "Content-Security-Policy", operation: "remove" }
                ]
            },
            condition: {
                urlFilter: "*://target.com/*",
                resourceTypes: ["main_frame", "sub_frame"]
            }
        }
    ]
});
```

### 3.3 Content Script ↔ Background 通信

```javascript
// content.js — 发送消息给background
chrome.runtime.sendMessage({ action: "getData", key: "cookies" }, (response) => {
    console.log("Received from background:", response);
});

// background.js — 接收并响应
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    if (message.action === "hookData") {
        // 存储Hook到的数据
        chrome.storage.local.get(["hookLogs"], (result) => {
            const logs = result.hookLogs || [];
            logs.push({ ...message.data, timestamp: Date.now() });
            // 只保留最近1000条
            if (logs.length > 1000) logs.splice(0, logs.length - 1000);
            chrome.storage.local.set({ hookLogs: logs });
        });
    }
    if (message.action === "getData") {
        chrome.storage.local.get([message.key], (result) => {
            sendResponse(result[message.key]);
        });
        return true; // 异步响应
    }
});

// 长连接模式（适合高频数据传输）
// content.js
const port = chrome.runtime.connect({ name: "hookStream" });
port.postMessage({ type: "init", url: location.href });

// background.js
chrome.runtime.onConnect.addListener((port) => {
    if (port.name === "hookStream") {
        port.onMessage.addListener((msg) => {
            console.log("[Stream]", msg);
        });
    }
});
```

### 3.4 本地存储中转

```javascript
// background.js — 存储抓取的Cookie供外部脚本使用
chrome.cookies.onChanged.addListener((changeInfo) => {
    if (changeInfo.cookie.domain.includes("target.com")) {
        chrome.storage.local.get(["targetCookies"], (result) => {
            const cookies = result.targetCookies || {};
            if (changeInfo.removed) {
                delete cookies[changeInfo.cookie.name];
            } else {
                cookies[changeInfo.cookie.name] = changeInfo.cookie.value;
            }
            chrome.storage.local.set({ targetCookies: cookies });
        });
    }
});

// popup.js — 一键导出当前cookies
document.getElementById("exportBtn").addEventListener("click", async () => {
    const result = await chrome.storage.local.get(["targetCookies"]);
    const cookies = result.targetCookies || {};
    
    // 复制到剪贴板
    const cookieStr = Object.entries(cookies).map(([k, v]) => `${k}=${v}`).join("; ");
    await navigator.clipboard.writeText(cookieStr);
    
    // 同时发送到本地Server
    try {
        await fetch("http://127.0.0.1:3456/cookies", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(cookies)
        });
    } catch(e) { /* 本地Server未启动时忽略 */ }
});
```

### 3.5 右键菜单快捷工具

```javascript
// background.js — 注册右键菜单
chrome.runtime.onInstalled.addListener(() => {
    chrome.contextMenus.create({
        id: "extractParams",
        title: "提取此页面所有加密参数",
        contexts: ["page"]
    });
    chrome.contextMenus.create({
        id: "hookCurrentPage",
        title: "对此页面启用Hook",
        contexts: ["page"]
    });
});

chrome.contextMenus.onClicked.addListener(async (info, tab) => {
    if (info.menuItemId === "extractParams") {
        // 在当前页面执行提取脚本
        const results = await chrome.scripting.executeScript({
            target: { tabId: tab.id },
            func: () => {
                // 提取页面中所有可能的加密参数
                const scripts = document.querySelectorAll("script");
                const patterns = /(?:sign|token|key|secret|encrypt|cipher)\s*[=:]\s*["']([^"']+)["']/gi;
                const found = [];
                scripts.forEach(s => {
                    let match;
                    while ((match = patterns.exec(s.textContent)) !== null) {
                        found.push({ pattern: match[0], value: match[1] });
                    }
                });
                return found;
            }
        });
        console.log("Extracted params:", results[0]?.result);
    }
    
    if (info.menuItemId === "hookCurrentPage") {
        await chrome.scripting.executeScript({
            target: { tabId: tab.id },
            files: ["inject.js"],
            world: "MAIN"
        });
    }
});
```

---

## 四、逆向专用Extension模板

### 4.1 API Hook Monitor

```javascript
// inject.js — 完整的API监控器
(function() {
    const LOG_SERVER = "http://127.0.0.1:3456/log";
    const logs = [];

    function sendLog(entry) {
        logs.push(entry);
        // 批量发送（每10条）
        if (logs.length >= 10) {
            const batch = logs.splice(0, 10);
            navigator.sendBeacon(LOG_SERVER, JSON.stringify(batch));
        }
    }

    // Hook crypto.subtle
    const originalSubtle = crypto.subtle;
    const subtleMethodNames = ['encrypt', 'decrypt', 'sign', 'verify', 'digest'];
    
    subtleMethodNames.forEach(method => {
        const original = originalSubtle[method].bind(originalSubtle);
        originalSubtle[method] = async function(...args) {
            const result = await original(...args);
            sendLog({
                api: `crypto.subtle.${method}`,
                args: args.map(a => a instanceof ArrayBuffer ? 
                    Array.from(new Uint8Array(a)).map(b => b.toString(16).padStart(2,'0')).join('').slice(0, 64) : 
                    JSON.stringify(a).slice(0, 200)),
                result: result instanceof ArrayBuffer ?
                    Array.from(new Uint8Array(result)).map(b => b.toString(16).padStart(2,'0')).join('').slice(0, 64) : 
                    'non-buffer',
                stack: new Error().stack.split('\n').slice(1, 5).join('\n'),
                time: Date.now()
            });
            return result;
        };
    });

    // Hook JSON.parse/stringify（捕获数据序列化）
    const originalParse = JSON.parse;
    JSON.parse = function(text, reviver) {
        const result = originalParse(text, reviver);
        if (typeof text === 'string' && text.length > 50) {
            sendLog({ api: 'JSON.parse', input: text.slice(0, 200), time: Date.now() });
        }
        return result;
    };

    console.log('[ReverseHelper] API Hook Monitor Active');
})();
```

### 4.2 Cookie Exporter

```javascript
// background.js — 实时Cookie导出
const TARGET_DOMAIN = ".target.com";
const EXPORT_ENDPOINT = "http://127.0.0.1:3456/cookies";

async function exportAllCookies() {
    const cookies = await chrome.cookies.getAll({ domain: TARGET_DOMAIN });
    const cookieMap = {};
    cookies.forEach(c => { cookieMap[c.name] = c.value; });
    
    try {
        await fetch(EXPORT_ENDPOINT, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ domain: TARGET_DOMAIN, cookies: cookieMap, timestamp: Date.now() })
        });
    } catch(e) { /* silent */ }
    
    return cookieMap;
}

// Cookie变化时自动导出
chrome.cookies.onChanged.addListener((changeInfo) => {
    if (changeInfo.cookie.domain.includes("target.com")) {
        exportAllCookies();
    }
});

// 定时导出（每30秒）
setInterval(exportAllCookies, 30000);
```

### 4.3 JS Function Tracer

```javascript
// inject.js — 函数调用追踪器
(function() {
    window.__tracedFunctions = {};
    
    /**
     * 追踪指定函数的所有调用
     * @param {object} obj - 函数所在对象
     * @param {string} methodName - 函数名
     */
    window.__trace = function(obj, methodName) {
        const original = obj[methodName];
        if (typeof original !== 'function') return;
        
        window.__tracedFunctions[methodName] = [];
        
        obj[methodName] = function(...args) {
            const entry = {
                name: methodName,
                args: args.map(a => {
                    try { return JSON.stringify(a).slice(0, 100); }
                    catch { return String(a).slice(0, 100); }
                }),
                stack: new Error().stack.split('\n').slice(2, 6).join('\n'),
                timestamp: performance.now()
            };
            
            window.__tracedFunctions[methodName].push(entry);
            console.log(`[Trace] ${methodName}(`, ...args.slice(0, 3), ')');
            
            const result = original.apply(this, args);
            
            // 如果返回Promise，追踪结果
            if (result && typeof result.then === 'function') {
                result.then(r => { entry.result = String(r).slice(0, 100); });
            } else {
                entry.result = String(result).slice(0, 100);
            }
            
            return result;
        };
        
        console.log(`[Trace] Now tracing: ${methodName}`);
    };

    // 使用方式（在Console中执行）：
    // __trace(window, 'encryptData')
    // __trace(CryptoUtils.prototype, 'sign')
    // 查看记录：__tracedFunctions['encryptData']
})();
```

### 4.4 Request Signature Verifier

```javascript
// inject.js — 拦截请求并验证签名
(function() {
    const originalFetch = window.fetch;
    
    // 你逆向出的签名算法
    async function computeSign(params, timestamp) {
        const sortedKeys = Object.keys(params).filter(k => k !== 'sign').sort();
        const signStr = sortedKeys.map(k => `${k}=${params[k]}`).join('&') + '&t=' + timestamp;
        
        const encoder = new TextEncoder();
        const key = await crypto.subtle.importKey(
            'raw', encoder.encode('discovered_secret_key'),
            { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
        );
        const signature = await crypto.subtle.sign('HMAC', key, encoder.encode(signStr));
        return Array.from(new Uint8Array(signature)).map(b => b.toString(16).padStart(2, '0')).join('');
    }
    
    window.fetch = async function(input, init) {
        const url = new URL(typeof input === 'string' ? input : input.url, location.origin);
        const params = Object.fromEntries(url.searchParams);
        
        if (params.sign) {
            const originalSign = params.sign;
            const computed = await computeSign(params, params.t || '');
            const match = originalSign === computed;
            
            console.log(
                `%c[SignVerify ${match ? '✓' : '✗'}] ${url.pathname}`,
                `color: ${match ? 'green' : 'red'}; font-weight: bold`
            );
            if (!match) {
                console.log('  Original:', originalSign);
                console.log('  Computed:', computed);
                console.log('  Params:', params);
            }
        }
        
        return originalFetch.apply(this, arguments);
    };
})();
```

---

## 五、调试与发布

### 5.1 开发者模式加载

1. 打开 `chrome://extensions/`
2. 开启右上角「开发者模式」
3. 点击「加载已解压的扩展程序」
4. 选择扩展目录
5. 修改代码后点击刷新图标重新加载

### 5.2 Service Worker调试

- `chrome://extensions/` → 找到你的扩展 → 点击「Service Worker」链接
- 打开的DevTools即为background的调试面板
- 注意：Service Worker会休眠，非持久化（不同于MV2的background page）

### 5.3 Content Script调试

- 在目标页面打开DevTools → Sources → Content scripts 分组
- 可以直接在Content Script中设置断点
- 注意：inject.js注入到主世界后，在Sources的Page分组中查看

### 5.4 打包为.crx

```bash
# 通过Chrome打包
# chrome://extensions/ → 打包扩展程序 → 选择目录

# 或使用命令行
chrome --pack-extension=/path/to/extension --pack-extension-key=/path/to/key.pem
```

---

## 六、注意事项

### 6.1 Manifest V3 限制

| 限制 | 影响 | 替代方案 |
|------|------|---------|
| 无 `webRequestBlocking` | 不能同步阻断/修改请求 | 使用 `declarativeNetRequest` |
| Service Worker 非持久 | 后台脚本会休眠 | 使用 `chrome.alarms` 或事件驱动 |
| 无远程代码执行 | 不能执行远程URL的JS | 所有代码必须打包在扩展内 |
| eval 限制 | CSP 禁止 `eval` | 使用 `chrome.scripting.executeScript` |

### 6.2 CSP限制处理

```javascript
// 某些页面的CSP会阻止脚本注入，解决方案：
// 1. 通过declarativeNetRequest移除CSP响应头
chrome.declarativeNetRequest.updateDynamicRules({
    removeRuleIds: [100],
    addRules: [{
        id: 100,
        priority: 1,
        action: {
            type: "modifyHeaders",
            responseHeaders: [
                { header: "Content-Security-Policy", operation: "remove" },
                { header: "Content-Security-Policy-Report-Only", operation: "remove" }
            ]
        },
        condition: { urlFilter: "*://*.target.com/*", resourceTypes: ["main_frame"] }
    }]
});

// 2. 使用chrome.scripting API的world: "MAIN"（Chrome 111+）
chrome.scripting.executeScript({
    target: { tabId },
    func: () => { /* 在主世界中执行 */ },
    world: "MAIN"
});
```

### 6.3 Extension与反检测的兼容性

- **问题**：某些网站检测Extension的存在（通过 `chrome.runtime.id` 或探测 `web_accessible_resources`）
- **方案**：
  1. 不暴露 `web_accessible_resources`（使用 `chrome.scripting` 动态注入）
  2. Hook `document.querySelectorAll` 隐藏注入的script标签
  3. 在inject.js执行完后立即移除script元素（`script.onload = () => script.remove()`）
