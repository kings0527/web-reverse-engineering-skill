# CDP反检测技术：三种方案深度对比

> 本文从nodriver、rebrowser-patches、patchright三个开源项目中提取CDP反检测的核心代码模式，分析Runtime.enable泄露的修补策略和各自的优缺点。

## 1. CDP检测的核心问题

### 1.1 Runtime.enable泄露机制

当Playwright/Puppeteer启动浏览器时，会调用`Runtime.enable`来启用JS运行时域。这个调用会：
1. 触发`executionContextCreated`事件，暴露自动化框架的执行上下文
2. 在页面JS中可以通过特定API检测到CDP Runtime域已启用
3. Cloudflare、DataDome等反爬系统会主动探测此信号

### 1.2 其他检测向量

| 检测向量 | 原理 | 严重性 |
|---------|------|--------|
| `navigator.webdriver` | Chrome自动化启动时设置`webdriver=true` | 中 |
| sourceURL注入 | `page.evaluate()`注入`//# sourceURL=__pptr:__`标记 | 低 |
| `--enable-automation` | Chrome启动参数暴露自动化标志 | 中 |
| `Target.setAutoAttach` | CDP Target域的自动附加留下可检测痕迹 | 高 |
| DevTools窗口检测 | `outerWidth - innerWidth`差异 | 低 |

## 2. 方案一：nodriver直连模式

### 2.1 核心架构

nodriver（v0.50.3）完全绕过Playwright/Selenium，直接用Python与Chrome DevTools Protocol通信：

```python
# 来源：nodriver/core/connection.py - Connection类
class Connection:
    socket: websockets.asyncio.client.ClientConnection | None
    websocket_url: str = None

    def __init__(self, target=None, parent=None, auto_attach=False, **kwargs):
        self.session_id = None
        self.websocket_url = parent.websocket_url if parent else None
        self.handlers = collections.defaultdict(list)
        self.lock = asyncio.Lock()
        self.socket = None
        self.target = target
        self.parent = parent
        self._mapper: dict[int, asyncio.Future] = {}
        self._listener_task: asyncio.Task | None = None

    async def aopen(self):
        """直接建立WebSocket连接，无中间层"""
        if not self.socket or bool(self.socket.close_code):
            self.socket = await websockets.connect(
                self.websocket_url,
                ping_timeout=PING_TIMEOUT,   # 900秒
                max_size=MAX_SIZE,           # 2^28字节
            )
            self._listener_task = asyncio.create_task(self._listener())
```

### 2.2 为什么nodriver能通过31/31检测

1. **无中间层**：直接WebSocket → Chrome CDP，不引入Playwright的`Target.setAutoAttach`等额外CDP调用
2. **无WebDriver协议**：不通过Selenium WebDriver，`navigator.webdriver`保持undefined
3. **进程级控制**：Chrome启动时传入`--disable-blink-features=AutomationControlled`
4. **Flat Mode**：通过`Target.setAutoAttach(flatten=true)`直接控制所有iframe

```python
# 来源：nodriver使用示例
import nodriver as uc

async def main():
    browser = await uc.start()
    page = await browser.get('https://target.com')
    # 直接CDP通信，无Playwright/Selenium中间层泄露
    result = await page.evaluate('document.title')
    await browser.stop()

uc.loop().run_until_complete(main())
```

### 2.3 优缺点

| 维度 | 优势 | 劣势 |
|------|------|------|
| 检测通过率 | 31/31全通过 | - |
| 生态 | 纯Python，无Node.js依赖 | 社区较小 |
| API丰富度 | 直接CDP，可调用所有协议 | 需要手动处理CDP细节 |
| 学习曲线 | 简单直接 | 需要理解CDP协议 |
| iframe支持 | Flat Mode穿透 | 需要手动管理session |

## 3. 方案二：rebrowser-patches三策略修补

### 3.1 补丁结构

rebrowser-patches为Puppeteer和Playwright提供运行时修补，核心是解决`Runtime.enable`泄露：

