# 密码学常量快速识别指南

## 概述
在混淆/WASM代码中，通过扫描特征常量可快速识别加密算法类型。

## 常量特征表

### AES
- S-Box首字节序列：`0x63, 0x7c, 0x77, 0x7b, 0xf2, 0x6b, 0x6f, 0xc5`
- Rcon首元素：`0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80`
- 数值特征：256字节的查找表

### SHA-256
- 初始哈希值：`0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a`
- K常量首元素：`0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5`
- 64个32位轮常量

### MD5
- 初始IV：`0x67452301, 0xefcdab89, 0x98badcfe, 0x10325476`
- T表首元素：`0xd76aa478, 0xe8c7b756, 0x242070db`
- 旋转量：`7, 12, 17, 22, 5, 9, 14, 20, 4, 11, 16, 23, 6, 10, 15, 21`

### ChaCha20
- Sigma常量："expand 32-byte k"
- 十六进制：`0x61707865, 0x3320646e, 0x79622d32, 0x6b206574`
- Quarter-round旋转量：16, 12, 8, 7

### RC4
- S-Box初始化：256元素数组，值0-255的排列
- 无固定常量，但 KSA 循环特征明显（256次swap）

### SM3（国密）
- IV：`0x7380166f, 0x4914b2b9, 0x172442d7, 0xda8a0600`
- T值（前16轮）：`0x79cc4519`
- T值（后48轮）：`0x7a879d8a`

### SM4（国密）
- S-Box首字节：`0xd6, 0x90, 0xe9, 0xfe, 0xcc, 0xe1, 0x3d, 0xb7`
- 系统参数FK：`0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc`
- 常量CK：32个32位值

### XXTEA
- Delta常量：`0x9E3779B9`（黄金比例相关）

### Blake3
- IV与SHA-256相同前8个字
- 消息调度排列特征

## 扫描方法

### JavaScript代码扫描
```javascript
// 在代码中搜索32位十六进制常量
const patterns = {
  'SHA-256': [0x6a09e667, 0xbb67ae85, 0x3c6ef372],
  'MD5': [0x67452301, 0xefcdab89, 0x98badcfe],
  'ChaCha20': [0x61707865, 0x3320646e],
  'XXTEA': [0x9E3779B9],
  'SM3': [0x7380166f, 0x4914b2b9],
  'SM4': [0xa3b1bac6, 0x56aa3350],
};

function scanCode(code) {
  const results = [];
  for (const [algo, constants] of Object.entries(patterns)) {
    for (const c of constants) {
      if (code.includes(`0x${c.toString(16)}`) || code.includes(c.toString())) {
        results.push({ algorithm: algo, constant: c, confidence: 0.8 });
      }
    }
  }
  return results;
}
```

### WASM二进制扫描
```javascript
// 扫描WASM Data section中的常量
function scanWasmConstants(wasmBinary) {
  const view = new DataView(wasmBinary.buffer);
  const results = [];
  for (let i = 0; i < wasmBinary.length - 4; i++) {
    const val = view.getUint32(i, true); // little-endian
    if (KNOWN_CONSTANTS.has(val)) {
      results.push({ offset: i, value: val, algorithm: KNOWN_CONSTANTS.get(val) });
    }
  }
  return results;
}
```

## 结构模式识别

除常量外，还可通过代码结构判断算法类型：
- **ARX模式**（Add-Rotate-XOR）：旋转+异或+加法 → ChaCha/Blake/Salsa
- **SPN模式**（Substitution-Permutation）：S-Box查表+行列变换 → AES
- **Feistel结构**：左右半块交换+轮函数 → DES/SM4
- **Merkle-Damgård**：消息分块+压缩函数迭代 → MD5/SHA-1/SHA-256

## 综合判定流程

```
1. 扫描代码中的32位十六进制常量
   ├── 命中已知常量 → 直接判定算法类型
   └── 未命中 → 进入结构分析
2. 分析代码结构模式
   ├── 256字节查找表 → AES/SM4 (SPN)
   ├── 左右半块交换 → DES/Blowfish (Feistel)
   ├── 旋转+异或+加法 → ChaCha/Salsa (ARX)
   └── 分块+压缩迭代 → MD5/SHA (Merkle-Damgård)
3. 验证：用已知输入测试输出是否匹配标准实现
```
