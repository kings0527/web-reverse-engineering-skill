# AI 辅助逆向工具设计

> 本文涵盖：MCP工具接口定义模式（ToolDefinition）、AI Agent逆向五阶段工作流、Skill定义最佳实践、工具注册与调度架构。所有代码片段均来自 js-reverse-mcp 和 reverse-skill 项目。

## 概述

AI辅助逆向工程的核心思路是将浏览器调试、AST分析、网络抓包等能力封装为标准化工具接口（MCP Protocol），让AI Agent能够像人类逆向工程师一样操作浏览器、设置断点、查看源码。本文解析这套工具链的设计模式和最佳实践。

---

## 1. MCP 工具接口定义模式

### 1.1 ToolDefinition 接口

来源：`js-reverse-mcp/src/tools/ToolDefinition.ts`

每个工具都遵循统一的 `ToolDefinition` 接口：

```typescript
// 来源：js-reverse-mcp - tools/ToolDefinition.ts
export interface ToolDefinition<Schema extends zod.ZodRawShape = zod.ZodRawShape> {
  name: string;
  description: string;
  annotations: {
    title?: string;
    category: ToolCategory;
    readOnlyHint: boolean;  // 是否只读（不影响浏览器状态）
  };
  schema: Schema;           // zod schema定义输入参数
  handler: (
    request: Request<Schema>,
    response: Response,
    context: Context,
  ) => Promise<void>;
}
```

**设计要点**：

- **zod schema**：AI通过schema理解每个参数的类型和含义，自动生成正确的调用
- **annotations.category**：工具分类（导航/网络/调试/逆向），用于调度策略
- **readOnlyHint**：告诉AI这个工具是否安全（不会修改状态），影响AI的调用决策

### 1.2 工具分类体系

来源：`js-reverse-mcp/src/tools/categories.ts`

```typescript
// 来源：js-reverse-mcp - tools/categories.ts
export enum ToolCategory {
  NAVIGATION = 'navigation',            // 页面导航
  BROWSER_STATE = 'browser_state',       // 浏览器状态管理
  NETWORK = 'network',                   // 网络请求分析
  DEBUGGING = 'debugging',               // 断点调试
  REVERSE_ENGINEERING = 'reverse_engineering',  // JS逆向专用
}
```

**为什么分类重要？** 不同类别的工具在调用时需要不同的前置处理。例如，导航类工具需要在CDP静默状态下执行（不触发任何CDP调用），因为反爬系统会检测CDP活动。

### 1.3 实际工具定义示例

```typescript
// 来源：js-reverse-mcp - tools/debugger.ts
export const listScripts = defineTool({
  name: 'list_scripts',
  description:
    'Lists all JavaScript scripts loaded in the current page. ' +
    'Returns script ID, URL, and source map information. ' +
    'Use this to find scripts before setting breakpoints or searching. ' +
    'Script IDs are valid for the current page load only.',
  annotations: {
    title: 'List Scripts',
    category: ToolCategory.REVERSE_ENGINEERING,
    readOnlyHint: true,   // 只读操作，安全
  },
  schema: {
    filter: zod.string().optional()
      .describe('Optional filter string to match against script URLs (case-insensitive partial match).'),
  },
  handler: async (request, response, context) => {
    const debugger_ = context.debuggerContext;
    if (!debugger_.isEnabled()) {
      response.appendResponseLine('Debugger is not enabled. Please select a page first.');
      return;
    }
    // ... 实现逻辑
  },
});

// XHR断点工具 —— 写操作
export const breakOnXhr = defineTool({
  name: 'break_on_xhr',
  description:
    'Sets a breakpoint that triggers when an XHR/Fetch request URL contains the specified string.',
  annotations: {
    title: 'Break on XHR',
    category: ToolCategory.REVERSE_ENGINEERING,
    readOnlyHint: false,  // 会修改调试状态
  },
  schema: {
    url: zod.string().describe('URL pattern to break on (partial match).'),
  },
  handler: async (request, response, context) => {
    // ... 设置XHR断点
  },
});
```

---

## 2. 工具注册与调度架构

### 2.1 统一注册机制

来源：`js-reverse-mcp/src/main.ts`

```typescript
// 来源：js-reverse-mcp - main.ts
// 聚合9个工具模块
const tools = [
  ...Object.values(consoleTools),
  ...Object.values(debuggerTools),    // 12个调试工具
  ...Object.values(frameTools),       // iframe框架工具
  ...Object.values(networkTools),     // 网络分析工具
  ...Object.values(pagesTools),       // 页面管理工具
  ...Object.values(screenshotTools),  // 截图工具
  ...Object.values(scriptTools),      // 脚本注入工具
  ...Object.values(siteDataTools),    // 站点数据工具
  ...Object.values(websocketTools),   // WebSocket工具
];

// 排序确保一致性（AI按工具名列举时更稳定）
tools.sort((a, b) => a.name.localeCompare(b.name));

for (const tool of tools) {
  registerTool(tool);
}
```

