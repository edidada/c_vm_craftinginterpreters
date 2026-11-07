# Part III. A Bytecode Virtual Machine（clox，C）

> 第 14–30 章。从这章开始风格一变：从 Java 换成 C，从"教学清晰"换成"工业性能"。

## 第 14 章 Chunks of Bytecode

- 定义 **chunk**：字节数组 + 常量池（`Value*` 列表）—— 指令 + 数据打包。
- Zens different instructions 也一起揭示：新增 OP_RETURN 等指令的表现。
- **反汇编器**（disassembler）：把字节流解读并打印回人类可读形式 —— 是调试 VM 的 essential 工具。
- VM 的骨架：`run()` 读取指令，switch 分发，暂时只处理 RETURN 并打印。

**Design Note: Test Your Language** —— 给语言做测试。构建自动化测试基础设施（作者提到项目中有一整套 `test/` 目录的独立测试框架，是个值得投入的投资）。

## 第 15 章 A Virtual Machine

- 把 chunk 真正接上：`VM` 持有 chunk、指令指针 `ip`、**`Value*` 栈**。
- 引入栈：`PUSH`/`POP` 的宏。
- 这就是"stack-based VM"的雏形，体现在：常量是 `OP_CONSTANT` 从常量池读并 push，RETURN 把栈顶打印。
- 用 `#define READ_CONSTANT()` 等宏减少样板代码。
- `#ifdef DEBUG_TRACE_EXECUTION` 打印栈和反汇编 —— 每执行一条指令的调试快照。

**Design Note: Register-Based Bytecode** —— 栈式 vs 寄存器式 bytecode 的对比：寄存器 VM（如 Lua 5.x）指令宽、decode 复杂但指令数少；栈式简单、可移植好。本书选栈式。

## 第 16 章 Scanning on Demand

- C 版 Scanner：**按需扫描**（一次性给 token 数组 vs parser 向 scanner 请求下一个 token）。clox 选择按需，省内存、更有趣。
- 手写状态机消耗：以 switch-case 从当前字符跳转状态，比正则更快。
- token 结构体 `{ TokenType type; const char* start; int length; int line; }`。
- 数字字面量：支持 `1234.56`，但不支持 `.5` 和 `5.`（简化 C 的做法：需要前后缀）。
- 关键性能技巧：token 不复制字符串，而是**指向源代码原文的指针**。

## 第 17 章 Compiling Expressions

- clox 的编译器是**单遍编译器**（single-pass）：边解析边直接发字节码，不做 AST！
- **Pratt 解析算法**（又称为 precedence climbing）核心数据是**每个 token 一行表**：`prefix 函数, infix 函数, infix 优先级`。
- 函数式方法论：解析函数直接把字节码 emit 进 chunk；文本从源码 slice（字符串字面量的截取）。
- 优先级表格：用枚举 Precedence { NONE, ASSIGNMENT, OR, AND, EQUALITY, COMPARISON, TERM, FACTOR, UNARY, CALL, PRIMARY }。
- `parsePrecedence()` 是 Pratt 核心循环：读 prefix → 当下一个 token 的优先级够高时读 infix。
- 分组表达式、一元负号、二元运算、常量的完整 compile。

**Design Note: It's Just Parsing** —— 编译 JSON、SQL、正则、CSS 等都是 parsing；解析无处不在，学会 Pratt 解析是通用技能。

## 第 18 章 Types of Values

- 定义 Value：`typedef struct` 的 tagged union `{ ValueType type; union { bool boolean; double number; Obj* obj; } as; }`。
- 装箱/拆箱的宏：`BOOL_VAL`, `NUMBER_VAL`, `AS_BOOL` 等。
- 类型判断的 dispatch：`if (IS_NUMBER(a) && IS_NUMBER(b))`。
- **NaN boxing**：利用 IEEE 754 double 中未使用的位模式来打包任何值到 64 位里 —— 高性能解释器的常用技巧。

## 第 19 章 Strings

- **字符串对象**（Obj），支持连接（`+` 运算符重载到 OP_ADD）。
- 介绍 **string interning**（字符串驻留）：同一字符串内容在进程中只保留一份 —— 这是后续 hash table 与快速比较的基础。
- 字符串相等比较优化：**比较指针**而不是逐字符（来自 interning）。
- FNV-1a 哈希函数（简单的、质量够用的 32 位 hash）。

**Design Note: String Encoding** —— ASCII / UTF-8 / UTF-16 的取舍。Lox 简化处理：按 UTF-8 字节处理，但很多操作假定字形与字节一一对应 —— 真实产品语言需要处理 grapheme clusters 等复杂问题。

## 第 20 章 Hash Tables

- 手写 hash table：**开放寻址**（open addressing）+ **线性探测**（linear probing）。
- **tombstone**（墓碑）处理删除：删除不能简单地清空 bucket（会破坏探测链），所以要放占位的 tombstone。
- 负载因子到阈值（0.75）就 grow：分配新数组、重新安置。
- **字符串驻留的真正实现**：`strings` 这张 hash table 记录"已驻留的字符串集合"，`tableFindString` 在不分配内存的前提下查找相同内容。

## 第 21 章 Global Variables

- 全局变量 = hash table 里的一条记录（string → value）。
- 编译期把标识符编译成 `OP_DEFINE_GLOBAL` / `OP_GET_GLOBAL` / `OP_SET_GLOBAL` 指令 + 常量池索引。
- 一个有趣行为：Lox 允许**在定义一个全局前引用**（运行时若报错会停止），像 JavaScript 的动态行为。

