# JavaScript 反混淆技术实战

> 本文涵盖：webcrack反混淆流水线全解析、AST Matcher模式匹配、控制流平坦化还原算法、字符串数组识别与解密、VM沙箱解码技术。所有代码片段均来自真实开源项目。

## 概述

JS反混淆是将经过混淆器（如javascript-obfuscator、obfuscator.io）处理的代码还原为可读代码的过程。现代反混淆不再是简单的正则替换，而是结合了AST模式匹配、VM沙箱执行和数据流分析的复合技术。

---

## 1. webcrack 反混淆流水线

### 1.1 完整流水线入口

来源：`webcrack/packages/webcrack/src/deobfuscate/index.ts`

webcrack采用**有序流水线**设计，按严格的依赖顺序执行12个步骤：

```typescript
// 来源：webcrack - deobfuscate/index.ts
async run(ast, state, sandbox) {
  if (!sandbox) return;  // 需要VM沙箱才能执行解码

  // Phase 1: 识别混淆基础设施
  const stringArray = findStringArray(ast);   // 找字符串数组
  if (!stringArray) return;

  const rotator = findArrayRotator(stringArray);  // 找数组旋转函数
  const decoders = findDecoders(stringArray);     // 找解码器函数

  // Phase 2: 内联对象属性（_0xabc["prop"] → _0xabc.prop）
  state.changes += applyTransform(ast, inlineObjectProps).changes;

  // Phase 3: 内联解码器包装（去掉多层函数壳）
  for (const decoder of decoders) {
    state.changes += applyTransform(ast, inlineDecoderWrappers, decoder).changes;
  }

  // Phase 4: VM沙箱解码（在真实VM中执行解码函数）
  const vm = new VMDecoder(sandbox, stringArray, decoders, rotator);
  state.changes += (await applyTransformAsync(ast, inlineDecodedStrings, { vm })).changes;

  // Phase 5: 清理混淆基础设施（删除字符串数组、旋转函数、解码器）
  if (decoders.length > 0) {
    stringArray.path.remove();
    rotator?.remove();
    decoders.forEach((decoder) => decoder.path.remove());
  }

  // Phase 6: 后处理（合并字符串、移除死代码、还原控制流）
  state.changes += applyTransforms(ast,
    [mergeStrings, deadCode, controlFlowObject, controlFlowSwitch],
    { noScope: true }
  ).changes;
}
```

### 1.2 流水线设计原理

**为什么严格按这个顺序？**

1. **先识别再操作**：必须先找到 `stringArray`、`rotator`、`decoders` 三件套，后续步骤才能执行
2. **先内联再解码**：`inlineObjectProps` 把 `_0xabc["charCodeAt"]` 变成 `_0xabc.charCodeAt`，让后续AST匹配更简单
3. **先去壳再执行**：`inlineDecoderWrappers` 剥离多层包装函数，露出真正的解码调用
4. **先解码再清理**：必须在删除字符串数组之前完成VM解码，否则解码函数无法执行
5. **先清理再反平坦化**：删除死代码后再处理控制流，减少干扰

---

## 2. AST Matcher 模式匹配

### 2.1 @codemod/matchers 库

webcrack使用 `@codemod/matchers` 库进行AST模式匹配，这是比手写 `if-else` 遍历更优雅的方式：

```typescript
// 来源：webcrack - deobfuscate/control-flow-switch.ts
import * as m from '@codemod/matchers';

// 定义捕获变量（类似正则的捕获组）
const sequenceName = m.capture(m.identifier());         // 捕获：序列变量名
const sequenceString = m.capture(
  m.matcher<string>((s) => /^\d+(\|\d+)*$/.test(s))   // 捕获：数字串如 "2|4|3|0|1"
);
const iterator = m.capture(m.identifier());              // 捕获：迭代器变量名

// 定义switch case模式：case "数字":
const cases = m.capture(
  m.arrayOf(
    m.switchCase(m.stringLiteral(m.matcher((s) => /^\d+$/.test(s))))
  )
);

// 完整模式：BlockStatement 包含三个特定语句
const matcher = m.blockStatement(
  m.anyList<t.Statement>(
    // 语句1: const sequence = "2|4|3|0|1".split("|")
    m.variableDeclaration(undefined, [
      m.variableDeclarator(
        sequenceName,
        m.callExpression(
          constMemberExpression(m.stringLiteral(sequenceString), 'split'),
          [m.stringLiteral('|')]
        )
      )
    ]),
    // 语句2: let iterator = 0（或混淆表达式如 -0x1a70 + 0x93d + 0x275 * 0x7）
    m.variableDeclaration(undefined, [m.variableDeclarator(iterator)]),
    // 语句3: while(true) { switch(sequence[iterator++]) { cases... } break; }
    infiniteLoop(
      m.blockStatement([
        m.switchStatement(
          m.memberExpression(
            m.fromCapture(sequenceName),        // 引用捕获的变量
            m.updateExpression('++', m.fromCapture(iterator)),
            true
          ),
          cases
        ),
        m.breakStatement()
      ])
    ),
    m.zeroOrMore()  // 后面可能有其他语句
  )
);
```

