# JSVMP架构深度解析：从操作码到VM执行

> 本文从两个开源JSVMP项目（jsvmp和twisted）中提取核心架构模式，对比分析栈式VM的设计哲学、操作码体系、编译器和IR混淆技术。

## 1. 操作码体系对比

### 1.1 jsvmp操作码表（38条，栈式VM）

jsvmp采用经典的分组编码方案，操作码按功能分段，便于逆向时快速分类：

| 分组 | 操作码 | 十六进制范围 | 说明 |
|------|--------|-------------|------|
| 栈操作 | PUSH, POP, DUP | 0x01-0x03 | 基本栈操纵 |
| 算术运算 | ADD, SUB, MUL, DIV, MOD, NEG | 0x10-0x15 | 6种算术操作 |
| 位移运算 | SHL, SHR, USHR | 0x16-0x18 | 含无符号右移 |
| 位运算 | BIT_AND, BIT_OR, BIT_XOR, BIT_NOT | 0x19-0x1C | 完整位操作集 |
| 比较运算 | EQ, NE, LT, LE, GT, GE | 0x20-0x25 | 严格相等(===) |
| 逻辑运算 | AND, OR, NOT, TYPEOF | 0x30-0x33 | typeof单独一条 |
| 变量操作 | LOAD, STORE, DECLARE | 0x40-0x42 | 闭包优先查找 |
| 控制流 | JMP, JIF, JNF | 0x50-0x52 | 条件跳转(true/false) |
| 函数操作 | CALL, RET, ENTER, LEAVE, CALL_METHOD | 0x60-0x64 | 含方法调用(this绑定) |
| 对象操作 | NEW_OBJ, GET_PROP, SET_PROP, NEW | 0x70-0x73 | 含构造函数 |
| 数组操作 | NEW_ARR, GET_ELEM, SET_ELEM | 0x80-0x82 | 数组元素访问 |
| 异常处理 | THROW, TRY, CATCH, FINALLY, END_TRY, BREAK, CONTINUE | 0x90-0x96 | 异常+循环控制 |
| 特殊 | NOP, HALT | 0x00, 0xFF | 首尾标记 |

**设计分析**：jsvmp用0x10/0x20/0x30等"整十"分界，给每组留了扩展空间。逆向时看到操作码值就能判断功能域，这是工程化的设计。

### 1.2 twisted操作码表（55条，含高级特性）

twisted在基础VM之上增加了现代JavaScript特性：

```typescript
// 来源：twisted/src/constant.ts
const enum Opcode {
    Push = 0x00, Pop = 0x01, Add = 0x02, Sub = 0x03,
    Mul = 0x04, Div = 0x05, Equal = 0x06,
    Jmp = 0x07, JmpIf = 0x08,
    Store = 0x09, Load = 0x0a,
    Apply = 0x0b,       // 方法调用（带this）
    Dependency = 0x0c,  // 外部依赖注入
    Property = 0x0d,    // 属性访问
    PushFrame = 0x0e, PopFrame = 0x0f, // 调用帧管理
    Halt = 0x10,
    BuildArray = 0x11, BuildObject = 0x12,
    LoadParameter = 0x13,
    Await = 0x14,       // async/await支持
    Construct = 0x15,   // new操作符
    MakeClosure = 0x22,  // 闭包创建
    LoadCapture = 0x23,  // 闭包变量访问
    InvokeValue = 0x24,  // 动态调用
    ForInInit = 0x32, ForInHas = 0x33, ForInNext = 0x34, // for-in循环
    Throw = 0x37, LandingPad = 0x38, // 异常处理
}
```

**关键差异**：twisted的操作码编号是连续的（0x00-0x38），没有jsvmp那样的分组间距。这意味着twisted更注重紧凑性，而jsvmp更注重可读性。

### 1.3 核心差异总结

| 维度 | jsvmp | twisted |
|------|-------|---------|
| 操作码数量 | 38条 | 55条 |
| 编码方案 | 分组间距(0x10/0x20...) | 连续编号(0x00-0x38) |
| 闭包支持 | 通过`_closureEnv`运行时捕获 | 专用`MakeClosure`+`LoadCapture`指令 |
| async支持 | 无 | `Await`操作码 |
| 常量存储 | ConstantPool对象（去重优化） | 内联到字节码流 |
| IR混淆 | 无 | SSA IR + ArithmeticDeformation Pass |

## 2. VM执行循环核心模式

### 2.1 jsvmp的switch-case派发

jsvmp采用经典的while+switch派发模式，代码直观但容易被静态分析：