## 第 22 章 Local Variables

- 全局变量慢（每读取都要 hash table 查找），局部变量是**栈槽**（stack slot）。
- **变量解析**（类似 jlox 的 Resolver，但这里在编译器里做）：局部变量的深度在编译期确定，编译为 `OP_GET_LOCAL`/`OP_SET_LOCAL` 直接操作栈（免 hash）。
- 块的结束（endScope）就是简单地丢弃栈顶若干槽。
- 引用未声明全局是合法的（Ch21 惰性），局部不行（编译器因此能区分）。

## 第 23 章 Jumping Back and Forth

- 条件语句不再用数据流，而用 `OP_JUMP_IF_FALSE` / `OP_JUMP` 等指令。
- **back-patching（回填）**：emit 跳转时先 emit 一个占位的操作数，跳转目标确定后**回填**进字节数组。
- while 循环是**向后跳**（`OP_LOOP`），for 循环 = 初始化 + 条件 + 体 + increment 的组合编译。

**Design Note: Considering Goto Harmful** —— Dijkstra 的名言争论；结构化编程是应对复杂度的必需，而 bytecode 的结构化跳转可以有限地视为安全的 goto。

## 第 24 章 Calls and Functions

- 函数编译时：指令 + arity + name init（作为 `ObjFunction`，常量池引用）。
- **call frame**（`CallFrame`）：保存 ip、slots 根 —— 用一个数组实现手动管理。
- `OP_CALL`：检查被调用者是不是 function，检查 arity，新开 frame，跳进函数第一字节。
- return 时恢复 caller 的 frame、弹出 frame。
- Native function 处理不同的返回路径。

## 第 25 章 Closures

- 核心问题：函数返回后，被捕获的局部变量生命周期问题。
- **Upvalue**（捕获变量）：两层结构：
  - 编译器把被闭包捕获的变量从"栈槽"变成"upvalue"，指令：`OP_GET_UPVALUE` 等。
  - 运行时有 **open upvalue**（指向栈）与 **closed upvalue**（把值复制到堆中的 `ObjUpvalue`）。
- 当栈槽离开作用域时"close"所有指向它的 upvalue（值从栈搬进堆）。

**Design Note: Closing Over the Loop Variable** —— 经典 for 循环 + 闭包 bug：循环变量是在每次迭代新绑定还是共享？JavaScript 的 `let` 方案 vs `var` 方案 vs Python 的选择。Lox 每次迭代新绑定。

## 第 26 章 Garbage Collection

- **Mark-Sweep**：从 roots（栈、CallFrames、globals、开放 upvalue 等）出发标记 → 回收未标记的。
- **gray / black / white** 三色标记 + `grayStack`。
- 收集时机：在分配对象前触发（按内存阈值 / 或下一次 GC）。
- `freeObject` 分类释放（字符串、函数、实例、class、closure 等）。
- C 中做 GC 的麻烦：roots 在 C 栈里，要扫描 C 栈（用地址上界）。**注意**：C 编译器可能把局部变量优化进寄存器或移除，导致丢 roots —— 这就是为什么 `markValue` 前会把根缘先 push，并严禁临时变量驻留堆引用不清理。

**Design Note: Generational Collectors** —— 分代 GC 的思路（弱分代假说）；mark-sweep 虽简单但每次全堆扫描对大堆不友好。

## 第 27 章 Classes and Instances

- `ObjInstance` 持有 `ObjClass*` 和 `Table fields`。
- `class` 表达式：`OP_CLASS` 创建 ObjClass；`OP_DEFINE_GLOBAL` 绑定名字。
- 实例的字段操作：`OP_GET_PROPERTY`（先查字段，再查方法），不存在且不是类方法 → runtime error。
- `OP_SET_PROPERTY` 只允许在已声明的字段上设置（与 class 一致、不允许随意加字段）。

## 第 28 章 Methods and Initializers

- 方法定义：`OP_METHOD` 把一个 ObjFunction 绑成 **ObjBoundMethod**（receiver + function）放进行为表。
- **`init` 方法**：调用上与普通方法区别对待，返回 `this` 而不是 nil —— `new` 的语法糖实现。
- 加快调用时 `init` 的 arity 不匹配检查。

**Design Note: Novelty Budget** —— 设计语言时有一个"新奇预算"（novelty budget）：一次只能引入一种突破性的新概念，太多新东西没人学得会。

## 第 29 章 Superclasses

- `class A < B` 时 emit `OP_INHERIT`：把超类的 methods **复制**到子类表。
- `super.method`：编译期就把方法的"receiver"换成了当前对象的 class 的超类方法表，指令为 `OP_GET_SUPER`。
- **super call**（`super.method(args)` 语法）：其实是 super get + call 的组合，编译成 `OP_SUPER_INVOKE`（合并性能优化）。
- super 的绑定通过 `ObjClass` + `receiver` 保存。

## 第 30 章 Optimization

最后一章：从"正确"走向"快"。

- 建立 benchmark 工程、用性能剖析找热点。
- 数值优化：NaN boxing 让 Value 只占 8 字节 vs 之前 tagged union 16 字节 → cache 友好。
- 栈上**缓存热点变量**，减少重复解引用。
- hash table 的优化：resize 时机、inline value 查找。
- 关键点：**优化前先 profile**，避免凭直觉改。
- 建议进阶方向：寄存器 VM、内联缓存（inline caching）、JIT compile threads、去 obfuscating。
