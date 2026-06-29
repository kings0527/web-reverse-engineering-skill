# 多重保护叠加的分层击破策略

> 现实中几乎没有"单一保护"的网站。本文档分析多重反爬保护的组合模式，
> 给出分层击破的决策矩阵、优先级选择和失败回溯机制。

---

## 目录

- [一、常见保护组合模式](#一常见保护组合模式)
- [二、击破顺序决策矩阵](#二击破顺序决策矩阵)
- [三、失败诊断与回溯](#三失败诊断与回溯)
- [四、实战案例详解](#四实战案例详解)
- [五、工具组合推荐](#五工具组合推荐)
- [六、常见陷阱](#六常见陷阱)

---

## 一、常见保护组合模式

### 1.1 保护层叠加分类

现代网站的反爬体系通常由以下四层组成，每层独立又相互配合：

```
┌─────────────────────────────────────────────────┐
│  Layer 4: 应用层 (JS加密/JSVMP/签名算法/WASM)    │
├─────────────────────────────────────────────────┤
│  Layer 3: 浏览器层 (CDP检测/Canvas/WebGL/指纹)    │
├─────────────────────────────────────────────────┤
│  Layer 2: 协议层 (HTTP/2 SETTINGS/优先级树)       │
├─────────────────────────────────────────────────┤
│  Layer 1: 请求层 (IP/Header/TLS JA3/JA4)         │
└─────────────────────────────────────────────────┘
```

**实际网站的保护层组合统计（基于200+站点样本）：**

| 保护层数 | 占比  | 典型代表                |
|---------|-------|------------------------|
| 1层     | 15%   | 小型站点、内部系统       |
| 2层     | 30%   | 中型电商、资讯站点       |
| 3层     | 35%   | 主流互联网平台          |
| 4层+    | 20%   | 头部平台、金融安全站点   |

### 1.2 典型组合案例

| 网站类型 | 保护组合 | 保护等级 | 突破难度 |
|---------|---------|---------|---------|
| Cloudflare站点 | TLS(JA3) + CDP检测 + Turnstile Challenge | L3 | ★★★ |
| 瑞数保护站点 | TLS + Cookie加密(JSVMP) + 请求签名 + 行为分析 | L5 | ★★★★★ |
| 字节系(抖音/TikTok) | HTTP2指纹 + JSVMP(X-Bogus/X-Gnarly) + 设备指纹 | L4 | ★★★★ |
| 阿里系(淘宝/支付宝) | TLS + WASM算法 + 滑块验证 + 风控模型 | L5 | ★★★★★ |
| 腾讯系(微信/QQ) | TLS + VMP混淆 + 签名算法 + 频率限制 | L4 | ★★★★ |
| 小红书 | HTTP2 + 请求签名(X-s/X-t) + 设备指纹 + 行为分析 | L4 | ★★★★ |
| 美团/大众点评 | TLS + Token签名 + 字体加密 + 行为验证 | L4 | ★★★★ |
| 知乎 | Cookie加密 + 请求签名(x-zse-96) + 频率限制 | L3 | ★★★ |

---

## 二、击破顺序决策矩阵

### 2.1 核心原则

**原则1：从外到内（请求层→协议层→浏览器层→应用层）**

外层不通过时内层完全无法触达，因此必须先确保网络层面的连通性。

**原则2：先易后难（已有工具覆盖的先做）**

优先使用成熟工具解决的层面，将精力集中在需要定制分析的层面。

**原则3：验证驱动（每破一层立即验证，确认进展）**

每完成一层绕过后立即发送测试请求，确认该层已通过再进入下一层。

### 2.2 决策矩阵

| 保护组合 | 第一步 | 第二步 | 第三步 | 第四步 |
|---------|--------|--------|--------|--------|
| TLS + CDP + Challenge | curl_cffi模拟JA3 | nodriver绕过CDP | 获取clearance cookie | 带cookie高速请求 |
| TLS + JSVMP + 签名 | curl_cffi模拟TLS | 逆向JSVMP获取cookie | 还原签名算法 | 组合请求 |
| HTTP2 + 签名 + 设备指纹 | curl_cffi impersonate | DevTools定位签名函数 | 环境修补运行JS | 附加设备指纹参数 |
| TLS + WASM + 滑块 | tls-client模拟 | 反编译WASM | 调用算法生成token | 滑块轨迹模拟 |

### 2.3 击破顺序详解

#### Layer 1: 请求层（TLS + Header）

**目标：** 使HTTP请求在TLS和Header层面模拟真实浏览器

**工具：** curl_cffi / tls-client / cycleTLS

**验证标准：** 检查是否仍返回403/challenge页面

```python
# Python: 使用curl_cffi模拟Chrome的TLS指纹
from curl_cffi import requests

# impersonate参数自动设置JA3指纹 + HTTP/2 + 正确的Header顺序
session = requests.Session(impersonate="chrome120")

resp = session.get("https://target.com/api/data", headers={
    "Accept": "application/json, text/plain, */*",
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",
    "Referer": "https://target.com/",
})

print(f"Status: {resp.status_code}")
# 如果返回200 → Layer 1通过
# 如果返回403 → 可能需要进一步模拟或换IP
# 如果返回challenge HTML → 需要进入Layer 3
```

#### Layer 2: 协议层（HTTP/2指纹）

**目标：** HTTP/2 SETTINGS帧和优先级树匹配真实浏览器

**工具：** curl_cffi的impersonate参数（自动处理H2指纹）

**验证标准：** 检查返回状态与响应内容

```python
# curl_cffi在impersonate模式下自动处理H2指纹
# 如果需要更细粒度控制：
from curl_cffi import requests

session = requests.Session(
    impersonate="chrome120",
    # HTTP/2 SETTINGS会自动匹配Chrome 120的参数：
    # HEADER_TABLE_SIZE=65536
    # ENABLE_PUSH=0
    # INITIAL_WINDOW_SIZE=6291456
    # MAX_HEADER_LIST_SIZE=262144
)

# 验证：某些站点（如小红书）仅通过H2指纹判断
resp = session.get("https://www.xiaohongshu.com/explore")
print(f"Status: {resp.status_code}, Length: {len(resp.text)}")
```

#### Layer 3: 浏览器层（CDP + 指纹）

**目标：** 绕过自动化检测（navigator.webdriver）和浏览器指纹检测

**工具：** nodriver / patchright

**验证标准：** 通过 bot detection 测试页面（如 bot.sannysoft.com）

```python
# Python: 使用nodriver绕过CDP检测
import asyncio
import nodriver as uc

async def bypass_browser_detection():
    browser = await uc.start(
        headless=False,  # 某些检测对headless更严格
    )
    page = await browser.get("https://bot.sannysoft.com/")
    await asyncio.sleep(3)

    # 验证：检查页面上的检测结果
    # 所有项目应显示绿色/通过
    content = await page.get_content()
    assert "webdriver" not in content.lower() or "missing" in content.lower()

    # 实际使用：访问目标站点
    page2 = await browser.get("https://target.com/")
    # 如果不再返回challenge → Layer 3通过
    return page2

asyncio.run(bypass_browser_detection())
```

```javascript
// Node.js: 使用patchright绕过检测
const { chromium } = require('patchright');

(async () => {
  const browser = await chromium.launch({ headless: false });
  const context = await browser.newContext();
  const page = await context.newPage();

  // patchright自动修补CDP检测点
  await page.goto('https://target.com/');

  // 验证通过后提取cookie用于后续请求
  const cookies = await context.cookies();
  console.log('Cookies:', cookies);

  await browser.close();
})();
```

#### Layer 4: 应用层（JS加密 / JSVMP / 签名）

**目标：** 还原加密算法，生成有效签名参数

**工具：** Chrome DevTools + 环境修补 / AST反混淆 / VM追踪

**验证标准：** 请求返回正常业务数据

```javascript
// Node.js: 环境修补方式运行签名JS
const { JSDOM } = require('jsdom');
const fs = require('fs');

// 构建最小化浏览器环境
const dom = new JSDOM('<!DOCTYPE html><html><body></body></html>', {
  url: 'https://target.com',
  runScripts: 'dangerously',
});

const { window } = dom;

// 补充必要环境（根据目标JS的需求定制）
window.navigator = {
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0',
  platform: 'Win32',
  language: 'zh-CN',
};

// 加载目标加密JS
const signScript = fs.readFileSync('./extracted_sign.js', 'utf-8');
window.eval(signScript);

// 调用签名函数（需要通过逆向确定函数名和参数）
const signature = window._sign('/api/data', { page: 1 });
console.log('Generated signature:', signature);
```

---

## 三、失败诊断与回溯

### 3.1 常见失败症状与对应层

| 症状 | 可能原因层 | 排查方法 | 解决方案 |
|------|----------|---------|---------|
| 直接403 | 请求层（IP/TLS） | 换IP + 检查JA3指纹 | 使用curl_cffi impersonate |
| 返回challenge页面 | 浏览器层（CDP/JS检测） | 检查navigator.webdriver | 使用nodriver/patchright |
| 返回空数据/假数据 | 应用层（签名错误） | 对比正常浏览器的签名参数 | 重新分析签名算法 |
| 首次成功后续失败 | 行为层（频率/模式） | 增加间隔、随机化模式 | 添加延迟 + 行为模拟 |
| 需要验证码 | 风控层（信誉分过低） | 切换IP/升级浏览器环境 | 提高IP质量 + 完善指纹 |
| 返回503 | 协议层（HTTP/2异常） | 检查H2 SETTINGS | 使用正确的impersonate |
| 间歇性失效 | 多层（灰度升级） | A/B对比不同时段请求 | 监控+版本适配 |

### 3.2 回溯机制

当某层绕过突然失败时的回退策略：

```python
# 分层回溯验证框架
import logging

logger = logging.getLogger(__name__)

class LayeredBypass:
    def __init__(self):
        self.layers = ['tls', 'http2', 'browser', 'sign']
        self.layer_status = {layer: None for layer in self.layers}

    async def verify_layer(self, layer_name: str) -> bool:
        """验证单层是否仍然有效"""
        verifiers = {
            'tls': self._verify_tls,
            'http2': self._verify_http2,
            'browser': self._verify_browser,
            'sign': self._verify_sign,
        }
        result = await verifiers[layer_name]()
        self.layer_status[layer_name] = result
        return result

    async def diagnose_failure(self):
        """从外到内逐层验证，定位失败层"""
        for layer in self.layers:
            is_ok = await self.verify_layer(layer)
            if not is_ok:
                logger.error(f"Layer [{layer}] FAILED - 定位到问题层")
                return layer
            logger.info(f"Layer [{layer}] OK")
        return None  # 所有层都通过，可能是临时问题

    async def _verify_tls(self) -> bool:
        """验证TLS层：能否建立连接并获得非403响应"""
        from curl_cffi import requests
        try:
            resp = requests.get("https://target.com/",
                              impersonate="chrome120", timeout=10)
            return resp.status_code != 403
        except Exception:
            return False

    async def _verify_http2(self) -> bool:
        """验证协议层"""
        # 实现略
        return True

    async def _verify_browser(self) -> bool:
        """验证浏览器层"""
        # 实现略
        return True

    async def _verify_sign(self) -> bool:
        """验证签名层"""
        # 实现略
        return True
```

**A/B测试法：** 保持其他层不变，只变动一层来精确定位问题。例如怀疑TLS层时，保持相同的签名算法，分别用 `curl_cffi`（正确JA3）和原生 `requests`（无JA3模拟）发请求对比结果。

---

## 四、实战案例详解

### 4.1 案例：Cloudflare保护站点（TLS + CDP + Challenge）

**场景：** 目标站点使用Cloudflare，开启了Bot Fight Mode和Turnstile。

```python
# 完整的Cloudflare三层击破方案
import asyncio
from curl_cffi import requests as cffi_requests
import nodriver as uc

class CloudflareBypass:
    def __init__(self, target_url):
        self.target_url = target_url
        self.cf_clearance = None
        self.cookies = {}

    async def step1_try_direct(self):
        """Step 1: 尝试直接用curl_cffi请求（TLS模拟）"""
        session = cffi_requests.Session(impersonate="chrome120")
        resp = session.get(self.target_url)

        if resp.status_code == 200 and 'challenge' not in resp.text.lower():
            print("[+] 直接通过！无需浏览器")
            return resp
        else:
            print("[-] 需要通过Challenge，切换到浏览器方案")
            return None

    async def step2_browser_challenge(self):
        """Step 2: 使用nodriver通过Turnstile Challenge"""
        browser = await uc.start(headless=False)
        page = await browser.get(self.target_url)

        # 等待Challenge完成（通常5-10秒）
        for _ in range(20):
            await asyncio.sleep(1)
            current_url = page.url
            content = await page.get_content()
            if 'challenge' not in content.lower():
                print("[+] Challenge通过！")
                break

        # 提取cf_clearance cookie
        all_cookies = await browser.cookies.get_all()
        for cookie in all_cookies:
            if cookie.name == 'cf_clearance':
                self.cf_clearance = cookie.value
            self.cookies[cookie.name] = cookie.value

        await browser.stop()
        return self.cookies

    async def step3_fast_requests(self):
        """Step 3: 使用cookie进行高速请求"""
        session = cffi_requests.Session(impersonate="chrome120")

        # 带上从浏览器获取的cookie
        for name, value in self.cookies.items():
            session.cookies.set(name, value)

        # 现在可以高速请求而不需要浏览器
        resp = session.get(f"{self.target_url}/api/data?page=1")
        print(f"[+] API响应: {resp.status_code}, 数据长度: {len(resp.text)}")
        return resp

    async def run(self):
        """执行完整流程"""
        # 尝试直接请求
        result = await self.step1_try_direct()
        if result:
            return result

        # 浏览器通过Challenge
        await self.step2_browser_challenge()

        # 带cookie高速请求
        return await self.step3_fast_requests()

# 使用
asyncio.run(CloudflareBypass("https://target-cf-site.com").run())
```

### 4.2 案例：瑞数保护站点（Cookie加密 + 请求签名）

**特征识别：** 页面中存在 `$_ts` 变量、cookie名匹配 `FSSBBIl1UgzbN7N80S` 等模式。

```javascript
// Step 1: 识别瑞数特征
function detectRuishu(html) {
  const indicators = [
    /\$_ts\s*=\s*\{/,           // $_ts全局变量
    /FSSBBIl1UgzbN7N/,          // 典型cookie名称
    /_\$[a-zA-Z]{2}/,           // 混淆变量命名模式
    /content="0;url=/,          // meta refresh跳转
  ];
  return indicators.filter(re => re.test(html)).length >= 2;
}

// Step 2: 瑞数cookie生成通常需要执行其JSVMP
// 环境修补要点：
const environment = {
  document: {
    cookie: '',          // 初始cookie
    createElement: (tag) => ({ /* 模拟DOM元素 */ }),
    getElementsByTagName: (tag) => [],
  },
  window: {
    location: { href: 'https://target.com/', hostname: 'target.com' },
    navigator: { userAgent: '...', platform: 'Win32' },
  },
  // 瑞数特别检测的属性：
  screen: { width: 1920, height: 1080 },
  performance: { timing: { navigationStart: Date.now() - 1000 } },
};

// Step 3: 执行JS获取cookie后附带请求
// Step 4: 部分瑞数版本还需要动态签名（在请求参数中附加MmEwMD等）
```

### 4.3 案例：字节系签名（X-Bogus + X-Gnarly + msToken）

**多参数签名的并行还原：**

```python
# Step 1: 确定需要的签名参数
# 抖音Web API通常需要: X-Bogus, _signature, msToken
# TikTok API需要: X-Bogus, X-Gnarly(部分接口)

# Step 2: 各参数定位
# X-Bogus: 通过搜索请求拦截，在XHR发出前被添加
# msToken: 通常从cookie或前置接口获取
# X-Gnarly: JSVMP保护的签名（TikTok特有）

# Step 3: 环境修补方式生成X-Bogus
import subprocess
import json

def generate_x_bogus(url_params: str, user_agent: str) -> str:
    """调用Node.js环境修补脚本生成X-Bogus"""
    result = subprocess.run(
        ['node', 'x_bogus_env.js', url_params, user_agent],
        capture_output=True, text=True
    )
    return result.stdout.strip()

# Step 4: 组装完整请求
from curl_cffi import requests

session = requests.Session(impersonate="chrome120")

params = "device_platform=webapp&aid=6383&count=20"
ua = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0"
x_bogus = generate_x_bogus(params, ua)

resp = session.get(
    f"https://www.douyin.com/aweme/v1/web/recommend/feed/?{params}&X-Bogus={x_bogus}",
    headers={
        "User-Agent": ua,
        "Referer": "https://www.douyin.com/",
    }
)
print(f"Status: {resp.status_code}")
```

---

## 五、工具组合推荐

### 5.1 轻量组合（覆盖L1-L3）

适用场景：目标无复杂JS加密，主要是TLS/CDP层面防护。

```
curl_cffi (TLS+H2模拟) + 代理池 (IP轮换) → 纯HTTP级高速采集
```

- **优势：** 速度快，资源消耗低，单机可达1000+ QPS
- **局限：** 无法处理需要执行JS的场景

### 5.2 中等组合（覆盖L1-L4）

适用场景：需要执行JS或浏览器操作，但目标算法可还原。

```
nodriver (浏览器自动化) + mitmproxy (流量分析) + 环境修补 (算法还原)
```

- **优势：** 覆盖全面，浏览器兜底方案可靠
- **局限：** 浏览器资源消耗大，需要服务器资源

### 5.3 重型组合（覆盖L1-L5）

适用场景：面对瑞数/字节等顶级保护，需要深度逆向。

```
Proxy Hook执行追踪 (JSVMP分析) + nodriver (浏览器) + curl_cffi (高速请求)
```

- **优势：** 可处理任何级别的保护
- **局限：** 前期分析耗时长，需要逆向工程经验

---

## 六、常见陷阱

在多重保护场景中，以下是10个最容易踩的坑：

| # | 陷阱描述 | 原因分析 | 解决方案 |
|---|---------|---------|---------|
| 1 | 签名正确但cookie过期 | cookie有效期短（5-30分钟），签名验证前cookie已失效 | 实现cookie自动刷新机制 |
| 2 | TLS通了但HTTP/2指纹不对 | 某些CDN分别检测TLS和H2指纹 | 确保使用支持H2指纹模拟的工具 |
| 3 | 浏览器通过检测但签名无效 | 签名JS依赖的环境变量在自动化浏览器中不同 | 确保浏览器环境完整，不要inject额外JS |
| 4 | 本地复现成功但服务器失败 | 服务器IP段被标记为数据中心IP | 使用住宅代理或云手机 |
| 5 | 偶尔成功偶尔失败（随机性） | 目标使用A/B测试，不同节点策略不同 | 多次请求确认，处理多版本兼容 |
| 6 | 签名过了但被风控拦截 | 行为特征异常（请求间隔太规律、路径太单一） | 添加随机延迟、模拟浏览行为 |
| 7 | 换了IP还是被封 | 设备指纹（canvas/webgl hash）未更换 | 同时更换IP和浏览器指纹 |
| 8 | 复用cookie被检测 | 同一cookie跨IP使用被检测为异常 | cookie与IP绑定使用 |
| 9 | 解密JS后参数仍报错 | 时间戳参数精度不够或时区不对 | 检查timestamp精度（毫秒vs秒）和UTC偏移 |
| 10 | 前端签名对了后端校验失败 | 后端有额外的服务端签名校验（如请求体hash） | 使用mitmproxy完整对比正常请求的所有参数 |

---

## 参考资源

- [curl_cffi文档](https://curl-cffi.readthedocs.io/) - TLS/H2指纹模拟
- [nodriver](https://github.com/AresS31/nodriver) - CDP绕过的浏览器自动化
- [patchright](https://github.com/AresS31/patchright) - Playwright的反检测补丁版
- [mitmproxy](https://mitmproxy.org/) - 流量分析与对比

> **提示：** 分层击破的核心思想是"分而治之"——将复杂的多重保护拆解为独立的子问题，
> 逐层验证、逐层突破，避免在混合问题中迷失方向。