### 2.2 匹配器设计思路

**为什么用 `m.capture` + `m.fromCapture`？**

混淆代码中的变量名每次不同（`_0x1a2b`、`_0x3c4d`...），不能用固定名称匹配。`capture` 机制在第一次匹配时捕获实际名称，后续 `fromCapture` 引用确保引用的是同一个变量。这就像正则表达式的反向引用 `\1`。

**为什么用 `m.anyList`？**

`anyList` 允许在 BlockStatement 中有额外的语句（混淆器可能在目标代码前后插入垃圾代码），只要目标模式存在即可匹配。

---

## 3. 控制流平坦化还原

### 3.1 还原算法

来源：`webcrack/packages/webcrack/src/deobfuscate/control-flow-switch.ts`

```typescript
// 来源：webcrack - deobfuscate/control-flow-switch.ts
BlockStatement: {
  exit(path) {
    if (!matcher.match(path.node)) return;

    // Step 1: 建立 case编号 → 语句列表 的映射
    const caseStatements = new Map(
      cases.current!.map((c) => [
        (c.test as t.StringLiteral).value,    // case "0", case "1", ...
        // 移除末尾的 continue 语句（循环体内的continue只是跳回while）
        t.isContinueStatement(c.consequent.at(-1))
          ? c.consequent.slice(0, -1)
          : c.consequent
      ])
    );

    // Step 2: 按序列字符串指定的顺序重排语句
    const sequence = sequenceString.current!.split('|');  // "2|4|3|0|1" → ["2","4","3","0","1"]
    const newStatements = sequence.flatMap((s) => caseStatements.get(s)!);

    // Step 3: 替换原始BlockStatement的前3个语句（序列定义+迭代器+while循环）
    path.node.body.splice(0, 3, ...newStatements);
    this.changes += newStatements.length + 3;
  }
}
```

### 3.2 还原前后对比

```javascript
// === 还原前：控制流平坦化 ===
const _0xa = "2|4|3|0|1".split("|");
let _0xb = 0;
while (true) {
  switch (_0xa[_0xb++]) {
    case "0": console.log("第四步"); continue;
    case "1": console.log("第五步"); continue;
    case "2": console.log("第一步"); continue;
    case "3": console.log("第三步"); continue;
    case "4": console.log("第二步"); continue;
  }
  break;
}

// === 还原后：扁平代码 ===
console.log("第一步");
console.log("第二步");
console.log("第三步");
console.log("第四步");
console.log("第五步");
```

**算法核心**：`splice(0, 3, ...newStatements)` —— 用按正确顺序排列的语句替换掉"序列定义+迭代器+while循环"这三个结构语句。

---

## 4. 字符串数组识别

### 4.1 识别模式

来源：`webcrack/packages/webcrack/src/deobfuscate/string-array.ts`

混淆器通常将字符串提取到一个大数组中，通过包装函数访问：

```javascript
// 混淆器生成的典型模式
function getStringArray() {
  var array = ["hello", "world", "base64str", ...];
  return (getStringArray = function () { return array; })();
}
```

### 4.2 识别代码

```typescript
// 来源：webcrack - deobfuscate/string-array.ts
const functionName = m.capture(m.anyString());
const arrayIdentifier = m.capture(m.identifier());
const arrayExpression = m.capture(
  m.arrayExpression(m.arrayOf(m.or(m.stringLiteral(), undefinedMatcher)))
);

// 模式1：函数赋值形式
const functionAssignment = m.assignmentExpression(
  '=',
  m.identifier(m.fromCapture(functionName)),
  m.functionExpression(undefined, [],
    m.blockStatement([m.returnStatement(m.fromCapture(arrayIdentifier))])
  )
);

// 匹配两种变体
const matcher = varFunctionOrDeclaration(
  m.identifier(functionName), [],
  m.or(
    // 变体A：return (getStringArray = function() { return array; })()
    m.blockStatement([
      variableDeclaration,
      m.returnStatement(m.callExpression(functionAssignment))
    ]),
    // 变体B：getStringArray = function() { return array; }; return getStringArray()
    m.blockStatement([
      variableDeclaration,
      m.expressionStatement(functionAssignment),
      m.returnStatement(m.callExpression(m.identifier(functionName)))
    ])
  )
);
```

### 4.3 简单数组内联

对于不使用解码器的简单混淆（只有 `array[0]`、`array[1]` 这样的直接索引），可以直接内联：

