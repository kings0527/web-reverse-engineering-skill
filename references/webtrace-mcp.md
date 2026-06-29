# WebTrace MCP 工具使用指南

> WebTrace 是一个开源的 Chrome/Edge Extension，通过 MCP 协议暴露 Web 逆向分析能力。
> AI Agent 可直接调用工具完成保护检测、JSVMP分析、WASM逆向、API监控等任务。

**项目地址**：https://github.com/kings0527/web-trace
**协议版本**：MCP 2024-11-05
**支持方法**：`initialize`、`tools/list`、`tools/call`

---

## 连接方式

### WebSocket（外部 Agent，推荐）

架构：`AI Agent → ws://127.0.0.1:3100/mcp → Bridge → Extension Service Worker`

```python
import asyncio, websockets, json

async def mcp_call(tool_name, arguments={}):
    async with websockets.connect("ws://127.0.0.1:3100/mcp") as ws:
        await ws.send(json.dumps({
            "jsonrpc": "2.0", "id": 1,
            "method": "initialize",
            "params": {"protocolVersion": "2024-11-05", "clientInfo": {"name": "agent", "version": "1.0.0"}, "capabilities": {}}
        }))
        await ws.recv()
        await ws.send(json.dumps({
            "jsonrpc": "2.0", "id": 2,
            "method": "tools/call",
            "params": {"name": tool_name, "arguments": arguments}
        }))
        return json.loads(await ws.recv())
```

默认端口 `3100`，可通过环境变量 `WEBTRACE_MCP_PORT` 覆盖。

### Chrome Runtime Port（Extension 内部 Agent）

```javascript
const port = chrome.runtime.connect('WEBTRACE_EXTENSION_ID', { name: 'webtrace-mcp' });
port.postMessage({ jsonrpc: '2.0', id: 1, method: 'tools/call', params: { name: 'detect_protection', arguments: {} } });
port.onMessage.addListener((msg) => console.log(msg));
```

### 与 nodriver 协同

```python
import nodriver as uc

browser = await uc.start(browser_args=[
    '--load-extension=/path/to/web-trace/dist',
    '--disable-extensions-except=/path/to/web-trace/dist',
])
page = await browser.get('https://target-site.com')
# WebTrace 自动加载，通过 WebSocket bridge 或 CDP evaluate 交互
```

---

## 工具详细说明

### detect_protection — 检测页面保护类型

**输入**：`{ "url": "(可选) 目标URL，省略则分析当前活跃tab" }`

**输出**：`{ "type": "none|obfuscation|jsvmp|wasm|combined", "level": 1-5, "features": [...], "confidence": 0.0-1.0 }`

**用途**：快速判定目标网站的反爬类型和等级，替代手动DevTools分析。

### analyze_jsvmp — 分析 JSVMP 结构

**输入**：`{ "code": "(必选) JSVMP代码字符串", "deobfuscate": "(可选) 是否预处理反混淆" }`

**输出**：派发循环类型(while-switch/if-else/handler-table)、字节码数组变量名、PC变量名、操作码数量、分支映射。

**用途**：自动识别VM派发循环结构，省去手动定位switch-case的时间。

### trace_execution — QuickJS 沙箱执行追踪

**输入**：`{ "code": "(必选) 待执行代码", "inputs": "(可选) 注入全局变量", "maxTraceEntries": "(可选) 最大trace条数[100-100000]，默认10000" }`

**输出**：trace日志（PC值/操作码/栈快照/时间戳）、执行结果、统计信息（执行时间/内存/是否截断）。

**用途**：离线安全执行JSVMP代码，收集运行时操作码序列用于算法还原。

### extract_bytecode — 提取字节码数组

**输入**（至少指定一个）：`{ "scriptUrl": "JS文件URL", "selector": "CSS选择器", "variableName": "全局变量名" }`

**输出**：`{ "bytecode": [...], "format": "int-array|uint8|int32|hex-string", "sourceInfo": {...} }`

**用途**：从页面中自动定位并提取JSVMP字节码，无需手动断点。

### extract_wasm — 提取 WASM 模块

**输入**：`{ "scriptUrl": "(可选) 加载WASM的JS URL", "pageContext": "(可选) 从页面上下文捕获" }`

**输出**：捕获的WASM二进制数据及元信息。

**用途**：拦截 `WebAssembly.instantiate` 调用，获取WASM模块二进制用于离线分析。

### analyze_wasm — WASM 反汇编分析

**输入**：`{ "wasmBytes": "(必选) WASM字节数组或Base64", "focusFunctions": "(可选) 关注的函数名列表" }`

**输出**：模块结构（导入/导出/函数列表）、反汇编输出、识别到的加密算法模式。

**用途**：对提取的WASM模块进行结构分析和加密算法特征匹配。

### dump_wasm_memory — 读取 WASM 线性内存

**输入**：`{ "offset": "(必选) 起始偏移字节", "length": "(必选) 读取长度", "format": "(可选) hex|uint8|utf8" }`

**输出**：指定范围的内存数据。

**用途**：在WASM执行后读取内存，提取密钥、中间状态等运行时数据。

### hook_api — 设置 API Hook