```javascript
// 来源：jsvmp/src/vm.js - VirtualMachine.execute()
execute(bytecode, context = {}, resetGlobals = true) {
    this.bytecode = bytecode;
    this.pc = 0;
    this.stack = [];
    this.callStack = [];
    this.instructionCount = 0;

    while (this.pc < this.bytecode.instructions.length) {
        const instruction = this.bytecode.instructions[this.pc];

        // 死循环保护：限制最大指令执行数
        this.instructionCount++;
        if (this.instructionCount > this.maxInstructions) {
            throw new Error(`执行指令数量超过限制 (${this.maxInstructions})`);
        }

        this.executeInstruction(instruction);
        this.pc++;

        if (instruction.opcode === OpCodes.HALT) break;
    }
}

// 单条指令派发：switch-case
executeInstruction(instruction) {
    const { opcode, operand } = instruction;
    switch (opcode) {
        case OpCodes.PUSH: return this.executePush(operand);
        case OpCodes.ADD:  return this.executeAdd();
        case OpCodes.LOAD: return this.executeLoad(operand);
        case OpCodes.CALL: return this.executeCall(operand);
        // ... 38条操作码
        default:
            throw new Error(`未实现的操作码: ${opcode}`);
    }
}
```

**为什么这样设计**：switch-case在V8中会被优化为跳转表（jump table），性能接近O(1)。但这种模式对逆向者非常友好——只需把switch分支提取出来就能建立完整的操作码映射。

### 2.2 twisted的handler表派发

twisted使用函数表替代switch-case，增加了间接性：

```typescript
// 来源：twisted/src/vm/vm.ts
class VM {
    private handlers: Record<number, any>;

    constructor(bytecode: number[], meta: string[] = [], dependencies: object[] = []) {
        this.handlers = {
            [Opcode.Push]: this.opPush.bind(this),
            [Opcode.Add]: this.opAdd.bind(this),
            [Opcode.Load]: this.opLoad.bind(this),
            [Opcode.MakeClosure]: this.opMakeClosure.bind(this),
            // ... 55条操作码全部绑定
        };
    }

    public async execute() {
        while (this.reader.hasNext()) {
            const opcode = this.reader.read();
            await this.handlers[opcode]?.(); // 间接调用
        }
        return this.context.frame.stack.peek();
    }
}
```

**为什么这样设计**：handler表可以在运行时动态替换（比如插入hook），且代码结构更清晰。但从逆向角度看，只需dump `handlers`对象就能还原映射关系。

### 2.3 闭包实现对比

**jsvmp的运行时闭包捕获**：在函数声明时拷贝当前作用域的所有变量到`_closureEnv` Map中：

```javascript
// 来源：jsvmp/src/vm.js - createClosureEnvironment()
createClosureEnvironment(func, frame) {
    func._closureEnv = new Map();
    // 1. 捕获当前调用帧的局部变量
    for (const [varName, varValue] of frame.locals) {
        if (varName !== 'this' && varName !== 'arguments') {
            func._closureEnv.set(varName, varValue);
        }
    }
    // 2. 捕获外层调用栈的变量（支持嵌套闭包）
    for (let i = this.callStack.length - 2; i >= 0; i--) {
        const outerFrame = this.callStack[i];
        for (const [varName, varValue] of outerFrame.locals) {
            if (!func._closureEnv.has(varName)) {
                func._closureEnv.set(varName, varValue);
            }
        }
    }
}
```

**twisted的指令级闭包**：通过`MakeClosure`指令在字节码层面显式创建闭包，捕获变量列表编码在指令流中：

```typescript
// 来源：twisted/src/vm/vm.ts - opMakeClosure()
private opMakeClosure() {
    const entryPc = this.reader.read();
    const numCaptures = this.reader.read();
    const caps: unknown[] = [];
    for (let i = 0; i < numCaptures; i++) {
        const slot = this.reader.read();
        caps.push(this.context.frame.variables.get(slot));
    }
    // 创建真正的async函数，挂载$pc/$caps标记供VM调度
    const fn = async (...args: unknown[]) => {
        const subVm = new VM(bytecode, meta, deps);
        return subVm.executeClosure(entryPc, caps, args);
    };
    (fn as any).$pc = entryPc;
    (fn as any).$caps = caps;
    this.context.frame.stack.push(fn);
}
```

**设计差异**：jsvmp的闭包是全量快照（值拷贝），twisted的闭包是精确捕获（只捕获用到的slot）。twisted的方案更省内存，但编译器需要做变量活跃性分析。

## 3. 编译器关键模式

### 3.1 作用域管理与回填补丁

