# HTTP/2 & QUIC 协议层指纹分析与绕过

> 协议层指纹是TLS指纹（JA3/JA4）之后的新一代反爬检测前沿。
> 本文档覆盖HTTP/2 SETTINGS帧指纹、优先级树指纹、QUIC报文指纹的原理与绕过。

## 目录

- [一、为什么协议层指纹重要](#一为什么协议层指纹重要)
- [二、HTTP/2 指纹原理](#二http2-指纹原理)
- [三、HTTP/2 指纹绕过方案](#三http2-指纹绕过方案)
- [四、QUIC/HTTP/3 指纹原理](#四quichttp3-指纹原理)
- [五、QUIC/HTTP/3 绕过方案](#五quichttp3-绕过方案)
- [六、检测与验证工具](#六检测与验证工具)
- [七、实战：模拟Chrome的完整HTTP/2指纹](#七实战模拟chrome的完整http2指纹)

---

## 一、为什么协议层指纹重要

### 超越TLS层的指纹检测

传统的TLS指纹（JA3/JA4）已被广泛了解和绕过，但HTTP/2连接建立后的 **SETTINGS帧参数** 同样能形成独特指纹。即使TLS指纹完美模拟了Chrome，如果HTTP/2层参数暴露为Python的`httpx`或Go的`net/http`，请求仍会被标记。

协议层指纹检测链：

```
TCP → TLS (JA3/JA4) → HTTP/2 (SETTINGS/PRIORITY/PSEUDO-HEADERS) → 应用层
         ↑ 已被重视            ↑ 新一代检测前沿
```

### 2024-2026 行业趋势

| CDN/WAF 提供商 | HTTP/2 指纹检测 | 部署时间 |
|---------------|----------------|---------|
| Cloudflare | ✅ 已部署（Bot Management） | 2023 Q4 |
| Akamai | ✅ 已部署（Bot Manager Premier） | 2024 Q1 |
| DataDome | ✅ 已部署 | 2024 Q2 |
| Kasada | ✅ 已部署 | 2023 |
| PerimeterX/HUMAN | ✅ 已部署 | 2024 |

**结论**：仅模拟TLS指纹已不够，必须同时模拟HTTP/2层指纹才能通过现代WAF检测。

---

## 二、HTTP/2 指纹原理

### 2.1 SETTINGS 帧指纹

HTTP/2连接建立后，客户端发送的第一个帧是 SETTINGS 帧，包含6个关键参数。不同浏览器/HTTP客户端的默认值和参数顺序各不相同：

| 参数 ID | 参数名 | Chrome 131 | Firefox 133 | Safari 18 | curl 8.7 | Python httpx |
|---------|--------|-----------|-------------|-----------|-----------|-------------|
| 0x1 | HEADER_TABLE_SIZE | 65536 | 65536 | 4096 | 4096 | 4096 |
| 0x2 | ENABLE_PUSH | 0 | 0 | 0 | 0 | 0 |
| 0x3 | MAX_CONCURRENT_STREAMS | 1000 | 100 | 100 | 100 | 100 |
| 0x4 | INITIAL_WINDOW_SIZE | 6291456 | 131072 | 2097152 | 65535 | 65535 |
| 0x5 | MAX_FRAME_SIZE | 16384 | 16384 | 16384 | 16384 | 16384 |
| 0x6 | MAX_HEADER_LIST_SIZE | 262144 | 65536 | _(不发送)_ | _(不发送)_ | _(不发送)_ |

**参数发送顺序同样是指纹特征：**

```
Chrome:  [1, 2, 3, 4, 5, 6]    → HEADER_TABLE_SIZE 在前
Firefox: [1, 2, 4, 3]           → 只发送4个参数，顺序不同
Safari:  [3, 4, 1, 5]           → 完全不同的顺序
curl:    [3, 4, 2]              → 最少的参数集
```

### 2.2 WINDOW_UPDATE 帧指纹

紧随 SETTINGS 帧之后，客户端通常会发送一个 WINDOW_UPDATE 帧来调整连接级流控窗口：

| 客户端 | 连接级 WINDOW_UPDATE 值 |
|--------|----------------------|
| Chrome 131 | 15663105 (即 15728640 - 65535) |
| Firefox 133 | 12517377 (即 12582912 - 65535) |
| Safari 18 | 10420225 (即 10485760 - 65535) |
| curl | 不发送（使用默认65535） |
| Python httpx | 不发送 |

计算方式：`目标窗口大小 - 默认窗口大小(65535) = WINDOW_UPDATE增量`

### 2.3 优先级树（PRIORITY）指纹

Chrome（直到版本130左右）使用分组优先级策略：

```
# Chrome 的优先级分组（已废弃但仍可检测）
Stream 0 (root)
├── Weight 256, Stream Group "leaders" (images, CSS)
├── Weight 220, Stream Group "followers" (JS async)
└── Weight 183, Stream Group "background" (prefetch)
```

Firefox 使用增量分配策略：

```
# Firefox 的FIFO优先级
Stream 1: depends_on=0, weight=41, exclusive=false
Stream 3: depends_on=0, weight=41, exclusive=false
Stream 5: depends_on=0, weight=41, exclusive=false
```

> **注意**：HTTP/2 PRIORITY 帧在 RFC 9218 中已被废弃，Chrome 131+ 使用 PRIORITY_UPDATE（Extensible Priorities）。但许多检测系统仍检查是否发送了旧的PRIORITY帧。

### 2.4 伪头部顺序指纹

HTTP/2请求中的伪头部（pseudo-headers）发送顺序也是指纹：

| 客户端 | 伪头部顺序 |
|--------|-----------|
| Chrome | `:method`, `:authority`, `:scheme`, `:path` |
| Firefox | `:method`, `:path`, `:authority`, `:scheme` |
| Safari | `:method`, `:scheme`, `:path`, `:authority` |
| curl | `:method`, `:path`, `:scheme`, `:authority` |
| Go net/http | `:method`, `:url`, `:authority`, `:scheme` _(注意：可能非标准)_ |

```python
# 检测脚本示例：解析HTTP/2 HEADERS帧中的伪头部顺序
import h2.events

def check_pseudo_header_order(headers):
    """检查伪头部顺序是否匹配Chrome"""
    pseudo_headers = [h for h, v in headers if h.startswith(':')]
    chrome_order = [':method', ':authority', ':scheme', ':path']
    return pseudo_headers == chrome_order
```

---

## 三、HTTP/2 指纹绕过方案

### 3.1 curl_cffi 方案（推荐）

[curl_cffi](https://github.com/yifeikong/curl_cffi) 是目前最成熟的HTTP/2指纹模拟方案，底层基于curl-impersonate：

```python
# 安装
# pip install curl_cffi

from curl_cffi import requests

# 使用Chrome浏览器指纹配置文件
response = requests.get(
    "https://tls.peet.ws/api/all",
    impersonate="chrome131",  # 模拟Chrome 131的完整指纹
)

print(response.json()["http2"]["sent_settings"])
# 输出将完全匹配真实Chrome的SETTINGS参数
```

支持的浏览器配置文件：

| 配置文件名 | HTTP/2 SETTINGS | WINDOW_UPDATE | 伪头部顺序 |
|-----------|----------------|---------------|-----------|
| `chrome131` | ✅ | ✅ | ✅ |
| `chrome124` | ✅ | ✅ | ✅ |
| `firefox133` | ✅ | ✅ | ✅ |
| `safari18_0` | ✅ | ✅ | ✅ |
| `edge131` | ✅ | ✅ | ✅ |

```python
# 完整示例：带会话保持的请求
from curl_cffi import requests

session = requests.Session(impersonate="chrome131")

# 会话内所有请求都使用一致的HTTP/2指纹
resp1 = session.get("https://example.com/api/init")
resp2 = session.post("https://example.com/api/data", json={"key": "value"})
```

### 3.2 自定义HTTP/2客户端方案

#### Go: 使用 utls + http2

```go
package main

import (
    "crypto/tls"
    "fmt"
    "io"
    "net/http"

    tls2 "github.com/refraction-networking/utls"
    "golang.org/x/net/http2"
)

func main() {
    // 配置HTTP/2 Transport以模拟Chrome
    t2 := &http2.Transport{
        // Chrome 131 的 SETTINGS 参数
        InitialWindowSize:     6291456,
        MaxHeaderListSize:     262144,
        MaxFrameSize:          16384,
        MaxConcurrentStreams:   1000,
        HeaderTableSize:       65536,
        // 连接级窗口大小
        ConnectionFlow: 15728640,
    }

    client := &http.Client{Transport: t2}
    resp, err := client.Get("https://tls.peet.ws/api/all")
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    fmt.Println(string(body))
}
```

#### Node.js: 使用内置 http2 模块

```javascript
const http2 = require('http2');

// 模拟 Chrome 131 的 SETTINGS
const client = http2.connect('https://example.com', {
  settings: {
    headerTableSize: 65536,
    enablePush: false,
    maxConcurrentStreams: 1000,
    initialWindowSize: 6291456,
    maxFrameSize: 16384,
    maxHeaderListSize: 262144,
  },
  // 连接级窗口大小
  // Node.js http2模块会自动发送WINDOW_UPDATE
});

const req = client.request({
  // Chrome的伪头部顺序
  ':method': 'GET',
  ':authority': 'example.com',
  ':scheme': 'https',
  ':path': '/api/data',
});

req.on('response', (headers) => {
  console.log('Status:', headers[':status']);
});

let data = '';
req.on('data', (chunk) => { data += chunk; });
req.on('end', () => {
  console.log(data);
  client.close();
});
req.end();
```

### 3.3 真实浏览器方案

使用CDP（Chrome DevTools Protocol）直接控制浏览器发起请求，天然绕过所有协议层指纹：

```javascript
// 使用 Playwright
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext();
  const page = await context.newPage();

  // 拦截并获取响应数据
  const response = await page.goto('https://target-api.com/data');
  const data = await response.json();

  // 或通过 page.evaluate 发起 fetch 请求
  const apiData = await page.evaluate(async () => {
    const resp = await fetch('/api/protected-endpoint');
    return resp.json();
  });

  console.log(apiData);
  await browser.close();
})();
```

---

## 四、QUIC/HTTP/3 指纹原理

### 4.1 QUIC Initial Packet 结构

QUIC协议的初始报文（Initial Packet）包含多个可用于指纹识别的字段：

```
QUIC Initial Packet:
┌─────────────────────────────────────────┐
│ Header Form (1 bit): 1 (Long Header)    │
│ Fixed Bit (1 bit): 1                    │
│ Long Packet Type (2 bits): 0 (Initial)  │
│ Version (32 bits): 0x00000001 (v1)      │
│ DCID Length (8 bits)                    │ ← 指纹特征
│ DCID (variable)                         │
│ SCID Length (8 bits)                    │ ← 指纹特征
│ SCID (variable)                         │
│ Token Length (variable)                 │
│ Token (variable)                        │
│ Packet Length (variable)                │
│ Packet Number (variable)                │
└─────────────────────────────────────────┘
```

各浏览器的QUIC实现差异：

| 特征 | Chrome (Chromium QUIC) | Firefox (neqo) | Safari (Apple QUIC) |
|------|----------------------|----------------|-------------------|
| DCID 长度 | 8 bytes | 16 bytes | 8 bytes |
| SCID 长度 | 0 bytes (Initial) | 8 bytes | 0 bytes |
| QUIC版本 | v1 (0x00000001) | v1 | v1 |
| Padding策略 | 填充至1200 bytes | 填充至1200 bytes | 填充至1200 bytes |
| 加密套件 | AES-128-GCM优先 | AES-128-GCM/ChaCha20 | AES-128-GCM |

### 4.2 Transport Parameters 指纹

QUIC的传输参数（类似HTTP/2的SETTINGS）也形成指纹：

| 参数 | Chrome | Firefox | 含义 |
|------|--------|---------|------|
| max_idle_timeout | 30000ms | 30000ms | 最大空闲超时 |
| initial_max_data | 15728640 | 12582912 | 连接级流控 |
| initial_max_stream_data_bidi_local | 6291456 | 131072 | 双向流本地流控 |
| initial_max_stream_data_bidi_remote | 6291456 | 131072 | 双向流远端流控 |
| initial_max_stream_data_uni | 6291456 | 131072 | 单向流流控 |
| initial_max_streams_bidi | 100 | 16 | 最大双向流数 |
| initial_max_streams_uni | 3 | 16 | 最大单向流数 |
| active_connection_id_limit | 4 | 8 | 连接ID限制 |

### 4.3 QUIC 检测工具

#### Wireshark 分析 QUIC 流量

```bash
# 设置Chrome导出TLS密钥（用于Wireshark解密QUIC）
# Linux/Mac:
export SSLKEYLOGFILE=/tmp/quic_keys.log

# Windows PowerShell:
$env:SSLKEYLOGFILE = "C:\temp\quic_keys.log"

# 启动Chrome
google-chrome --enable-quic --origin-to-force-quic-on=example.com:443

# Wireshark过滤器
# quic                          - 所有QUIC流量
# quic.frame_type == 0x06       - CRYPTO帧（含Transport Parameters）
# quic.long.packet_type == 0    - Initial Packets
```

#### quic-tracker 工具

```bash
# 使用 quic-tracker 测试目标站的QUIC实现
git clone https://github.com/nickolasgoodman/quic-tracker
cd quic-tracker
go build ./...

# 运行测试
./quic-tracker -host example.com -port 443
```

---

## 五、QUIC/HTTP/3 绕过方案

### 5.1 当前方案限制

QUIC指纹模拟的生态远不如HTTP/2成熟：

| 工具/库 | HTTP/3支持 | 指纹模拟 | 成熟度 |
|---------|-----------|---------|--------|
| curl_cffi | 实验性 (curl 8.7+) | 部分 | ⭐⭐ |
| Go quic-go | 完整 | 需手动配置 | ⭐⭐⭐ |
| Python aioquic | 完整 | 需手动配置 | ⭐⭐ |
| 真实浏览器 | 完整 | 天然匹配 | ⭐⭐⭐⭐⭐ |

### 5.2 可行方案

#### 方案1：降级到HTTP/2（推荐）

大多数网站同时支持HTTP/2和HTTP/3，且通过`Alt-Svc`头协商升级。只要不主动协商HTTP/3，就不会暴露QUIC指纹：

```python
from curl_cffi import requests

# curl_cffi 默认不启用HTTP/3，自动使用HTTP/2
resp = requests.get("https://example.com", impersonate="chrome131")
print(resp.http_version)  # 输出: 2 (HTTP/2)
```

#### 方案2：使用真实浏览器

```python
# Playwright 自动处理QUIC/HTTP/3协商
from playwright.async_api import async_playwright

async def fetch_with_http3():
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            args=['--enable-quic', '--origin-to-force-quic-on=target.com:443']
        )
        page = await browser.new_page()
        resp = await page.goto('https://target.com/api')
        # 此请求将使用HTTP/3，指纹完全匹配真实Chrome
        data = await resp.json()
        await browser.close()
        return data
```

#### 方案3：Go cloudflare-quic-go

```go
import (
    "github.com/quic-go/quic-go"
    "github.com/quic-go/quic-go/http3"
)

// 自定义Transport Parameters以模拟Chrome
transport := &quic.Transport{
    // 配置连接参数
}

roundTripper := &http3.RoundTripper{
    QuicConfig: &quic.Config{
        MaxIdleTimeout:        30 * time.Second,
        InitialMaxData:        15728640,
        InitialMaxStreamDataBidiLocal:  6291456,
        InitialMaxStreamDataBidiRemote: 6291456,
        MaxIncomingStreams:     100,
        MaxIncomingUniStreams:  3,
    },
}
```

---

## 六、检测与验证工具

### 6.1 自查工具

| 工具 | URL | 检测内容 |
|------|-----|---------|
| TLS Peet | https://tls.peet.ws/api/all | TLS + HTTP/2完整指纹（JSON） |
| http2.pro | https://http2.pro/doc/api | HTTP/2支持检测 |
| Scrapfly FP | https://tools.scrapfly.io/api/fp/anything | 综合指纹检测 |

```bash
# 使用curl验证HTTP/2指纹
curl -s https://tls.peet.ws/api/all | python -m json.tool | grep -A 20 "http2"

# 输出示例（curl默认指纹）:
# "http2": {
#   "sent_settings": {
#     "3": 100,    ← MAX_CONCURRENT_STREAMS
#     "4": 65535,  ← INITIAL_WINDOW_SIZE（明显是curl特征！）
#     "2": 0       ← ENABLE_PUSH
#   }
# }
```

### 6.2 抓包分析

#### Wireshark HTTP/2 过滤器

```
# 基础HTTP/2过滤
http2

# 只看SETTINGS帧
http2.type == 4

# 只看客户端发送的SETTINGS
http2.type == 4 && http2.flags.ack == 0

# 查看WINDOW_UPDATE帧
http2.type == 8

# 查看PRIORITY帧
http2.type == 2

# 查看HEADERS帧（含伪头部）
http2.type == 1
```

#### Chrome net-internals

```
# 在Chrome地址栏输入：
chrome://net-internals/#http2

# 可查看：
# - 当前所有HTTP/2会话
# - 每个会话的SETTINGS参数
# - 流的创建和优先级
# - WINDOW_UPDATE历史
```

---

## 七、实战：模拟Chrome的完整HTTP/2指纹

### 完整代码示例

```python
"""
完整模拟Chrome 131的HTTP/2指纹
使用curl_cffi实现，并通过tls.peet.ws验证
"""
from curl_cffi import requests
import json

def verify_fingerprint():
    """验证HTTP/2指纹是否匹配Chrome"""

    # 使用Chrome 131配置
    session = requests.Session(impersonate="chrome131")
    resp = session.get("https://tls.peet.ws/api/all")
    data = resp.json()

    # 提取HTTP/2指纹信息
    h2_info = data.get("http2", {})
    settings = h2_info.get("sent_settings", {})

    # Chrome 131 期望值
    expected = {
        "1": 65536,    # HEADER_TABLE_SIZE
        "2": 0,        # ENABLE_PUSH
        "3": 1000,     # MAX_CONCURRENT_STREAMS
        "4": 6291456,  # INITIAL_WINDOW_SIZE
        "5": 16384,    # MAX_FRAME_SIZE
        "6": 262144,   # MAX_HEADER_LIST_SIZE
    }

    print("=== HTTP/2 SETTINGS 指纹验证 ===")
    all_match = True
    for key, expected_val in expected.items():
        actual_val = settings.get(key)
        match = "✅" if actual_val == expected_val else "❌"
        if actual_val != expected_val:
            all_match = False
        print(f"  参数 {key}: 期望={expected_val}, 实际={actual_val} {match}")

    # 验证伪头部顺序
    pseudo_order = h2_info.get("sent_pseudo_headers", [])
    chrome_order = [":method", ":authority", ":scheme", ":path"]
    order_match = pseudo_order == chrome_order

    print(f"\n=== 伪头部顺序 ===")
    print(f"  期望: {chrome_order}")
    print(f"  实际: {pseudo_order}")
    print(f"  匹配: {'✅' if order_match else '❌'}")

    # 验证WINDOW_UPDATE
    window_update = h2_info.get("sent_window_update", 0)
    expected_wu = 15663105
    print(f"\n=== WINDOW_UPDATE ===")
    print(f"  期望: {expected_wu}")
    print(f"  实际: {window_update}")
    print(f"  匹配: {'✅' if window_update == expected_wu else '❌'}")

    return all_match and order_match

if __name__ == "__main__":
    success = verify_fingerprint()
    print(f"\n{'🎉 指纹完全匹配Chrome 131!' if success else '⚠️ 指纹存在差异'}")
```

### 验证结果对比

运行上述脚本后，对比真实Chrome 131访问同一接口的结果：

```
=== HTTP/2 SETTINGS 指纹验证 ===
  参数 1: 期望=65536, 实际=65536 ✅
  参数 2: 期望=0, 实际=0 ✅
  参数 3: 期望=1000, 实际=1000 ✅
  参数 4: 期望=6291456, 实际=6291456 ✅
  参数 5: 期望=16384, 实际=16384 ✅
  参数 6: 期望=262144, 实际=262144 ✅

=== 伪头部顺序 ===
  期望: [':method', ':authority', ':scheme', ':path']
  实际: [':method', ':authority', ':scheme', ':path']
  匹配: ✅

=== WINDOW_UPDATE ===
  期望: 15663105
  实际: 15663105
  匹配: ✅

🎉 指纹完全匹配Chrome 131!
```

---

## 参考资料

- [HTTP/2 RFC 9113](https://www.rfc-editor.org/rfc/rfc9113)
- [QUIC RFC 9000](https://www.rfc-editor.org/rfc/rfc9000)
- [Extensible Priorities RFC 9218](https://www.rfc-editor.org/rfc/rfc9218)
- [curl-impersonate 项目](https://github.com/lwthiker/curl-impersonate)
- [curl_cffi 文档](https://curl-cffi.readthedocs.io/)
- [tls.peet.ws 指纹检测](https://tls.peet.ws/)