**输入**：`{ "apiName": "(必选) API名称如fetch/XMLHttpRequest/crypto.subtle.digest", "mode": "(必选) intercept|observe", "options": { "captureArgs": true, "captureResult": true, "maxLogs": 1000 } }`

**输出**：`{ "hookId": "hook_xxx", "status": "active", "apiName": "...", "mode": "..." }`

**用途**：在目标页面隐蔽Hook浏览器API，监控签名参数生成和加密操作。

### get_hook_logs — 获取 Hook 日志

**输入**：`{ "hookId": "(可选) 指定hookId", "filter": { "apiName": "...", "timeRange": {...} }, "limit": 100 }`

**输出**：`{ "logs": [{ "hookId", "apiName", "timestamp", "args", "result" }], "totalAvailable": N, "activeHooks": [...] }`

**用途**：收集Hook捕获的API调用记录，提取签名参数值和加密输入输出。

### page_state — 获取页面状态

**输入**：无参数

**输出**：当前页面URL、title、cookies、localStorage、sessionStorage、scripts列表（含hasVMPattern标记）、网络请求列表。

**用途**：快速获取页面全貌，识别关键JS文件和cookie结构。

### deobfuscate — 反混淆 JS 代码

**输入**：`{ "code": "(必选) 混淆代码", "transforms": "(可选) 指定变换列表" }`

**可用变换**：`string-array`, `dead-code`, `control-flow`, `rename`, `constant-fold`, `member-expression`, `boolean-simplify`, `hex-numeric`, `unicode-escape`, `comma-expression`

**输出**：`{ "cleanedCode": "...", "transformsApplied": [...], "confidence": 0.85, "stats": { "reductionPercent": 76 } }`

**用途**：在线反混淆，直接对页面中的混淆代码进行清洗。

---

## 典型分析场景

### 场景1：未知网站快速判定

```
page_state → detect_protection → 根据level选择路径
```

1. 调用 `page_state` 获取cookie、scripts列表、hasVMPattern标记
2. 调用 `detect_protection` 获取保护类型和等级
3. level≤2 直接 `deobfuscate`；level=4 进入JSVMP流程；level=5 进入组合分析

### 场景2：JSVMP 算法还原

```
detect_protection → extract_bytecode → analyze_jsvmp → trace_execution → 提取算法
```

1. 确认 type=jsvmp
2. `extract_bytecode({variableName: "window._bytecode"})` 提取字节码数组
3. `analyze_jsvmp({code: vm_code, deobfuscate: true})` 分析派发循环结构
4. `trace_execution({code: vm_code, inputs: {msg: "test"}})` 收集trace日志
5. 根据trace中的操作码序列还原签名算法

### 场景3：WASM 加密分析

```
extract_wasm → analyze_wasm → dump_wasm_memory → 定位加密逻辑
```

1. `extract_wasm({pageContext: true})` 拦截WASM模块加载
2. `analyze_wasm({wasmBytes: captured_binary})` 反汇编并识别加密算法模式
3. 触发目标操作后 `dump_wasm_memory({offset: 0, length: 4096})` 提取密钥/状态

### 场景4：API 签名逆向

```
hook_api(fetch) → 触发操作 → get_hook_logs → 定位签名函数 → analyze_jsvmp
```

1. `hook_api({apiName: "fetch", mode: "observe"})` 监控所有fetch请求
2. 在浏览器中触发目标操作（导航、点击等）
3. `get_hook_logs({limit: 50})` 收集捕获的请求日志
4. 从日志中识别签名参数（如X-Bogus header、sign query参数）
5. 定位生成签名的函数，用 `analyze_jsvmp` 分析其结构

---

## 与 Skill 六步法的映射

| Skill步骤 | 可调用的 WebTrace Tool | 说明 |
|-----------|----------------------|------|
| Step 1：侦察 | `page_state` | 获取页面cookie/storage/scripts/网络请求 |
| Step 2：识别 | `detect_protection` | 自动判定保护类型和等级 |
| Step 3：定位 | `hook_api` + `get_hook_logs` | 通过Hook定位签名参数生成点 |
| Step 4：分析 | `analyze_jsvmp`, `trace_execution`, `deobfuscate` | VM结构分析、执行追踪、代码清洗 |
| Step 4：分析(WASM) | `extract_wasm`, `analyze_wasm`, `dump_wasm_memory` | WASM提取、反汇编、内存读取 |
| Step 5：还原 | `trace_execution` | 用trace日志验证还原算法的正确性 |
| Step 6：验证 | `hook_api`(intercept模式) | 拦截请求注入还原的签名验证正确性 |

---

## 执行约束

- **互斥锁**：同一时间仅一个Tool执行（避免浏览器API并发冲突）
- **超时**：每个Tool最大执行时间35秒
- **多Tab**：默认操作当前活跃Tab，`detect_protection` 可通过url参数指定目标
- **重试**：MCP协议层不内置重试，Agent可自行实现

## 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `WEBTRACE_MCP_PORT` | `3100` | WebSocket bridge 监听端口 |
| `WEBTRACE_MCP_HOST` | `127.0.0.1` | WebSocket bridge 绑定地址 |
| `WEBTRACE_LOG_LEVEL` | `info` | 日志级别(debug/info/warn/error) |
