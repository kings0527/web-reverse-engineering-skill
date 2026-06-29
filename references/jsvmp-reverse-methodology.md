# TikTok VM逆向方法论：从字节码到签名还原

> 本文从tiktok-reverse-engineering和tiktok-xgnarly-decoded两个项目中提取完整的TikTok VM逆向方法论，包括逆向五步法、操作码分类、X-Gnarly签名算法完整还原过程。

## 1. 逆向五步法详解

基于tiktok-reverse-engineering项目的实战经验，TikTok VM逆向遵循以下五步法：

### Step 1：字符串反混淆

TikTok的JS代码使用多层字符串数组隐藏真实逻辑。逆向的第一步是用Babel AST批量替换数组引用：

```javascript
// 来源：tiktok-reverse-engineering/deobf.js
// 原始代码中的混淆引用：Kg[123] → "navigator", aa[45] → 2654435769
const deobfuscateEncodedStringVisitor = {
  MemberExpression(path) {
    if (path.node?.object?.name == "Kg") {
      const value = giggernigger[path.node.property.value];
      path.replaceWith(t.valueToNode(value));
    }
    if (path.node?.object?.name == "aa") {
      const value = aa[path.node.property.value];
      path.replaceWith(t.valueToNode(value));
    }
  },
};
```

**关键洞察**：TikTok使用多个字符串数组（`Kg`和`aa`），一个是字符串常量池，一个是数值常量池。必须先识别所有数组并一次性替换，否则后续的静态分析无法进行。

### Step 2：字节码提取

TikTok VM的字节码存储在`Uint8Array`中，通过`fetchInstructions`函数解析二进制格式：

```javascript
// 来源：tiktok-reverse-engineering/bytearray.js（简化）
// 字节码格式：[字符串数量][字符串...][函数数量][函数信息...]
function fetchInstructions() {
    var R = [];
    var stringCount = readVarint(t);  // 变长整数编码
    var strings = [];
    for (var n = 0; n < stringCount; ++n) {
        strings.push(decodeString(t));  // UTF-8变长字符串
    }
    var funcCount = readVarint(t);
    for (n = 0; n < funcCount; ++n) {
        var funcIndex = readVarint(t);
        var strictMode = Boolean(readVarint(t));
        var exceptionHandlers = [];    // try/catch表
        var handlerCount = readVarint(t);
        for (var c = 0; c < handlerCount; ++c) {
            exceptionHandlers.push([readVarint(t), readVarint(t), readVarint(t), readVarint(t)]);
        }
        var bytecode = [];
        var instrCount = readVarint(t);
        for (var l = 0; l < instrCount; ++l) {
            bytecode.push(readVarint(t));
        }
        R.push([bytecode, funcIndex, strictMode, exceptionHandlers]);
    }
    return {strings, instructions: R};
}
```

**实战技巧**：与其花时间逆向压缩算法，不如直接在运行时提取已解压的字节码。在浏览器中断点到VM入口，直接dump `Uint8Array`的内容即可。

### Step 3：操作码识别与分类

将嵌套if-else分发改写为清晰的switch-case，逐步标注每条操作码的语义：

```javascript
// 来源：tiktok-reverse-engineering/disasm.js（核心结构）
// 原始：深层嵌套if-else（e < 38 → e < 19 → e < 9 → ...）
// 改写：switch-case + 注释
function OpHandler(instructionSet, strings) {
    let index = 0, pointer = -1;
    while (true) {
        var opcode = instruction[index++];
        switch (opcode) {
            case 0:  // SUB
                pointer--;
                devirtOutput += `SUB stack[${pointer}] -= stack[${pointer+1}]\n`;
                break;
            case 2:  // CALL
                var argCount = instruction[index++];
                pointer -= (argCount + 1);
                devirtOutput += `CALL func.apply(thisArg, [${argCount} args])\n`;
                break;
            case 41: // ADD
                pointer--;
                devirtOutput += `ADD stack[${pointer}] += stack[${pointer+1}]\n`;
                break;
            case 58: // RETURN
                devirtOutput += `RETURN stack[${pointer}]\n`;
                pointer--;
                break;
            // ... 77条操作码
        }
    }
}
```

### Step 4：执行追踪定位目标函数

在VM主循环中注入全局数组收集运行时数据，定位签名生成函数：

```javascript
// 来源：tiktok-reverse-engineering/README.md
// 在VM execute循环中注入追踪代码
while (true) {
    var opcode = o[c++];
    if (window.thatarray) {
        window.thatarray.push(
            `[VM] Opcode ${opcode} at PC ${c-1}, stack: ${l}, instrLen: ${o.length}`
        );
    }
    if (window.oparrays) {
        for (let i = 0; i < R.length; i++) {
            if (R[i][0] === o) window.oparrays.push(`${i}`);
        }
    }
    // ... 正常执行opcode
}

// 使用方式：
// 1. window.thatarray = []; window.oparrays = [];
// 2. 触发目标请求
// 3. [...new Set(window.oparrays)] → 得到参与签名的VM函数索引
```

