# mitmproxy 网络拦截与分析实战指南

> mitmproxy 是 Web 逆向中最强大的 MITM（中间人）代理工具。
> 本文档覆盖从基础抓包到高级脚本编写的完整工作流。

---

## 目录

- [一、安装与基础配置](#一安装与基础配置)
- [二、基础抓包](#二基础抓包)
- [三、脚本API核心](#三脚本api核心)
- [四、逆向工程专用脚本](#四逆向工程专用脚本)
- [五、与其他工具集成](#五与其他工具集成)
- [六、常见问题](#六常见问题)

---

## 一、安装与基础配置

### 1.1 安装

```bash
# pip安装（推荐Python 3.9+）
pip install mitmproxy

# 验证安装
mitmproxy --version
```

### 1.2 三大组件区别

| 组件 | 用途 | 特点 |
|------|------|------|
| `mitmproxy` | 交互式终端UI | 适合手动分析，支持键盘操作 |
| `mitmdump` | 命令行工具 | 适合脚本自动化，无UI开销 |
| `mitmweb` | Web界面 | 浏览器中查看流量，适合团队协作 |

### 1.3 证书安装

```bash
# 启动一次mitmproxy后，证书自动生成在 ~/.mitmproxy/
# 浏览器信任：导入 mitmproxy-ca-cert.pem 到"受信任的根证书颁发机构"
# Windows系统级：
certutil -addstore root %USERPROFILE%\.mitmproxy\mitmproxy-ca-cert.cer
# macOS：
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.mitmproxy/mitmproxy-ca-cert.pem
```

### 1.4 代理配置

```bash
# 系统代理（Windows）
netsh winhttp set proxy 127.0.0.1:8080
# 环境变量方式
export HTTP_PROXY=http://127.0.0.1:8080
export HTTPS_PROXY=http://127.0.0.1:8080
```

---

## 二、基础抓包

### 2.1 启动命令

```bash
# 基础启动（默认监听8080端口）
mitmproxy -p 8080

# 指定监听地址
mitmdump --listen-host 0.0.0.0 -p 8888

# 带脚本启动
mitmdump -s my_script.py -p 8080

# 只抓取特定域名
mitmdump --set flow_detail=2 -p 8080
```

### 2.2 过滤表达式

```bash
# 常用过滤语法（在mitmproxy交互界面中按f输入）
~d example.com          # 匹配域名
~u /api/               # URL路径包含
~m POST                # HTTP方法
~s                     # 仅响应
~q                     # 仅请求
~h "Content-Type: json" # Header匹配
~b "password"          # Body包含
~c 200                 # 状态码

# 组合使用
~d api.example.com & ~m POST & ~u /login
```

### 2.3 HTTPS解密原理

mitmproxy 的 HTTPS 解密通过动态证书签发实现：
1. 客户端连接代理，发起 CONNECT 请求
2. mitmproxy 用自己的 CA 为目标域名签发证书
3. 客户端信任 mitmproxy CA → 接受伪造证书
4. mitmproxy 与真实服务器建立独立 TLS 连接

---

## 三、脚本API核心

### 3.1 请求拦截与修改

```python
"""addon基本结构 - 保存为 modify_request.py"""
from mitmproxy import http

class RequestModifier:
    def request(self, flow: http.HTTPFlow) -> None:
        """拦截所有请求"""
        # 修改Header
        flow.request.headers["X-Custom-Header"] = "injected-value"
        
        # 修改User-Agent
        flow.request.headers["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        
        # 修改请求Body（POST请求）
        if flow.request.method == "POST" and b"token" in (flow.request.content or b""):
            import json
            body = json.loads(flow.request.content)
            body["token"] = "my_custom_token"
            flow.request.content = json.dumps(body).encode()

    def response(self, flow: http.HTTPFlow) -> None:
        """拦截所有响应"""
        # 移除安全Header
        flow.response.headers.pop("X-Frame-Options", None)
        flow.response.headers.pop("Content-Security-Policy", None)

addons = [RequestModifier()]
```

**启动方式：** `mitmdump -s modify_request.py`

### 3.2 响应篡改

```python
"""向HTML页面注入Hook脚本"""
from mitmproxy import http

INJECT_JS = """
<script>
// Hook XMLHttpRequest
(function() {
    const originalOpen = XMLHttpRequest.prototype.open;
    const originalSend = XMLHttpRequest.prototype.send;
    
    XMLHttpRequest.prototype.open = function(method, url) {
        this._url = url;
        this._method = method;
        console.log('[Hook] XHR Open:', method, url);
        return originalOpen.apply(this, arguments);
    };
    
    XMLHttpRequest.prototype.send = function(body) {
        console.log('[Hook] XHR Send:', this._method, this._url, body);
        return originalSend.apply(this, arguments);
    };
})();
</script>
"""

class JSInjector:
    def response(self, flow: http.HTTPFlow) -> None:
        if flow.response and "text/html" in flow.response.headers.get("content-type", ""):
            html = flow.response.text
            # 在</head>前注入
            flow.response.text = html.replace("</head>", INJECT_JS + "</head>")

addons = [JSInjector()]
```

### 3.3 WebSocket拦截

```python
"""WebSocket消息记录与分析"""
import json
from mitmproxy import http, websocket

class WebSocketLogger:
    def __init__(self):
        self.messages = []

    def websocket_message(self, flow: http.HTTPFlow):
        assert flow.websocket is not None
        message = flow.websocket.messages[-1]
        
        direction = "→ Server" if message.from_client else "← Client"
        content = message.content
        
        # 尝试JSON解析
        try:
            parsed = json.loads(content)
            print(f"[WS {direction}] JSON: {json.dumps(parsed, indent=2, ensure_ascii=False)[:200]}")
        except (json.JSONDecodeError, UnicodeDecodeError):
            print(f"[WS {direction}] Binary: {content[:50].hex()}")
        
        self.messages.append({
            "direction": direction,
            "content": content.decode("utf-8", errors="replace") if isinstance(content, bytes) else content,
            "timestamp": message.timestamp
        })

    def done(self):
        """脚本结束时保存所有消息"""
        with open("ws_messages.json", "w", encoding="utf-8") as f:
            json.dump(self.messages, f, ensure_ascii=False, indent=2)

addons = [WebSocketLogger()]
```

### 3.4 条件匹配

```python
"""精确匹配目标请求"""
import re
from mitmproxy import http

class ConditionalFilter:
    # 目标URL模式
    URL_PATTERN = re.compile(r"/api/v\d+/sign")
    # 目标域名
    TARGET_DOMAINS = ["api.target.com", "m.target.com"]

    def request(self, flow: http.HTTPFlow) -> None:
        # 域名匹配
        if flow.request.host not in self.TARGET_DOMAINS:
            return
        # URL正则匹配
        if not self.URL_PATTERN.search(flow.request.path):
            return
        # Header匹配
        if "X-Sign" not in flow.request.headers:
            return
        
        print(f"[Match] {flow.request.method} {flow.request.url}")
        print(f"  Sign: {flow.request.headers['X-Sign']}")
        print(f"  Body: {flow.request.text[:200]}")

addons = [ConditionalFilter()]
```

### 3.5 流量录制与回放

```bash
# 录制流量到文件
mitmdump -w traffic.flow -p 8080

# 从文件回放
mitmdump -nC traffic.flow

# 回放时修改（结合脚本）
mitmdump -nC traffic.flow -s modify_replay.py

# 导出为HAR格式（mitmproxy交互界面中）
# 按 E 导出，选择 HAR 格式

# 编程方式导出HAR
from mitmproxy import io
from mitmproxy.tools.web import export

# 读取flow文件
with open("traffic.flow", "rb") as f:
    reader = io.FlowReader(f)
    flows = list(reader.stream())
```

---

## 四、逆向工程专用脚本

### 4.1 自动提取加密参数

```python
"""从请求中提取签名参数并保存"""
import json
import time
from mitmproxy import http

class SignExtractor:
    def __init__(self):
        self.records = []
        self.output_file = "extracted_signs.jsonl"

    def request(self, flow: http.HTTPFlow) -> None:
        sign_headers = ["X-Sign", "X-Signature", "X-Token", "Authorization"]
        extracted = {}
        
        for header in sign_headers:
            if header in flow.request.headers:
                extracted[header] = flow.request.headers[header]
        
        # 从URL参数提取
        for key in ["sign", "sig", "token", "nonce", "timestamp"]:
            value = flow.request.query.get(key)
            if value:
                extracted[f"query_{key}"] = value
        
        if extracted:
            record = {
                "time": time.strftime("%Y-%m-%d %H:%M:%S"),
                "url": flow.request.url,
                "method": flow.request.method,
                "params": extracted,
                "body": flow.request.text[:500] if flow.request.text else None
            }
            with open(self.output_file, "a", encoding="utf-8") as f:
                f.write(json.dumps(record, ensure_ascii=False) + "\n")
            print(f"[Extract] {flow.request.path} → {list(extracted.keys())}")

addons = [SignExtractor()]
```

### 4.2 Cookie同步脚本

```python
"""浏览器Cookie自动同步到API请求"""
import json
from pathlib import Path
from mitmproxy import http

COOKIE_FILE = Path("cookies.json")

class CookieSync:
    def __init__(self):
        self.cookies = {}
        self._load_cookies()

    def _load_cookies(self):
        if COOKIE_FILE.exists():
            self.cookies = json.loads(COOKIE_FILE.read_text())

    def request(self, flow: http.HTTPFlow) -> None:
        # 如果是浏览器发出的请求（带完整cookie），保存cookie
        if "text/html" in flow.request.headers.get("accept", ""):
            cookie_str = flow.request.headers.get("cookie", "")
            if cookie_str:
                for pair in cookie_str.split("; "):
                    if "=" in pair:
                        k, v = pair.split("=", 1)
                        self.cookies[k.strip()] = v.strip()
                COOKIE_FILE.write_text(json.dumps(self.cookies, indent=2))
        
        # 对API请求注入最新cookie
        elif "/api/" in flow.request.path and self.cookies:
            cookie_str = "; ".join(f"{k}={v}" for k, v in self.cookies.items())
            flow.request.headers["cookie"] = cookie_str

addons = [CookieSync()]
```

### 4.3 请求/响应日志分析

```python
"""按API分组统计请求频率和参数变化"""
import json
import time
from collections import defaultdict
from mitmproxy import http

class APIAnalyzer:
    def __init__(self):
        self.stats = defaultdict(lambda: {"count": 0, "params": [], "timestamps": []})

    def request(self, flow: http.HTTPFlow) -> None:
        path = flow.request.path.split("?")[0]  # 去掉query string
        key = f"{flow.request.method} {path}"
        
        self.stats[key]["count"] += 1
        self.stats[key]["timestamps"].append(time.time())
        
        # 收集参数变化
        params = dict(flow.request.query)
        if flow.request.text:
            try:
                params.update(json.loads(flow.request.text))
            except json.JSONDecodeError:
                pass
        self.stats[key]["params"].append(params)

    def done(self):
        """脚本结束时输出统计报告"""
        print("\n" + "=" * 60)
        print("API 请求统计报告")
        print("=" * 60)
        for api, info in sorted(self.stats.items(), key=lambda x: x[1]["count"], reverse=True):
            print(f"\n{api} (共 {info['count']} 次)")
            # 分析参数变化
            if info["params"]:
                all_keys = set()
                for p in info["params"]:
                    all_keys.update(p.keys())
                changing_keys = [k for k in all_keys if len(set(str(p.get(k)) for p in info["params"])) > 1]
                if changing_keys:
                    print(f"  变化参数: {changing_keys}")

addons = [APIAnalyzer()]
```

### 4.4 签名验证脚本

```python
"""拦截请求，用本地算法重新生成签名，对比验证"""
import hashlib
import hmac
import json
import time
from urllib.parse import urlencode
from mitmproxy import http

SECRET_KEY = "your_discovered_secret_key"

class SignatureVerifier:
    def request(self, flow: http.HTTPFlow) -> None:
        if "X-Sign" not in flow.request.headers:
            return
        
        original_sign = flow.request.headers["X-Sign"]
        
        # 本地重新计算签名（根据你逆向出的算法）
        params = dict(flow.request.query)
        timestamp = params.get("t", str(int(time.time())))
        nonce = params.get("nonce", "")
        
        # 示例签名算法：HMAC-SHA256(排序参数 + timestamp + nonce)
        sign_params = {k: v for k, v in sorted(params.items()) if k not in ["sign", "sig"]}
        sign_string = urlencode(sign_params) + timestamp + nonce
        
        computed_sign = hmac.new(
            SECRET_KEY.encode(),
            sign_string.encode(),
            hashlib.sha256
        ).hexdigest()
        
        match = "✓" if computed_sign == original_sign else "✗"
        print(f"[{match}] {flow.request.path}")
        if computed_sign != original_sign:
            print(f"  Original: {original_sign}")
            print(f"  Computed: {computed_sign}")
            print(f"  SignStr:  {sign_string[:100]}")

addons = [SignatureVerifier()]
```

---

## 五、与其他工具集成

### 5.1 mitmproxy + Chrome

```bash
# 启动Chrome并指定代理
"C:\Program Files\Google\Chrome\Application\chrome.exe" --proxy-server=http://127.0.0.1:8080 --ignore-certificate-errors

# 或使用独立用户目录（不影响日常浏览）
chrome --proxy-server=http://127.0.0.1:8080 --user-data-dir=/tmp/chrome-proxy
```

### 5.2 mitmproxy + nodriver

```python
"""使用nodriver + mitmproxy进行自动化逆向"""
import nodriver as uc

async def main():
    browser = await uc.start(
        browser_args=[
            "--proxy-server=http://127.0.0.1:8080",
            "--ignore-certificate-errors"
        ]
    )
    page = await browser.get("https://target.com")
    # nodriver操作触发请求 → mitmproxy脚本自动抓取分析
    await page.wait(5)

if __name__ == "__main__":
    uc.loop().run_until_complete(main())
```

### 5.3 mitmproxy + Node.js（IPC通信）

```python
"""mitmproxy脚本通过HTTP与Node.js进程通信"""
import json
import urllib.request
from mitmproxy import http

NODE_SERVER = "http://127.0.0.1:3456"

class NodeBridge:
    def request(self, flow: http.HTTPFlow) -> None:
        if "X-Sign" in flow.request.headers:
            # 将签名参数发送给Node.js进行验证/重新生成
            data = json.dumps({
                "url": flow.request.url,
                "sign": flow.request.headers["X-Sign"],
                "params": dict(flow.request.query)
            }).encode()
            
            req = urllib.request.Request(
                f"{NODE_SERVER}/verify",
                data=data,
                headers={"Content-Type": "application/json"}
            )
            try:
                resp = urllib.request.urlopen(req, timeout=5)
                result = json.loads(resp.read())
                print(f"[Node] Verify result: {result}")
            except Exception as e:
                print(f"[Node] Error: {e}")

addons = [NodeBridge()]
```

---

## 六、常见问题

### 6.1 证书信任问题

**问题**：浏览器提示 `NET::ERR_CERT_AUTHORITY_INVALID`
**解决**：
1. 确认已正确安装 mitmproxy CA 证书到系统/浏览器
2. Chrome：设置 → 隐私和安全 → 安全 → 管理证书 → 导入
3. Firefox：需单独导入（使用自己的证书存储）

### 6.2 HSTS绕过

**问题**：目标站点使用 HSTS，浏览器拒绝不安全连接
**解决**：
- Chrome启动参数添加：`--ignore-certificate-errors`
- 或清除 HSTS 缓存：`chrome://net-internals/#hsts` → Delete domain

### 6.3 双向TLS（mTLS）处理

**问题**：服务器要求客户端证书
**解决**：
```bash
# 指定客户端证书
mitmdump --set client_certs=/path/to/client-cert.pem
```

### 6.4 HTTP/2支持

```bash
# mitmproxy默认支持HTTP/2，如需禁用：
mitmdump --set http2=false
```

### 6.5 上游代理

```bash
# 通过另一个代理连接（如Burp Suite串联）
mitmdump --mode upstream:http://127.0.0.1:8081 -p 8080
```