jsvmp编译器的核心技巧是"先占位后回填"（backpatching），用于处理前向跳转：

```javascript
// 来源：jsvmp/src/compiler.js - compileIfStatement()
compileIfStatement(node) {
    this.compileNode(node.test);
    const elseJump = this.getCurrentAddress();   // 记录占位位置
    this.bytecode.addInstruction(OpCodes.JNF, 0); // 先填0，稍后回填
    this.compileNode(node.consequent);

    if (node.alternate) {
        const endJump = this.getCurrentAddress();
        this.bytecode.addInstruction(OpCodes.JMP, 0);
        this.patchInstruction(elseJump, this.getCurrentAddress()); // 回填else地址
        this.compileNode(node.alternate);
        this.patchInstruction(endJump, this.getCurrentAddress());  // 回填end地址
    } else {
        this.patchInstruction(elseJump, this.getCurrentAddress());
    }
}

// 回填工具方法
patchInstruction(address, operand) {
    this.bytecode.instructions[address].operand = operand;
}
```

### 3.2 函数编译与跳转跳过

函数体被内联到主字节码流中，通过JMP跳过函数体，函数对象记录startAddress：

```javascript
// 来源：jsvmp/src/compiler.js - compileFunctionDeclaration()
compileFunctionDeclaration(node) {
    const jumpIndex = this.bytecode.getInstructionCount();
    this.bytecode.addInstruction(OpCodes.JMP, 0);  // 跳过函数体

    const functionStartAddress = this.bytecode.getInstructionCount();
    const funcInfo = {
        name: node.id.name,
        params: node.params.map(p => p.name),
        startAddress: functionStartAddress,        // VM执行入口
        closureScope: this.captureClosure()
    };

    this.enterScope();
    this.compileNode(node.body);
    this.bytecode.addInstruction(OpCodes.RET);
    this.exitScope();

    // 回填JMP目标（跳过整个函数体）
    this.bytecode.instructions[jumpIndex].operand = this.bytecode.getInstructionCount();
    // 将函数对象存入常量池
    const funcIndex = this.bytecode.addConstant(funcInfo);
    this.bytecode.addInstruction(OpCodes.PUSH, funcIndex);
    this.bytecode.addInstruction(OpCodes.DECLARE, nameIndex);
}
```

## 4. IR混淆技术（twisted）

twisted的核心创新是在字节码编译前引入SSA IR中间层，通过混淆Pass管线增加逆向难度。

### 4.1 算术恒等变形

`ArithmeticDeformationPass`对相邻的Load/Push+运算指令序列进行等价变换：

```typescript
// 来源：twisted/src/obfuscator/passes/arithmetic.ts

// Push双常量变形：x+y = (x-t)+(y+t) 或 (x+t)+(y-t)
// 原始：Push x; Push y; Add
// 变形：Push (x-t); Push (y+t); Add  （结果不变）

// Load双变量展开：A+B = (A-t)+(B+t)
// 原始：Load A; Load B; Add  （3条指令）
// 变形：Load A; Push t; Sub; Load B; Push t; Add; Add  （7条指令）
function expandLoadLoadAdd(ua, ub, t, alt) {
    if (!alt) {
        return [
            createInstruction(Opcode.Load, [createArg(ua.kind, sa)]),
            createInstruction(Opcode.Push, [createArg(ArgKind.Number, t)]),
            createInstruction(Opcode.Sub, []),    // A - t
            createInstruction(Opcode.Load, [createArg(ub.kind, sb)]),
            createInstruction(Opcode.Push, [createArg(ArgKind.Number, t)]),
            createInstruction(Opcode.Add, []),    // B + t
            createInstruction(Opcode.Add, []),    // (A-t) + (B+t) = A+B
        ];
    }
    // 另一种变体：(A+t) + (B-t) = A+B
}
```

**对抗方法**：这类混淆可以用代数简化规则消除——检测"Push t; Sub; Push t; Add"模式并折叠为NOP。twisted自带的`assembler/hyperion.ts`包含反混淆Pass。

### 4.2 混淆Pass管线设计

twisted的IR混淆采用编译器Pass管线架构，每个Pass独立变换IR，可组合叠加：
- **ArithmeticDeformationPass**：算术恒等变形（上文）
- **常量拆分**：将数值常量拆分为多个运行时计算的子表达式
- **变量加载展开**：简单的Load变为多步间接寻址
- **控制流重组**：线性代码块拆分为通过跳转表连接的基本块

这种设计的启示是：高级JSVMP保护本质上是一个编译器优化问题的反面——正常编译器优化代码性能，混淆编译器优化代码不可读性。