**追踪结果**：
```
vm249 → X-Gnarly签名生成
vm103 → X-Bogus签名生成
vm42  → strData加密数据
```

### Step 5：反汇编输出与算法还原

将字节码反汇编为可读汇编，标注每个函数的入口/出口：

```assembly
// 来源：tiktok-reverse-engineering 反汇编输出示例
------------------------103--------------------------
// 0 PUSH_STRING → stack[0] = "d41d8cd98f00b204e9800998ecf8427e"
// 3 SET_VAR scope[0][4] ← stack[0]
// 6 GET_VAR → stack[0] = scope[0][3]
// 9 PUSH_UNDEFINED → stack[1] = undefined
// 10 STRICT_NOT_EQUAL stack[0] = stack[0] !== stack[1]
```

## 2. TikTok VM操作码分类表

基于77条操作码的功能分类：

| 类别 | 操作码编号 | 指令 | 说明 |
|------|-----------|------|------|
| 算术 | 0,22,41,60,67 | SUB,MUL,ADD,MOD,DIV | 基本四则+取模 |
| 位移 | 8,11,72 | USHR,SHL,SHR | 含无符号右移 |
| 位运算 | 12,34,63 | XOR,BITOR,BITAND | 位级操作 |
| 比较 | 5,24,27,33,50,54,64,74 | EQ,LT,GTE,EQ2,NEQ,STRICT_NE,GT,LTE | 多种相等判断 |
| 控制流 | 1,3,20,26,29,31 | JIF,JUMP,JIF_OR_POP,JIT_OR_POP,SWITCH,JIT | 6种跳转 |
| 变量 | 30,47,55,66 | GET_GLOBAL,SET_VAR,SCOPE_REF,GET_VAR | 作用域链访问 |
| 对象 | 16,17,35,49,65,71 | SET_PROP,DEL_PROP,GET_PROP,NEW_OBJ,GET_ELEM,SET_PROP_NP | 属性操作全集 |
| 函数 | 2,7,58,75 | CALL,PUSH_FUNC,RETURN,BIND_APPLY | 含构造函数 |
| 栈操作 | 6,9,39,40,52,53,57,61 | PUSH,POP,UNDEF,NULL,DUP,SET_UNDEF,MAGIC,TRUE | 常量推入 |
| 类型 | 4,13,14,38,62 | TO_NUM,NEG,TYPEOF,NOT,TYPEOF_GLOBAL | 类型转换+检测 |
| 循环 | 10,15 | FOR_IN_SETUP,FOR_IN_NEXT | for-in迭代器 |
| 异常 | 19,48 | TRY,THROW | 异常处理 |
| 特殊 | 21,23,28,42,43,44,45 | SCOPE,FALSE,POST_DEC,PRE_DEC,IN,INSTANCEOF,PRE_INC | 杂项 |

## 3. X-Gnarly签名算法完整还原

X-Gnarly是TikTok API请求的核心签名参数，由tiktok-xgnarly-decoded项目完整还原。

### 3.1 整体流程

```
输入(queryString, body, userAgent, counters)
  → 构建16字段TLV载荷（payload.js）
  → 48字节随机密钥生成
  → ChaCha-XOR流密码加密（cipher.js）
  → 密钥嵌入密文（位置由校验和决定）
  → 添加魔数头'K'(ASCII 75)
  → 自定义Base64编码
  → X-Gnarly签名值（约332字符）
```

### 3.2 ChaCha-XOR变体实现

TikTok使用的ChaCha变体与标准RFC 8439有4处关键差异：

```javascript
// 来源：tiktok-xgnarly-decoded/src/cipher.js

// 差异1：自定义sigma常量（非标准"expand 32-byte k"）
// 标准ChaCha: [0x61707865, 0x3320646e, 0x79622d32, 0x6b206574]
// TikTok变体: 从常量表索引[9,69,51,92]提取
export const SIGMA = [1196819126, 600974999, 3863347763, 1451689750];

// 差异2：12个密钥字全部随机（非8密钥+4 nonce）
// 每次调用都是全新的对称密钥，密钥嵌入到密文中供解码器恢复

// 差异3：可变轮数[5,20]，由密钥低位nibble之和决定
export function deriveRounds(keyWords) {
    let r = 0;
    for (const w of keyWords) r = (r + (w & 15)) & 15;
    return r + 5;  // 范围 [5, 20]
}

// 差异4：标准quarter-round和column/diagonal交替结构不变
function quarter(s, a, b, c, d) {
    s[a] = u32(s[a] + s[b]); s[d] = rotl(s[d] ^ s[a], 16);
    s[c] = u32(s[c] + s[d]); s[b] = rotl(s[b] ^ s[c], 12);
    s[a] = u32(s[a] + s[b]); s[d] = rotl(s[d] ^ s[a], 8);
    s[c] = u32(s[c] + s[d]); s[b] = rotl(s[b] ^ s[c], 7);
}

// 加密函数：XOR流密码（加密和解密是同一操作）
export function chachaXor(bytes, keyWords, rounds) {
    const state = [...SIGMA, ...keyWords]; // 16-word state
    for (let off = 0; off < bytes.length; off += 64) {
        const stream = chachaBlock(state, rounds);
        state[12] = u32(state[12] + 1); // 块计数器递增
        for (let i = 0; i < Math.min(64, bytes.length - off); i++) {
            bytes[off + i] ^= (stream[i >>> 2] >>> (8 * (i & 3))) & 0xff;
        }
    }
    return bytes;
}
```

