# 白盒分支诊断：精准定位复刻差异

## 概述
当黑盒复刻（Node.js环境修补后运行原始代码）的输出与浏览器不一致时，
使用分支诊断法快速定位差异点，而非盲目调试。

## 诊断流程

### Step 1：记录浏览器端执行路径
在Chrome DevTools中使用LogPoint记录所有分支决策：
```javascript
// 在每个 if/switch 处设置LogPoint
// 格式：`BRANCH: ${condition} → ${result}`
```

### Step 2：记录Node端执行路径
在Node复刻代码中插入相同格式的日志：
```javascript
console.log(`BRANCH: ${condition} → ${result}`);
```

### Step 3：对比两条路径
```bash
# 使用diff工具对比
diff browser_trace.log node_trace.log | head -20
```

### Step 4：定位首个分歧点
diff输出的第一个差异行就是问题根源。通常原因：
- 环境变量不一致（navigator/screen/location属性值）
- 时间戳精度不同
- 随机数种子不同
- 类型隐式转换差异（浏览器vs Node的==行为）

### Step 5：修正并验证
修正该分支的条件或输入后，重新运行验证。

## 常见差异原因

| 差异表现 | 常见原因 | 修复方法 |
|---------|---------|---------|
| 同一函数返回值不同 | 环境API返回值差异 | 补全该API的mock |
| 分支走了不同路径 | 类型检测失败（typeof/instanceof） | 修正对象原型链 |
| 数值精度不同 | 浮点运算平台差异 | 使用相同精度截断 |
| 时间相关值不同 | Date.now()调用时机 | 固定时间戳 |
| 随机值不同 | Math.random()未mock | 固定随机种子 |

## 自动化诊断工具

使用WebTrace的`hook_api`可以自动记录浏览器端的分支决策路径，
配合Node端的日志输出，实现半自动化的diff诊断。

### 浏览器端自动记录
```javascript
// 通过hook_api设置LogPoint
// hook所有条件表达式的求值结果
// 输出格式统一为：FILE:LINE CONDITION=VALUE
```

### Node端对应日志
```javascript
// 在环境修补代码中添加trace模式
const TRACE_MODE = process.env.TRACE === '1';
function trace(file, line, condition, value) {
  if (TRACE_MODE) console.log(`${file}:${line} ${condition}=${value}`);
}
```

### diff对比脚本
```bash
# 生成对比报告
diff <(sort browser_trace.log) <(sort node_trace.log) > divergence.diff
# 首个差异即为根因
head -5 divergence.diff
```

## 案例：知乎__zse_ck复刻诊断

**问题**：Node生成的cookie与浏览器不一致

**诊断过程**：
1. LogPoint记录emo.js中关键分支（canvas返回值）
2. 发现：canvas.toDataURL()在Node mock中返回空字符串
3. 修正：补全canvas 2D context的完整mock
4. 验证：cookie生成一致

## 高频问题速查

| 症状 | 首查方向 | 修复模板 |
|------|---------|---------|
| 签名长度不对 | 编码方式差异（hex/base64/raw） | 统一编码输出格式 |
| 签名前半段对后半段错 | 时间戳或随机数介入点 | 固定时间+mock随机 |
| 每次结果都不同 | 未固定的随机源 | 枚举所有Math.random/crypto调用 |
| 偶发不一致 | 异步时序问题 | 确保Promise resolve顺序 |
| 特定输入才出错 | 边界条件分支 | 用出错输入单步trace |