### 2.2 注册包装器：互斥锁 + 超时

```typescript
// 来源：js-reverse-mcp - main.ts
const toolMutex = new Mutex();
const DEFAULT_TOOL_TIMEOUT_MS = 35_000;

function registerTool(tool: ToolDefinition): void {
  server.registerTool(tool.name, {
    description: tool.description,
    inputSchema: tool.schema,
    annotations: tool.annotations,
  }, async (params): Promise<CallToolResult> => {
    // 1. 获取互斥锁（防止并发工具调用冲突）
    let guard = await toolMutex.acquire({ timeoutMs: DEFAULT_TOOL_TIMEOUT_MS });
    try {
      return await withToolTimeout(
        (async () => {
          const context = await getContext();
          
          // 2. 分类决策：导航类工具不触发CDP初始化
          if (tool.annotations.category !== ToolCategory.NAVIGATION &&
              tool.annotations.category !== ToolCategory.BROWSER_STATE) {
            await context.ensureCollectorsInitialized();
            await context.detectOpenDevToolsWindows();
          }
          
          // 3. 执行工具handler
          const response = new McpResponse();
          await tool.handler({ params }, response, context);
          return { content: await response.handle(tool.name, context) };
        })(),
        DEFAULT_TOOL_TIMEOUT_MS,
        tool.name
      );
    } finally {
      guard.dispose();
    }
  });
}
```

**关键设计决策**：

| 机制 | 目的 | 为什么需要 |
|------|------|-----------|
| 互斥锁 | 串行化工具调用 | 浏览器CDP不支持并发调试操作 |
| 超时控制 | 防止AI死等 | 断点可能永不触发，需要35秒兜底 |
| 分类静默 | 导航工具跳过CDP初始化 | 反爬系统检测CDP活动痕迹 |

---

## 3. AI Agent 五阶段工作流

### 3.1 工作流定义

来源：`reverse-skill/skills/js-reverse/SKILL.md`

```
Observe → Capture → Rebuild → Patch → DeepDive
```

每个阶段有明确的**输入**、**动作**和**产出**：

**阶段1：Observe（观察）**
```
目标：确认目标请求、相关脚本、候选函数，不猜测

动作：
- list_scripts → 找到可疑脚本
- list_network_requests → 定位目标请求
- get_request_initiator → 回溯调用来源
- search_in_sources → 搜索签名参数名

产出：目标URL、initiator线索、可疑脚本列表
```

**阶段2：Capture（采样）**
```
目标：最小侵入采样，拿到参数样例和运行时证据

原则：Hook-preferred, Breakpoint-last

动作：
- break_on_xhr → 在目标API上设XHR断点
- evaluate_script → 轻量运行时观察
- get_paused_info → 断点命中后查看调用栈和变量

产出：函数入参/出参样例、调用顺序、运行时证据
```

**阶段3：Rebuild（复现）**
```
目标：将页面证据整理成本地Node复现材料

规则：
- 本地补环境必须以页面观测证据为依据
- 不允许空想式补 window/document/navigator
- 每次只记录一个最小因果补丁决策
```

**阶段4：Patch（补环境）**
```
目标：按报错驱动补环境，直到本地代码稳定跑出目标参数

规则：
- 先看缺什么，再补什么
- 一次只做一个最小补丁决策
- 每次补丁后立即复测
```

**阶段5：DeepDive（深化）**
```
目标：本地跑通后，再做去混淆、控制流还原、业务逻辑提纯

可选：如果只需出签名，此阶段可降级
```

### 3.2 核心原则

```
Observe-first    先页面观察，不猜测
Hook-preferred   优先用Hook采样，而非断点
Breakpoint-last  断点是最后手段（打断执行流）
Rebuild-oriented 目标是本地复现，不是理解全部代码
Evidence-first   所有结论必须有证据支撑
```

---

## 4. Skill 定义最佳实践

### 4.1 Skill 结构模板

来源：`reverse-skill/skills/SKILL.md`（总控）

一个完整的Skill定义包含以下要素：

```markdown
# Skill 结构模板

## 适用范围
- 明确列出适用场景
- 明确列出不适用场景（指向其他Skill）

## 工具映射
- 将通用操作映射到具体的MCP工具名
- 不假设裸工具名存在

## 核心原则
- 3-5条指导原则（如Observe-first）

## 工作流
- 分阶段定义，每阶段有明确的输入/动作/产出

## 必读引用
- 列出相关的参考文档

## 路由上下文
- 上游入口（从哪里进入本Skill）
- 下游出口（什么时候转移到其他Skill）
```