```
patches/
├── playwright-core/   # Playwright修补
│   └── ...
└── puppeteer-core/    # Puppeteer修补
    └── ...
```

### 3.2 三种Runtime.enable修补策略

**策略1：addBinding（推荐，默认策略）**

核心思想：在主世界创建一个随机名称的binding，通过调用这个binding来获取`executionContextId`，完全不需要`Runtime.enable`：

```javascript
// 概念性代码（基于rebrowser-patches README分析）
// 环境变量：REBROWSER_PATCHES_RUNTIME_FIX_MODE=addBinding

// 1. 在主世界创建随机binding
const bindingName = `__rebinding_${crypto.randomUUID()}`;
await page.exposeBinding(bindingName, (source, ...args) => {
    // 通过binding回调获取executionContextId
    return source.executionContextId;
});

// 2. 页面加载时自动触发binding，获取context ID
// 3. 后续所有evaluate操作都使用这个已知的context ID
// 4. 不需要调用Runtime.enable，因此不会泄露
```

**优势**：保持主世界访问能力，支持Web Worker和iframe，是最透明的方案。

**策略2：alwaysIsolated**

```javascript
// 环境变量：REBROWSER_PATCHES_RUNTIME_FIX_MODE=alwaysIsolated

// 通过Page.createIsolatedWorld创建隔离的执行上下文
const isolatedWorld = await page.createIsolatedWorld({
    frameId: mainFrame.id,
    worldName: '__rebrowser_isolated'
});

// 所有evaluate操作都在隔离世界中执行
// 隔离世界不触发executionContextCreated事件
```

**劣势**：隔离世界无法访问主世界的变量（如`window.xxx`），限制了实用性。

**策略3：enableDisable**

```javascript
// 环境变量：REBROWSER_PATCHES_RUNTIME_FIX_MODE=enableDisable

// 快速开关Runtime域：
await Runtime.enable();   // 触发executionContextCreated
// 立即捕获context ID
await Runtime.disable();  // 关闭Runtime域
// 窗口期极短，页面检测代码很难捕捉到
```

**风险**：虽然窗口期极短（微秒级），但理论上存在被高精度检测代码捕获的可能。

### 3.3 优缺点

| 维度 | 优势 | 劣势 |
|------|------|------|
| 兼容性 | 与现有Puppeteer/Playwright代码兼容 | 需要patch，版本更新可能冲突 |
| 检测通过率 | Cloudflare/DataDome通过 | 部分高级检测仍可能触发 |
| 维护 | 自动化patch工具 | Playwright版本更新后需重新patch |
| 灵活性 | 三策略可选 | 策略选择需要理解原理 |

## 4. 方案三：patchright全面修补

### 4.1 Chromium启动参数清洗

patchright通过ts-morph对Playwright源码进行AST级修补，移除暴露自动化特征的启动参数：

```typescript
// 来源：patchright/driver_patches/chromiumSwitchesPatch.ts
export function patchChromiumSwitches(project: Project) {
    const chromiumSwitchesSourceFile = project.addSourceFileAtPath(
        "packages/playwright-core/src/server/chromium/chromiumSwitches.ts"
    );

    // 移除暴露自动化特征的启动参数
    const switchesToDisable = [
        "assistantMode ? '' : '--enable-automation'",  // 核心：移除自动化标志
        "'--disable-popup-blocking'",
        "'--disable-component-update'",
        "'--disable-default-apps'",
        "'--disable-extensions'",
        "'--disable-client-side-phishing-detection'",
        "'--disable-component-extensions-with-background-pages'",
        "'--allow-pre-commit-input'",
        "'--disable-ipc-flooding-protection'",
        "'--metrics-recording-only'",
        "'--unsafely-disable-devtools-self-xss-warnings'",
        "'--disable-back-forward-cache'",
        // ... 还有disable-features长列表
    ];

    // AST操作：从数组字面量中移除这些元素
    chromiumSwitchesArray.getElements()
        .filter((element) => switchesToDisable.includes(element.getText()))
        .forEach((element) => { chromiumSwitchesArray.removeElement(element); });

    // 添加反检测启动参数
    chromiumSwitchesArray.addElement(
        `'--disable-blink-features=AutomationControlled'`
    );
}
```

