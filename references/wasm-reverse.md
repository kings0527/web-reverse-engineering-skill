# WASM 深度逆向工程指南

> 随着支付宝、京东、字节等头部网站开始用WASM保护核心算法，
> WASM逆向成为Web逆向工程的新前沿。本文档提供完整的WASM逆向工作流。

## 目录

- [一、WASM在反爬中的应用场景](#一wasm在反爬中的应用场景)
- [二、WASM基础知识](#二wasm基础知识)
- [三、WASM逆向工具链](#三wasm逆向工具链)
- [四、WASM逆向方法论](#四wasm逆向方法论)
- [五、WASM算法还原](#五wasm算法还原)
- [六、实战案例](#六实战案例)
- [七、工具能力对比](#七工具能力对比)

---

## 一、WASM在反爬中的应用场景

### 哪些网站使用WASM保护

| 网站/平台 | WASM用途 | 保护目标 |
|-----------|---------|---------|
| 支付宝（Alipay） | 设备指纹、风控签名 | `apdid`、`awsc` 参数 |
| 京东（JD） | 签名算法 | `sign`、`h5st` 参数 |
| 字节跳动（抖音/TikTok） | 部分加密逻辑 | `X-Bogus` 辅助计算 |
| Cloudflare Turnstile | 验证码核心逻辑 | Challenge 求解 |
| hCaptcha | 工作量证明 | PoW 计算 |

### WASM保护的优势

| 对比维度 | JavaScript 混淆 | WASM 保护 |
|---------|----------------|-----------|
| 格式 | 文本（可搜索） | 二进制（不可直接阅读） |
| 调试难度 | 中等（DevTools支持好） | 高（调试支持有限） |
| 性能 | 受限于解释器 | 接近原生性能 |
| 逆向工具 | AST分析、beautify | 反汇编、反编译 |
| 保护强度 | 中（配合控制流平坦化） | 高（天然二进制+可配合OLLVM） |

---

## 二、WASM基础知识

### 2.1 WASM二进制格式

WASM模块由多个Section组成，每个Section有固定ID：

```
WASM Binary Format:
┌──────────────────────────────────────┐
│ Magic Number: 0x00 0x61 0x73 0x6D    │  ("\0asm")
│ Version: 0x01 0x00 0x00 0x00         │  (version 1)
├──────────────────────────────────────┤
│ Section 1:  Type     (函数签名定义)     │
│ Section 2:  Import   (导入声明)        │
│ Section 3:  Function (函数索引)        │
│ Section 4:  Table    (间接调用表)      │
│ Section 5:  Memory   (线性内存声明)     │
│ Section 6:  Global   (全局变量)        │
│ Section 7:  Export   (导出声明)        │
│ Section 10: Code     (函数体字节码)     │
│ Section 11: Data     (数据段初始化)     │
└──────────────────────────────────────┘
```

关键概念：

- **线性内存**：WASM使用一块连续的字节数组作为内存，JS和WASM共享同一个`ArrayBuffer`
- **栈机器**：WASM是基于栈的虚拟机，指令从栈上取值并压入结果
- **模块化**：每个.wasm文件是一个独立模块，通过import/export与外部交互

### 2.2 WASM与JS的交互

```javascript
// 典型的WASM加载和调用模式
const wasmBuffer = await fetch('/crypto.wasm').then(r => r.arrayBuffer());

// Import对象：JS提供给WASM调用的函数和内存
const importObject = {
  env: {
    memory: new WebAssembly.Memory({ initial: 256, maximum: 512 }),
    table: new WebAssembly.Table({ initial: 0, element: 'anyfunc' }),
    // JS回调函数，WASM可以调用
    __console_log: (ptr, len) => {
      const bytes = new Uint8Array(memory.buffer, ptr, len);
      console.log(new TextDecoder().decode(bytes));
    },
    // 提供随机数
    __get_random: () => Math.random() * 0xFFFFFFFF | 0,
  },
};

// 实例化WASM模块
const { instance } = await WebAssembly.instantiate(wasmBuffer, importObject);

// 调用导出函数
const result = instance.exports.compute_signature(inputPtr, inputLen);

// 从线性内存中读取结果
const outputBytes = new Uint8Array(instance.exports.memory.buffer, result, 32);
```

---

## 三、WASM逆向工具链

### 3.1 wabt (WebAssembly Binary Toolkit)

安装：

```bash
# macOS
brew install wabt

# Ubuntu/Debian
apt-get install wabt

# Windows (预编译二进制)
# 从 https://github.com/WebAssembly/wabt/releases 下载

# 或通过npm
npm install -g wabt
```

核心工具使用：

```bash
# 1. wasm2wat: 二进制 → 文本格式（WAT）
wasm2wat crypto.wasm -o crypto.wat

# 2. wasm-objdump: 查看section信息
wasm-objdump -h crypto.wasm          # 查看section头
wasm-objdump -x crypto.wasm          # 查看所有详情
wasm-objdump -d crypto.wasm          # 反汇编代码段

# 3. wasm-decompile: 反编译为类C伪代码（实验性但很有用）
wasm-decompile crypto.wasm -o crypto.dcmp

# 4. wasm-validate: 验证wasm文件格式正确性
wasm-validate crypto.wasm

# 完整工作流示例
wasm-objdump -x crypto.wasm | grep -i "export"   # 找到导出函数
wasm2wat crypto.wasm -o crypto.wat                # 转为文本格式
wasm-decompile crypto.wasm -o crypto.pseudo.c     # 反编译
```

WAT格式示例：

```wat
;; wasm2wat 输出示例
(module
  (type (;0;) (func (param i32 i32) (result i32)))
  (import "env" "memory" (memory (;0;) 256 512))
  (func $compute_hash (type 0) (param $input_ptr i32) (param $input_len i32) (result i32)
    (local $i i32)
    (local $temp i32)
    ;; 循环处理输入
    (block $break
      (loop $loop
        (br_if $break (i32.ge_u (local.get $i) (local.get $input_len)))
        ;; 从内存加载一个字节
        (i32.load8_u (i32.add (local.get $input_ptr) (local.get $i)))
        ;; ... 哈希计算逻辑
        (local.set $i (i32.add (local.get $i) (i32.const 1)))
        (br $loop)
      )
    )
    (local.get $temp)
  )
  (export "compute_hash" (func $compute_hash))
)
```

### 3.2 Ghidra WASM分析

#### 安装与配置

```bash
# 1. 下载安装 Ghidra (需要 JDK 17+)
# https://ghidra-sre.org/

# 2. 安装 WASM 插件
# 方法A: ghidra-wasm-plugin
git clone https://github.com/nicholasrice/ghidra-wasm-plugin
cd ghidra-wasm-plugin
gradle buildExtension
# 将生成的zip文件安装到 Ghidra: File → Install Extensions

# 方法B: 使用 Ghidra 10.4+ 内置WASM支持（推荐）
# Ghidra 10.4 起已内置WASM Loader
```

#### 分析流程

1. **导入文件**：File → Import File → 选择 .wasm 文件
2. **自动分析**：Analysis → Auto Analyze（选中所有分析器）
3. **查看导出函数**：Symbol Tree → Exports
4. **反编译**：双击函数 → 右侧显示反编译的C代码
5. **交叉引用**：右键函数 → References → Find references to

#### Ghidra分析技巧

```
# 函数重命名：识别出算法后重命名函数
# 右键函数名 → Rename Function → 输入有意义的名字

# 类型标注：为参数添加类型信息
# 右键参数 → Retype Variable → 选择或创建结构体

# 常量搜索：Search → For Scalars → 输入已知算法常量
# 例如搜索 0x6a09e667 (SHA-256 初始值)
```

### 3.3 Chrome DevTools WASM调试

```
# 步骤1: 打开DevTools → Sources面板
# WASM文件显示在 wasm:// 协议下

# 步骤2: 查看反汇编
# 点击.wasm文件，Chrome会显示WAT格式的文本

# 步骤3: 设置断点
# 点击行号即可设置断点（仅支持有DWARF信息的WASM）
# 对于没有调试信息的WASM，需要通过Scope面板观察栈变化

# 步骤4: Memory Inspector
# 在断点处，右键Memory → Inspect Memory
# 可以查看线性内存的十六进制内容

# 步骤5: 条件断点
# 右键行号 → Add conditional breakpoint
# 可以基于栈值或locals值设置条件
```

### 3.4 其他工具

```bash
# radare2/rizin: 轻量级逆向框架
rizin -A crypto.wasm
> afl          # 列出所有函数
> pdf @func.1  # 反汇编函数1
> VV @func.1   # 图形化视图

# Binary Ninja: 商业级逆向工具
# 通过插件支持WASM: https://github.com/aspect-build/binja-wasm

# binaryen (wasm-opt): 优化/反优化WASM
npm install -g binaryen

# 去除优化（使代码更易读）
wasm-opt crypto.wasm -O0 --flatten --rereloop -o crypto_readable.wasm

# 打印函数信息
wasm-opt crypto.wasm --print-functions
```

---

## 四、WASM逆向方法论

### 4.1 目标定位：找到关键函数

#### 方法1：从JS调用点反推

```javascript
// 在DevTools Console中Hook WebAssembly.instantiate
const origInstantiate = WebAssembly.instantiate;
WebAssembly.instantiate = async function(bufferSource, importObject) {
  console.log('[WASM] instantiate called');
  console.log('[WASM] imports:', Object.keys(importObject?.env || {}));

  const result = await origInstantiate.apply(this, arguments);
  const exports = result.instance.exports;

  console.log('[WASM] exports:', Object.keys(exports));

  // Hook每个导出函数
  for (const [name, fn] of Object.entries(exports)) {
    if (typeof fn === 'function') {
      const original = fn;
      exports[name] = function(...args) {
        console.log(`[WASM] ${name} called with:`, args);
        const ret = original.apply(this, args);
        console.log(`[WASM] ${name} returned:`, ret);
        return ret;
      };
    }
  }
  return result;
};
```

#### 方法2：从网络请求参数反推

```javascript
// Hook fetch/XHR，找到加密参数的来源
const origFetch = window.fetch;
window.fetch = function(url, options) {
  if (url.includes('/api/')) {
    console.log('[Fetch]', url);
    console.log('[Fetch] body:', options?.body);
    // 在这里打断点，向上追踪调用栈找到WASM调用
    debugger;
  }
  return origFetch.apply(this, arguments);
};
```

#### 方法3：Hook WASM的import函数

```javascript
// 拦截WASM通过import调用的JS函数
const originalImports = { ...importObject.env };
for (const [name, fn] of Object.entries(importObject.env)) {
  if (typeof fn === 'function') {
    importObject.env[name] = function(...args) {
      console.log(`[WASM→JS] ${name}(${args.join(', ')})`);
      return fn.apply(this, args);
    };
  }
}
```

### 4.2 静态分析流程

```bash
# Step 1: 获取WAT文本
wasm2wat target.wasm -o target.wat

# Step 2: 识别导出函数
grep "(export" target.wat
# 输出示例:
# (export "compute_sign" (func $func42))
# (export "init_key" (func $func15))
# (export "memory" (memory 0))

# Step 3: 分析关键函数体
# 在WAT中搜索 $func42，观察指令模式

# Step 4: 搜索密码学常量
grep -n "0x6a09e667\|0x5be0cd19\|0x428a2f98" target.wat
# 如果匹配 → SHA-256

grep -n "0x61707865\|0x3320646e" target.wat
# 如果匹配 → ChaCha20 ("expand 32-byte k")

# Step 5: 使用wasm-decompile获取高级视图
wasm-decompile target.wasm -o target.pseudo.c
```

### 4.3 动态分析流程

```javascript
// Step 1: 在WASM导出函数处设置断点（通过wrapper）
const wasmExports = instance.exports;
const originalSign = wasmExports.compute_sign;

wasmExports.compute_sign = function(ptr, len, outPtr) {
  // 读取输入
  const memory = new Uint8Array(wasmExports.memory.buffer);
  const input = memory.slice(ptr, ptr + len);
  console.log('[Input]', new TextDecoder().decode(input));

  // 调用原函数
  const result = originalSign(ptr, len, outPtr);

  // 读取输出
  const output = memory.slice(outPtr, outPtr + 32); // 假设32字节输出
  console.log('[Output]', Buffer.from(output).toString('hex'));

  return result;
};

// Step 2: 内存快照对比
function dumpMemory(label) {
  const mem = new Uint8Array(wasmExports.memory.buffer);
  return { label, snapshot: mem.slice(0, 4096) }; // 前4KB
}

// 调用前后对比内存变化
const before = dumpMemory('before');
wasmExports.compute_sign(inputPtr, inputLen, outputPtr);
const after = dumpMemory('after');

// 找到变化的内存区域
for (let i = 0; i < before.snapshot.length; i++) {
  if (before.snapshot[i] !== after.snapshot[i]) {
    console.log(`Memory changed at offset ${i}: ${before.snapshot[i]} → ${after.snapshot[i]}`);
  }
}
```

### 4.4 常见加密算法识别

#### 通过常量识别

| 算法 | 特征常量 | 十六进制值 |
|------|---------|-----------|
| AES | S-Box首字节 | `0x63, 0x7c, 0x77, 0x7b` |
| SHA-256 | 初始哈希值H0 | `0x6a09e667` |
| SHA-256 | 轮常量K[0] | `0x428a2f98` |
| ChaCha20 | Sigma常量 | `0x61707865, 0x3320646e, 0x79622d32, 0x6b206574` |
| Blake3 | IV[0] | `0x6A09E667` (同SHA-256) |
| SM3 | IV[0] | `0x7380166F` |
| SM4 | FK[0] | `0xA3B1BAC6` |
| MD5 | T[1] | `0xD76AA478` |
| CRC32 | 多项式 | `0xEDB88320` |

#### 通过结构识别

| 结构模式 | 特征指令 | 可能的算法 |
|---------|---------|-----------|
| ARX (Add-Rotate-XOR) | `i32.add`, `i32.rotl`, `i32.xor` 循环 | ChaCha20, Blake2/3, xxHash |
| SPN (代替-置换网络) | 查表 + 位移 + 异或 | AES, SM4 |
| Feistel结构 | 左右交换 + 轮函数 | DES, Blowfish, SM4 |
| Merkle-Damgård | 分块处理 + 压缩函数 | MD5, SHA-1, SHA-256, SM3 |
| Sponge结构 | 吸收+挤出阶段 | SHA-3/Keccak |

---

## 五、WASM算法还原

### 5.1 从WASM到可读代码

转换思路：WAT指令 → 操作语义 → 高级语言实现

```wat
;; WAT 示例片段（一个简单的XOR循环）
(func $xor_encode (param $data_ptr i32) (param $len i32) (param $key i32)
  (local $i i32)
  (local.set $i (i32.const 0))
  (block $break
    (loop $loop
      (br_if $break (i32.ge_u (local.get $i) (local.get $len)))
      (i32.store8
        (i32.add (local.get $data_ptr) (local.get $i))
        (i32.xor
          (i32.load8_u (i32.add (local.get $data_ptr) (local.get $i)))
          (local.get $key)
        )
      )
      (local.set $i (i32.add (local.get $i) (i32.const 1)))
      (br $loop)
    )
  )
)
```

等价的JS实现：

```javascript
// 对应上面WAT的JS还原
function xorEncode(data, key) {
  const result = new Uint8Array(data.length);
  for (let i = 0; i < data.length; i++) {
    result[i] = data[i] ^ (key & 0xFF);
  }
  return result;
}
```

### 5.2 内存布局分析

```javascript
// WASM线性内存的load/store对应关系
// i32.load  offset=0 align=2  → 读取4字节，小端序
// i32.load8_u                  → 读取1字节，无符号扩展
// i32.load16_u                 → 读取2字节，无符号扩展
// i32.store offset=0 align=2  → 写入4字节，小端序

// 确定参数在内存中的偏移
function analyzeMemoryLayout(wasmInstance) {
  const memory = new DataView(wasmInstance.exports.memory.buffer);

  // 常见的内存布局模式:
  // [0x0000 - 0x0FFF]: 栈空间
  // [0x1000 - 0x1FFF]: 全局数据/常量
  // [0x2000 - ...]:    堆空间（动态分配）

  // 通过导出的 __heap_base 或 __data_end 确定分区
  const heapBase = wasmInstance.exports.__heap_base?.value || 0x2000;
  const dataEnd = wasmInstance.exports.__data_end?.value || 0x1000;

  console.log(`Stack: 0x0000 - 0x${(dataEnd - 1).toString(16)}`);
  console.log(`Data:  0x${dataEnd.toString(16)} - 0x${(heapBase - 1).toString(16)}`);
  console.log(`Heap:  0x${heapBase.toString(16)} - ...`);
}

// 结构体布局推断示例
// 假设WASM中有这样的内存访问模式:
// i32.load offset=0   → field_0 (4 bytes)
// i32.load offset=4   → field_1 (4 bytes)
// i32.load offset=8   → field_2 (4 bytes)
// i32.load offset=12  → field_3 (4 bytes)
// 推断为: struct { i32 field_0; i32 field_1; i32 field_2; i32 field_3; }
```

### 5.3 纯JS/Python复现

```python
"""
WASM算法JS等价实现的Python验证模板
"""
import struct

class WasmAlgorithmReimpl:
    """WASM算法的纯Python复现"""

    def __init__(self):
        # 模拟线性内存
        self.memory = bytearray(65536)  # 64KB初始内存
        # 模拟全局变量
        self.globals = {}

    def i32_load(self, addr):
        """模拟 i32.load（小端序）"""
        return struct.unpack_from('<I', self.memory, addr)[0]

    def i32_store(self, addr, value):
        """模拟 i32.store（小端序）"""
        struct.pack_into('<I', self.memory, addr, value & 0xFFFFFFFF)

    def i32_rotl(self, value, shift):
        """模拟 i32.rotl（循环左移）"""
        shift &= 31
        return ((value << shift) | (value >> (32 - shift))) & 0xFFFFFFFF

    def compute_hash(self, input_bytes):
        """
        还原的哈希函数
        对应WASM导出函数: compute_hash(ptr, len) -> ptr
        """
        # 将输入写入内存
        input_ptr = 0x1000
        self.memory[input_ptr:input_ptr + len(input_bytes)] = input_bytes

        # ... 在此实现从WAT逆向得到的算法逻辑 ...

        # 读取输出
        output_ptr = 0x2000
        return bytes(self.memory[output_ptr:output_ptr + 32])

    def verify(self, input_data, expected_output):
        """验证复现结果是否匹配WASM输出"""
        result = self.compute_hash(input_data)
        match = result == expected_output
        print(f"Input:    {input_data.hex()}")
        print(f"Expected: {expected_output.hex()}")
        print(f"Got:      {result.hex()}")
        print(f"Match:    {'✅' if match else '❌'}")
        return match
```

---

## 六、实战案例

### 6.1 从Export入口到算法还原

以一个简化的WASM加密函数为例，展示完整逆向过程：

#### Step 1: 获取并分析WASM模块

```bash
# 下载目标网站的.wasm文件
curl -o target.wasm "https://example.com/static/crypto.wasm"

# 查看基本信息
wasm-objdump -x target.wasm

# 输出:
# Section Details:
# Export[3]:
#  - func[12] <compute_sign> -> "compute_sign"
#  - func[5]  <init_state>   -> "init_state"
#  - memory[0]               -> "memory"
```

#### Step 2: 反编译关键函数

```bash
# 使用 wasm-decompile 获取伪代码
wasm-decompile target.wasm -o target.c

# target.c 中 compute_sign 的输出（简化）:
# export function compute_sign(a:int, b:int):int {
#   var c:int = g_a - 64;   // 分配栈空间
#   c[15]:int = a;          // 保存参数
#   c[14]:int = b;          // 保存参数
#   // 初始化state
#   c[0]:int = 1732584193;  // 0x67452301 ← MD5 初始值!
#   c[1]:int = -271733879;  // 0xEFCDAB89
#   c[2]:int = -1732584194; // 0x98BADCFE
#   c[3]:int = 271733878;   // 0x10325476
#   // ... 分块处理循环 ...
# }
```

#### Step 3: 识别算法

```python
# 通过特征常量确认是MD5
MD5_INIT = [0x67452301, 0xEFCDAB89, 0x98BADCFE, 0x10325476]
# 匹配! 这是标准MD5的初始化向量

# 进一步确认：检查轮常量
MD5_T = [0xD76AA478, 0xE8C7B756, 0x242070DB, ...]  # 64个轮常量
```

#### Step 4: 纯JS复现

```javascript
// 确认为MD5后，直接使用标准MD5实现
const crypto = require('crypto');

function computeSign(input) {
  // WASM中的compute_sign实质上是 MD5(input)
  return crypto.createHash('md5').update(input).digest('hex');
}

// 验证
const testInput = Buffer.from('test_input_data');
const wasmResult = '...'; // 从WASM调用获取的结果
const jsResult = computeSign(testInput);
console.log('Match:', wasmResult === jsResult); // true ✅
```

#### Step 5: 处理非标准变体

```javascript
// 实际场景中，WASM往往不是标准算法，而是变体
// 常见变体：
// 1. 修改初始化向量
// 2. 额外的前/后处理（加盐、密钥混合）
// 3. 修改轮函数中的常量
// 4. 多算法组合（如 HMAC-SHA256 + 自定义XOR）

function computeSignVariant(input, timestamp) {
  // 示例：MD5变体，先混合时间戳
  const salt = Buffer.from(timestamp.toString());
  const mixed = Buffer.concat([salt, input, salt]);
  const hash = crypto.createHash('md5').update(mixed).digest();
  // 后处理：取前16字节进行XOR置换
  for (let i = 0; i < 16; i++) {
    hash[i] ^= salt[i % salt.length];
  }
  return hash.toString('hex');
}
```

---

## 七、工具能力对比

### 开源WASM分析工具能力矩阵

| 能力 | wabt | Ghidra | Chrome DevTools | radare2 |
|------|------|--------|-----------------|----------|
| WASM静态反汇编 | ✅ | ✅ | ✅ | ✅ |
| 自动算法识别 | ❌ | ⚠️ 手动搜索 | ❌ | ⚠️ 部分支持 |
| 动态trace | ❌ | ❌ | ⚠️ 粗粒度 | ❌ |
| 内存读写追踪 | ❌ | ❌ | ⚠️ 需手动注入 | ❌ |
| 控制流图生成 | ❌ | ✅ | ❌ | ✅ |
| Section解析 | ✅ | ✅ | ❌ | ✅ |
| 导出函数识别 | ✅ | ✅ | ✅ | ✅ |

### 开源方案覆盖度评估

```
开源工具组合覆盖度：~85%

✅ 完全覆盖（100%）:
  - 静态反汇编/反编译
  - Section解析
  - 导出函数识别
  - 基本调试
  - 控制流图生成

⚠️ 基本覆盖（70-80%）:
  - 算法识别（手动常量搜索）
  - 内存布局分析
  - 交叉引用

❌ 待提升（<50%）:
  - 指令级动态追踪
  - 全量内存trace
  - 自动化程度
  - 混淆WASM的处理（如wasm-obfuscator处理后的代码）
```

### 推荐工作流

```
实际逆向推荐流程：

1. 快速定位：Hook JS层的WASM调用 ──→ 确定目标函数
2. 静态分析：wabt + Ghidra ──→ 理解算法结构
3. 常量搜索：搜索已知算法常量 ──→ 快速识别标准算法
4. 动态验证：Chrome DevTools + Memory Hook ──→ 确认输入输出
5. 算法还原：手动转写为JS/Python ──→ 完成复现
6. 正确性验证：对比WASM输出和复现结果 ──→ 确保一致
```

---

## 参考资料

- [WebAssembly 规范](https://webassembly.github.io/spec/)
- [wabt GitHub](https://github.com/WebAssembly/wabt)
- [Ghidra](https://ghidra-sre.org/)
- [binaryen (wasm-opt)](https://github.com/WebAssembly/binaryen)
- [Chrome DevTools WASM调试](https://developer.chrome.com/docs/devtools/wasm/)
- [radare2 WASM](https://r2wiki.readthedocs.io/en/latest/options/wasm/)