### 4.2 路由执行契约

reverse-skill的总控Skill定义了一个严格的执行契约：

```markdown
## 路由执行契约（必须立即执行）

读完本文件后，必须按顺序执行：
1. NOW：读取 routing.md，完成路由判定
2. NOW：读取目标子模块 SKILL.md，提取第一步可执行动作
3. NEXT：若涉及本机工具，读取 tool-index.md 校验路径
4. THEN：若缺工具，调用 bootstrap-reverse.ps1 自动补齐
5. ACT：执行任务，不停留在"等待确认"状态
```

### 4.3 按需自举

工具不在本地时，不报错而是自动安装：

```powershell
# 来源：reverse-skill - SKILL.md
powershell -File "scripts\bootstrap-reverse.ps1" -Capability @('jshookmcp') -StartServices
```

支持的自动注册能力包括：jadx、frida、r2、nmap、pwntools、ghidra-mcp 等30+工具。

---

## 5. Context 接口设计

### 5.1 Context 的作用

来源：`js-reverse-mcp/src/tools/ToolDefinition.ts`

Context 是工具handler访问浏览器能力的统一接口：

```typescript
// 来源：js-reverse-mcp - tools/ToolDefinition.ts（精简版）
export type Context = Readonly<{
  // 页面管理
  getSelectedPage(): Page;
  getPages(): Page[];
  selectPage(page: Page): void;

  // 调试器
  debuggerContext: DebuggerContext;

  // 网络分析
  getRequestInitiator(request: HTTPRequest): RequestInitiator | undefined;
  getNetworkRequestById(reqid: number): HTTPRequest;

  // WebSocket分析
  getWebSocketConnections(): WebSocketData[];
  getWebSocketById(wsid: number): WebSocketData;

  // 框架选择（支持iframe穿透）
  getSelectedFrame(): Frame;
  selectFrame(frame: Frame): void;

  // 文件保存
  saveFile(data: Uint8Array, filename: string): Promise<{ filename: string }>;
}>;
```

**设计模式**：Context是 `Readonly` 的，工具不能修改Context本身，只能通过Context提供的方法操作浏览器。这确保了工具之间的隔离性。

### 5.2 Response 接口

```typescript
// 来源：js-reverse-mcp - tools/ToolDefinition.ts
export interface Response {
  appendResponseLine(value: string): void;     // 追加文本响应
  setIncludePages(value: boolean): void;       // 包含页面列表
  setIncludeNetworkRequests(value: boolean): void;  // 包含网络请求
  attachImage(value: ImageContentData): void;  // 附加截图
  attachNetworkRequest(reqid: number): void;   // 附加特定请求
}
```

工具通过Response接口返回结构化结果，AI可以解析文本并做出决策。

---

## 6. AI逆向工作流实战示例

以还原一个API签名参数为例：

```
Step 1 - Observe:
  AI → list_network_requests(filter="api/data")
  AI → get_request_initiator(requestId=42)
  AI → search_in_sources(query="_signature")
  → 定位：bundle.js 中的 sign() 函数

Step 2 - Capture:
  AI → break_on_xhr(url="api/data")
  AI → [用户触发请求]
  AI → get_paused_info(includeScopes=true)
  → 获得：sign(timestamp, path) 的入参和返回值

Step 3 - Rebuild:
  AI → get_script_source(url="bundle.js", startLine=1024, endLine=1100)
  AI → save_script_source(url="bundle.js", filePath="./bundle.js")
  → 本地阅读反混淆后的代码

Step 4 - Patch:
  AI → evaluate_script(code="typeof window")
  → 在Node.js中补环境，逐步运行sign()

Step 5 - DeepDive:
  AI → [如果需要] 对混淆代码做AST分析
  → 输出纯JS签名函数
```

---

## 7. 设计总结

| 设计模式 | 实现方式 | 解决的问题 |
|---------|---------|-----------|
| 类型安全Schema | zod定义参数 | AI自动生成正确的工具调用 |
| 工具分类 | ToolCategory枚举 | 不同类别不同的前置处理策略 |
| 串行化执行 | Mutex互斥锁 | CDP不支持并发调试 |
| 超时保护 | Promise.race | 防止AI死等断点 |
| 分阶段工作流 | Observe→Capture→Rebuild | 避免过早深入，保持证据驱动 |
| 按需自举 | bootstrap脚本 | 工具缺失时自动安装 |
| Context隔离 | Readonly接口 | 工具间互不干扰 |