### 3.3 TLV编码结构

16个字段按Type-Length-Value格式序列化，整数用大端序编码：

```javascript
// 来源：tiktok-xgnarly-decoded/src/payload.js

// 字段写入顺序（field 0最后写入，因为它的值是所有整数字段的XOR）
export const FIELD_ORDER = [1, 2, 6, 7, 8, 9, 10, 11, 4, 5, 3, 12, 13, 14, 15, 0];

// 16个字段含义（从动态追踪webmssdk_2.0.0.485还原）：
// Field 1: 65 (固定常量，field 14的高16位 = field1 << 16)
// Field 2: ubcode (API类型码，如POST请求为4)
// Field 3: md5(queryString) — 查询字符串MD5
// Field 4: md5(body) — 请求体MD5
// Field 5: md5(userAgent) — UA的MD5
// Field 6: Math.floor(timestamp / 1000) — Unix时间戳(秒)
// Field 7: 3181061566 (构建时烘焙的常量)
// Field 8: timestamp % 0x80000000 — 时间戳低31位
// Field 9: "5.1.3-ZTCA" — SDK版本标识
// Field 10: "1.0.0.368" — SDK版本号
// Field 11: 1 — 固定值
// Field 12: totalXHRRequests + totalFetchRequests — 请求计数
// Field 13: interceptedXHR + interceptedFetch — 拦截计数
// Field 14: (65 << 16) | random16 — 混合字段
// Field 15: random32 — 完全随机的uint32

// 线格式：[count_u8][key_u8][len_u16_be][value_bytes...]...
// len ≤ 4 → 大端序整数，len > 4 → UTF-8字符串
```

### 3.4 完整编码流程

```javascript
// 来源：tiktok-xgnarly-decoded/src/encode.js
export function encode(queryString, body, userAgent, counters = {}, options = {}) {
    // 1. 构建16字段载荷
    const fields = {
        1: 65, 2: ubcode,
        3: md5(queryString), 4: md5(body), 5: md5(userAgent),
        6: Math.floor(ts / 1000), 7: 3181061566,
        8: ts % 0x80000000, 9: "5.1.3-ZTCA", 10: sdkVersion,
        11: 1, 12: totalRequests, 13: interceptedRequests,
        14: field14, 15: field15,
    };

    // 2. TLV序列化
    const plaintext = encodePayload(fields);

    // 3. 生成48字节随机密钥 → 12个uint32
    const keyBytes = randomBytes(48);
    const keyWords = new Array(12);
    for (let i = 0; i < 12; i++) {
        keyWords[i] = ((keyBytes[i*4] | (keyBytes[i*4+1] << 8) |
            (keyBytes[i*4+2] << 16) | (keyBytes[i*4+3] << 24)) >>> 0);
    }

    // 4. ChaCha-XOR加密
    const rounds = deriveRounds(keyWords);
    const cipher = new Uint8Array(plaintext);
    chachaXor(cipher, keyWords, rounds);

    // 5. 密钥嵌入：位置 = (sum(keyBytes) + sum(cipher)) % (cipherLen + 1)
    const insertPos = computeInsertPos(keyBytes, cipher);

    // 6. 组装：[魔数'K'][cipher前半][密钥48字节][cipher后半]
    const out = new Uint8Array(1 + cipher.length + 48);
    out[0] = 75; // 'K'
    out.set(cipher.subarray(0, insertPos), 1);
    out.set(keyBytes, 1 + insertPos);
    out.set(cipher.subarray(insertPos), 1 + insertPos + 48);

    // 7. 自定义Base64编码
    return encodeBase64(out);
}
```

**逆向验证方法**：解码时反向操作——先Base64解码，找到魔数'K'，然后暴力搜索密钥起始位置（检测有效TLV头），提取密钥后用同样的ChaCha-XOR解密。

## 4. 实战经验总结

1. **Observe-First**：先在浏览器中断点到`xn()`和`bn()`调用入口，确认X-Gnarly和X-Bogus的生成点
2. **字节码硬编码**：对于压缩格式的字节码，直接提取运行时解压后的`Uint8Array`比逆向解压算法更高效
3. **函数索引映射**：通过全局数组收集VM函数索引，用`[...new Set(window.oparrays)]`去重
4. **差分验证**：对比浏览器端签名和复现代码的中间值，逐字段验证TLV载荷正确性
5. **轮数可观察**：ChaCha轮数可以从密钥直接计算（`deriveRounds`），不需要额外编码到密文中