**为什么移除这些参数**：`--enable-automation`是Chrome的自动化标志，会设置`navigator.webdriver=true`并影响多个浏览器行为。`--metrics-recording-only`等参数虽然看似无害，但它们的组合模式本身就是自动化的指纹。

### 4.2 页面级修补

patchright对CRPage类进行全面改造，包括随机化脚本标签、重构binding机制等：

```typescript
// 来源：patchright/driver_patches/crPagePatch.ts
export function patchCRPage(project: Project) {
    const crPageSourceFile = project.addSourceFileAtPath(
        "packages/playwright-core/src/server/chromium/crPage.ts"
    );

    // 1. 添加crypto导入（用于生成随机脚本标签名）
    crPageSourceFile.addImportDeclaration({
        moduleSpecifier: "crypto",
        defaultImport: "crypto",
    });

    // 2. 替换updateRequestInterception为直接网络管理器拦截
    // 原始的updateRequestInterception()会留下可检测的CDP调用痕迹
    updateRequestInterceptionStatement.replaceWithText(`
        this._networkManager.setRequestInterception(true);
        this.initScriptTag = crypto.randomBytes(20).toString('hex');
    `);

    // 3. 添加exposeBinding方法：跨所有frame session初始化binding
    crPageClass.addMethod({
        name: "exposeBinding",
        isAsync: true,
        parameters: [{ name: "binding", type: "PageBinding" }],
    });
    crExposeBindingMethod.setBodyText(`
        await this._forAllFrameSessions(frame => frame._initBinding(binding));
        await Promise.all(
            this._page.frames().map(frame => frame.evaluateExpression(binding.source).catch(e => {}))
        );
    `);

    // 4. FrameSession类添加属性追踪
    frameSessionClass.addProperty({
        name: "_exposedBindingNames",
        type: "string[]",
        initializer: "[]",
    });
    frameSessionClass.addProperty({
        name: "_evaluateOnNewDocumentScripts",
        type: "InitScript[]",
        initializer: "[]",
    });
    frameSessionClass.addProperty({
        name: "_parsedExecutionContextIds",
        type: "number[]",
        initializer: "[]",
    });
}
```

### 4.3 32个修补模块

patchright的`driver_patches/`目录包含32个独立的修补模块，覆盖Playwright的方方面面：
- `chromiumSwitchesPatch.ts`：启动参数清洗
- `crPagePatch.ts`：页面级修补
- 其他模块覆盖：网络管理、帧会话、浏览器上下文、输入事件等

### 4.4 优缺点

| 维度 | 优势 | 劣势 |
|------|------|------|
| 覆盖面 | 32个模块全面修补 | 复杂度高 |
| API兼容 | 与Playwright API完全兼容 | 更新滞后于Playwright |
| 检测通过率 | 主流检测通过 | 某些定制检测可能触发 |
| 使用简单度 | `npm i patchright`即用 | 修补过程可能失败 |

## 5. 三种方案综合对比

| 维度 | nodriver | rebrowser-patches | patchright |
|------|----------|-------------------|------------|
| 技术路线 | 直连CDP WebSocket | 运行时策略修补 | AST级源码修补 |
| 语言 | Python | Node.js | Node.js |
| 中间层 | 无 | Puppeteer/Playwright | Playwright(fork) |
| Runtime.enable | 不调用 | 三策略回避 | 从协议层消除 |
| 维护成本 | 低（CDP协议稳定） | 中（版本兼容） | 高（32个补丁维护） |
| 最佳场景 | Python爬虫 | 现有Puppeteer项目迁移 | 需要Playwright生态 |
| 风险点 | CDP协议变化 | patch失败 | Playwright大版本更新 |

**选型建议**：
- 新项目首选nodriver（最简单、通过率最高）
- 现有Puppeteer项目用rebrowser-patches
- 需要Playwright高级API（如多浏览器支持）用patchright