```typescript
// 来源：webcrack - deobfuscate/string-array.ts
if (variableDeclaration.match(path.node)) {
  const binding = path.scope.getBinding(arrayIdentifier.current!.name)!;
  const memberAccess = m.memberExpression(
    m.fromCapture(arrayIdentifier),
    m.numericLiteral(m.matcher((value) => value < length))
  );
  // 检查：数组只被索引访问，且对象是只读的
  if (!binding.referenced || !isReadonlyObject(binding, memberAccess)) return;
  inlineArrayElements(arrayExpression.current!, binding.referencePaths);
  path.remove();
}
```

**安全校验**：`isReadonlyObject` 确保数组没有被 `push`、`splice` 等修改操作，否则内联会导致错误。

---

## 5. VM沙箱解码技术

### 5.1 为什么需要VM沙箱

字符串解码函数通常包含复杂的位运算、Base64变体、XOR等操作。静态分析还原这些算法成本极高，更实用的方法是：**在受控VM中直接执行解码函数**。

```javascript
// webcrack的VM解码流程
const sandbox = createNodeSandbox();  // 或 createBrowserSandbox()
const vm = new VMDecoder(sandbox, stringArray, decoders, rotator);

// 对每个 CallExpression 尝试解码
// 如果 _0xabc(0x12, 0x34) 返回字符串 → 替换为 StringLiteral
await applyTransformAsync(ast, inlineDecodedStrings, { vm });
```

### 5.2 VM沙箱的安全隔离

webcrack支持两种沙箱模式：

| 模式 | 实现 | 优势 | 限制 |
|------|------|------|------|
| Node沙箱 | `vm.createContext()` | 快速、无需浏览器 | 无法执行引用浏览器API的代码 |
| 浏览器沙箱 | Headless浏览器 | 完整执行环境 | 启动慢、资源消耗大 |

### 5.3 通用VM解码模式

当不使用webcrack时，可以手动构建VM沙箱：

```javascript
// 通用的VM沙箱解码模式
const vm = require('vm');

function decodeStrings(code, decoderPattern) {
  const ast = parse(code);
  
  // Step 1: 提取字符串数组和解码函数的源代码
  let stringArrayCode = '';
  let decoderCode = '';
  traverse(ast, {
    VariableDeclaration(path) {
      if (isArrayExpression(path.node.declarations[0]?.init)) {
        stringArrayCode = generate(path.node).code;
      }
    },
    FunctionDeclaration(path) {
      if (matchesDecoderPattern(path.node)) {
        decoderCode = generate(path.node).code;
      }
    }
  });

  // Step 2: 在VM沙箱中加载
  const sandbox = { console, parseInt, atob, btoa, String, Array };
  vm.createContext(sandbox);
  vm.runInContext(`${stringArrayCode}; ${decoderCode};`, sandbox);

  // Step 3: 遍历所有调用点，尝试执行并替换
  const replacements = [];
  traverse(ast, {
    CallExpression(path) {
      try {
        const callCode = generate(path.node).code;
        const result = vm.runInContext(callCode, sandbox);
        if (typeof result === 'string' && result.length > 0) {
          replacements.push({ path, value: result });
        }
      } catch (e) {
        // 解码失败则跳过
      }
    }
  });

  // Step 4: 批量替换
  for (const { path, value } of replacements) {
    path.replaceWith(t.stringLiteral(value));
  }
  return generate(ast).code;
}
```

---

## 6. 控制流对象还原

除了 switch-case 平坦化，混淆器还使用**对象属性查找**来打乱控制流：

```javascript
// 混淆前
if (x > 0) { doSomething(); }

// 混淆后：用对象属性替换布尔判断
var _0xctrl = {
  'abcde': function(a, b) { return a > b; },
  'fghij': function(a) { return !a; }
};
if (_0xctrl['abcde'](x, 0)) { doSomething(); }
```

webcrack的 `controlFlowObject` 变换会识别这种模式，将对象属性调用还原为直接运算。

---

## 7. 死代码移除

混淆器注入的死代码通常有几种模式：

```javascript
// 模式1：永远不会执行的分支
if (false) { /* 大量垃圾代码 */ }

// 模式2：无条件return后的代码
function f() { return result; /* 死代码 */ }

// 模式3：永假条件
if ('undefined' !== typeof _0xdead) { /* 实际执行 */ }
else { /* 死代码，但看起来像正常逻辑 */ }
```

webcrack的 `deadCode` 变换会识别这些模式并安全移除。

---

## 8. 反混淆最佳实践

| 步骤 | 动作 | 工具 | 注意事项 |
|------|------|------|---------|
| 1 | 自动反混淆 | `npx webcrack input.js` | 先让自动化工具处理能处理的 |
| 2 | 检查残留 | 阅读webcrack输出 | 多层混淆可能需要多次运行 |
| 3 | 手动处理 | Babel AST + VM沙箱 | 针对webcrack未覆盖的混淆模式 |
| 4 | 运行时验证 | 浏览器断点 | 对比反混淆前后的执行结果 |

**关键原则**：先自动后手动，先静态后动态，先还原再理解。
