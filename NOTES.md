# Crafting Interpreters 读书笔记

> 《Crafting Interpreters》— Robert Nystrom 著，免费在线阅读：<https://craftinginterpreters.com/>
> 原始代码仓库：<https://github.com/munificent/craftinginterpreters>（本仓库已将其作为 git submodule 引入）

---

## 总览

这本书通过**手写一门语言 Lox 的两个解释器**来讲解解释器原理：

| 解释器 | 语言 | 风格 | 章节 |
|---|---|---|---|
| **jlox** | Java | 树遍历（tree-walk）解释器，学习概念的脚手架 | 4 – 13 章 |
| **clox** | C | 字节码 + 虚拟机（VM），追求真正可用的性能 | 14 – 30 章 |

先在 jlox 里用最直观的方式理解每个概念，再用 clox 重新实现一遍，体会"工程化"带来的复杂度与收益。这是本书最核心的教学设计。

一条主线流水线（贯穿全书的地图）：

```
源代码 → 词法扫描(Scanner) → 语法分析(Parser) → AST → 语义分析/解析(Resolver)
      → 求值(Interpreter)                    [jlox, 树遍历]

源代码 → 扫描(Scanner) → 编译器(Compiler, 单遍) → 字节码(Chunk) → 虚拟机(VM) → 执行
                                        [clox, 栈式 VM]
```

---

## 第一部分：Welcome（欢迎）

### 第 1 章 Introduction

- 为什么学解释器：**解释器是"图灵柱"** —— 能在单一程序里压缩大量计算逻辑。
- 三个理由：处理其他语言（SQL、日志 DSL 等）、提升对 C#/Java 等语言的理解、以及纯粹的智力快感。
- 本书的语言 Lox 是动态类型、GC、有类的脚本语言。
- 教学顺序：从孤儿语言特性到完整实现，先功后名。

**Design Note: What's in a Name?** —— 给语言命名是个 Marketing 问题，名字会随时间改变对语言的感觉（Lox 一词无所指也没关系，像 Java 也只是为了好玩）。

### 第 2 章 A Map of the Territory

解释器实现技术的地图（本章纯理论，不写代码）：

- **编译 vs 解释**：编译是**翻译**到更低层；解释是**执行**但逐段分析。
- **流水线各阶段**：
  1. *Scanning / Lexing*：字符 → token 流。
  2. *Parsing*：token → AST（自顶向下递归下降 / Pratt）。
  3. *Semantic analysis*：绑定变量、类型检查（Lox 里是变量→声明点的解析）。
  4. *Interpreting*：
     - **Tree-walk**：直接遍历 AST（jlox 的路）。
     - **Transpilers**：翻译到另一种高级语言源码。
     - **Bytecode compilers + VM**：翻译成自定义指令集，由 VM 循环执行（clox 的路）。
     - **JIT**：运行时把热点代码编译成机器码（JavaScript 引擎的方法）。
- 关键洞见：所有这些技术只是**语言实现工程**的不同位置的选择，一一映射到工具链上。

**Design Note: 表达式 vs 语句** —— Lox 把声明和表达式分开，牺牲了灵活性换取可读性；Lisp/Ruby 等让一切是表达式。

### 第 3 章 The Lox Language

Lox 语言特性（相当于"产品需求文档"）：

- 动态类型、原生支持 GC、支持函数式（一等函数）+ 面向对象。
- 语法上极简：C 风格注释、分号结尾、块、if/while/for、函数、类（单继承）、`return`。
- 只有三种副作用允许变量绑定：声明、赋值、变量绑定到函数参数。
- 一个类的最小序列：`data + behavior`（字段 + 方法）。
- 语义上特别注意：变量作用域是**块级**（C 风格），但**没有变量提升**。
- `print` 是内建语句而不是函数（简化设计）。

**Design Note: Implicit Variable Declaration** —— 为什么 Lox 需要显式 `var`。省略它靠约定（如 Python 的赋值即声明），但语义会产生全局作用域陷阱，Lox 选择更严格。

---

## 第二部分：A Tree-Walk Interpreter（jlox，Java）

### 第 4 章 Scanning

- Token 的结构：关键字、标识符、数字、字符串、双字符运算符（`!=`、`<=`）等。
- 手写 Scanner：一次文本 → token 列表，用 `switch-case` 完成 DFA 的状态转换。
- 关键点：字符串和数字字面量的解析；行尾追踪（用于报错信息）。
- 错误处理：Error 类只负责收集（line + message），区分 `ScanError`/`ParseError`。

**Design Note: Implicit Semicolons** —— 讨论省略分号还是让newline代替（像 JavaScript 的 ASI 的坑），Lox 保留分号但允许染色（染色使得省略换行作为分隔）。

### 第 5 章 Representing Code

- 引入 AST：用类表示语法树（每个节点都是 Expr 的子类）。
- **元编程**：写一个 `GenerateAst.java` 程序自动生成 Expr 类（Binary、Grouping、Literal、Unary、Assign、Variable、Logical、Call、Get、Set、This、Super…），避免手动重复代码。
- **Visitor 模式**：为每种节点添加访问逻辑，避免在每个类上写 `accept`。

**Design Note: Logic Versus History** —— 语言 Abdullah 里 `a + b = c` 的合法性讨论：语法设计受历史 (FORTRAN) 而非逻辑影响。

### 第 6 章 Parsing Expressions

- **递归下降解析**（Recursive Descent）：每个语法产生式对应一个函数；一个个 token 向前推进，失败时通过提前匹配 token 类型做**分支预测**。
- 关键技能：把优先级用**表达式语法分层**（assignment → expression → equality → comparison → term → factor → unary → primary）。
- 错误恢复：**panic mode + 同步**。在匹配错时跳过到下一条语句的开始 token（`;` 等），继续恢复。
- Pratt parsing（本章引入但实际在 clox 中使用更深入）。

**Design Note: Logic Versus History**（本章实际标题）—— 讨论赋值是否是表达式（C 的经典设计陷阱）。

### 第 7 章 Evaluating Expressions

- `Interpreter` 实现 Visitor，用**一个方法一个 case** 来 eval 表达式树。
- **truthiness**：Lox 中 `false` 和 `nil` 是假，其他一切（包括 `0`、`""`）都是真。
- 二元运算的**操作数类型检查**：这一步在运行时才发生（动态类型），错误处理用 `RuntimeError` 异常，在 REPL 外层捕获。
- `==` 的定义：Lox 中不同类型永不相等（`"1" != 1`）。

**Design Note: Static and Dynamic Typing** —— 静态类型 vs 动态类型的哲学讨论：安全性 vs 灵活性，"类型是最好的文档"。

### 第 8 章 Statements and State

- 增加**语句**：`expression statement`、`print statement`、声明。
- **Environment**：一个 `HashMap<String, Object>`，负责变量名到值的绑定；**作用域链**由嵌套 Environment 形成（`enclosing` 指针）。
- 赋值即"在环境里找到该名字并替换值"，然后返回右值。
- `ResolveError`/`RuntimeError` 的分工。
- REPL：解析到 EOF 就退出；支持" expressions + statements 混合"，没有分号的表达式也直接求值打印（结果用 `(expr)` 语法 ? 不 —— 用表达式语句兼 REPL 的糖）。

**Design Note: Implicit Variable Declaration** —— 赋值一个未声明变量是定义还是错误？Lox 选择报错（严格），避免打错变量名产生"幽灵变量"。

### 第 9 章 Control Flow

- 添加 `if/else`、逻辑运算符 `and/or`（**短路求值**：它们是语法糖级的"控制流表达式"）、`while`、`for`。
- `for` 被 desugar 成 `while` 的 Idea：语法糖让使用者写起来更舒服，但解释器内部其实没有 `for` 的独立表示。

**Design Note: Spoonfuls of Syntactic Sugar** —— 语言加糖能增加表达力但增加学习成本和实现复杂度。Lox 的原则：糖必须**可以被 desugar 成已有语法**（`for` desugar 成 `while`），这样就不需要两条执行路径。

### 第 10 章 Functions

- 函数声明、调用表达式 `callee(args)`。
- LoxFunction 自己持有 `closure`（环境）。
- 调用流程：call → 新环境（把参数绑定）→ 执行 body → `return` 值（没有 return 就是 `nil`）。
- Native functions（`clock()`）：用 Java 接口模拟宿主函数，区别于用户函数。

### 第 11 章 Resolving and Binding

- **语义分析 / Resolver** 第二遍遍历：绑定每个变量**使用**到它的**声明**位置。
- 核心数据：给每个 Variable/Assign 节点标注一个**深度数**，运行时不用沿环境链逐层查找，直接按数 hopping。
- 作用域：`beginScope()`/`endScope()` 镜像块的嵌套。
- Lox 的静态作用域规则 detail：**在声明前引用**（如 `var a = a;`）是不合法的 —— 解析为错误，虽然 JavaScript 里它会"成功"读到 `undefined`。
- `this` 也用同样机制：`this` 也是"把 `this` 绑定为一个特殊局部"。

### 第 12 章 Classes

- `class Name { method(); }`。
- 类 = 一组方法 + `init` 构造器；类不是值（不能赋值给变量？可以赋值，但 class 表达式只能用于声明…… 实际上 jlox 里 class 是 statement）。
- **方法绑定**：把 receiver 绑定为 `this` 并存进闭包 —— 这是"方法即闭包"的统一（函数式 + 面向对象互相解释）。
- 字段是实例上的属性，读写用 `Get`/`Set` 表达式。

**Design Note: Prototypes and Power** —— 基于类的语言 vs 基于原型（prototype-based，如 JavaScript）的语言：后者更表达能力更强但风险更大，大部分程序员在类的模型里更轻松。

### 第 13 章 Inheritance

- `<` 进行继承：`class A < B { ... }` —— 把超类的方法复制进子类的方法表。
- `super.method()`：静态解析为超类方法表里的查找。
- 重要细节：**重写方法时要重新绑定 receiver**，且 super 的绑定要求方法所在对象仍然持有 `this`。

---

## 第三部分：A Bytecode Virtual Machine（clox，C）

从这章开始风格一变：从 Java 换成 C，从"教学清晰"换成"工业性能"。

### 第 14 章 Chunks of Bytecode

- 定义 **chunk**：字节数组 + 常量池（`Value*` 列表）—— 指令 + 数据打包。
- Zens different instructions 也一起揭示：新增 OP_RETURN 等指令的表现。
- **反汇编器**（disassembler）：把字节流解读并打印回人类可读形式 —— 是调试 VM 的 essential 工具。
- VM 的骨架：`run()` 读取指令，switch 分发，暂时只处理 RETURN 并打印。

**Design Note: Test Your Language** —— 给语言做测试。构建自动化测试基础设施（作者提到项目中有一整套 `test/` 目录的独立测试框架，是个值得投入的投资）。

### 第 15 章 A Virtual Machine

- 把 chunk 真正接上：`VM` 持有 chunk、指令指针 `ip`、**`Value*` 栈**。
- 引入栈：`PUSH`/`POP` 的宏。
- 这就是"stack-based VM"的雏形，体现在：常量是 `OP_CONSTANT` 从常量池读并 push，RETURN 把栈顶打印。
- 用 `#define READ_CONSTANT()` 等宏减少样板代码。
- `#ifdef DEBUG_TRACE_EXECUTION` 打印栈和反汇编 —— 每执行一条指令的调试快照。

**Design Note: Register-Based Bytecode** —— 栈式 vs 寄存器式 bytecode 的对比：寄存器 VM（如 Lua 5.x）指令宽、decode 复杂但指令数少；栈式简单、可移植好。本书选栈式。

### 第 16 章 Scanning on Demand

- C 版 Scanner：**按需扫描**（一次性给 token 数组 vs parser 向 scanner 请求下一个 token）。clox 选择按需，省内存、更有趣。
- 手写状态机消耗：以 switch-case 从当前字符跳转状态，比正则更快。
- token 结构体 `{ TokenType type; const char* start; int length; int line; }`。
- 数字字面量：支持 `1234.56`，但不支持 `.5` 和 `5.`（简化 C 的做法：需要前后缀）。
- 关键性能技巧：token 不复制字符串，而是**指向源代码原文的指针**。

### 第 17 章 Compiling Expressions

- clox 的编译器是**单遍编译器**（single-pass）：边解析边直接发字节码，不做 AST！
- **Pratt 解析算法**（又称为 precedence climbing）核心数据是**每个 token 一行表**：`prefix 函数, infix 函数, infix 优先级`。
- 函数式方法论：解析函数直接把字节码 emit 进 chunk；文本从源码 slice（字符串字面量的截取）。
- 优先级表格：用枚举 Precedence { NONE, ASSIGNMENT, OR, AND, EQUALITY, COMPARISON, TERM, FACTOR, UNARY, CALL, PRIMARY }。
- `parsePrecedence()` 是 Pratt 核心循环：读 prefix → 当下一个 token 的优先级够高时读 infix。
- 分组表达式、一元负号、二元运算、常量的完整 compile。

**Design Note: It's Just Parsing** —— 编译 JSON、SQL、正则、CSS 等都是 parsing；解析无处不在，学会 Pratt 解析是通用技能。

### 第 18 章 Types of Values

- 定义 Value：`typedef struct` 的 tagged union `{ ValueType type; union { bool boolean; double number; Obj* obj; } as; }`。
- 装箱/拆箱的宏：`BOOL_VAL`, `NUMBER_VAL`, `AS_BOOL` 等。
- 类型判断的 dispatch：`if (IS_NUMBER(a) && IS_NUMBER(b))`。
- **NaN boxing**（设计笔记中的进阶玩法）：利用 IEEE 754 double 中未使用的位模式来打包（pack）任何值到 64 位里 —— 高性能解释器的常用技巧。

### 第 19 章 Strings

- **字符串对象**（Obj），支持连接（`+` 运算符重载到 OP_ADD）。
- 介绍 **string interning**（字符串驻留）：同一字符串内容在进程中只保留一份 —— 这是后续 hash table 与快速比较的基础。
- 字符串相等比较优化：**比较指针**而不是逐字符（来自 interning）。
- FNV-1a 哈希函数（简单的、质量够用的 32 位 hash）。

**Design Note: String Encoding** —— ASCII / UTF-8 / UTF-16 的取舍。Lox 简化处理：按 UTF-8 字节处理，但很多操作假定字形与字节一一对应 —— 真实产品语言需要处理 grapheme clusters 等复杂问题。

### 第 20 章 Hash Tables

- 手写 hash table：**开放寻址**（open addressing）+ **线性探测**（linear probing）。
- **tombstone**（墓碑）处理删除：删除不能简单地清空 bucket（会破坏探测链），所以要放占位的 tombstone。
- 负载因子到阈值（0.75）就 grow：分配新数组、重新安置。
- **字符串驻留的真正实现**：`strings` 这张 hash table 记录 "已驻留的字符串集合"，`tableFindString` 在不分配内存的前提下查找相同内容。

### 第 21 章 Global Variables

- 全局变量 = hash table 里的一条记录（string → value）。
- 编译期把标识符编译成 `OP_DEFINE_GLOBAL` / `OP_GET_GLOBAL` / `OP_SET_GLOBAL` 指令 + 常量池索引。
- 一个有趣行为：Lox 允许**在定义一个全局前引用**（运行时若报错会停止），像 JavaScript 的动态行为。

### 第 22 章 Local Variables

- 全局变量慢（每读取都要 hash table 查找），局部变量是**栈槽**（stack slot）。
- **变量解析**（类似 jlox 的 Resolver，但这里在编译器里做）：局部变量的深度在编译期确定，编译为 `OP_GET_LOCAL`/`OP_SET_LOCAL` 直接操作栈（免 hash）。
- 块的结束（endScope）就是简单地丢弃栈顶若干槽。
- 遇到 `_variable`（未声明即引用）之 laws：引用未声明全局是合法的（上面 Ch21 惰性），局部不行（编译器因此能区分）。

### 第 23 章 Jumping Back and Forth

- 条件语句不再用数据流，而用 `OP_JUMP_IF_FALSE` / `OP_JUMP` 等指令。
- **back-patching（回填）**：emit 跳转时先 emit 一个占位的操作数，跳转目标确定后**回填**进字节数组。
- while 循环是**向后跳**（`OP_LOOP`），for 循环 = 初始化 + 条件 + 体 + increment 的组合编译。

**Design Note: Considering Goto Harmful** —— Dijkstra 的名言争论；结构化编程是应对复杂度的必需，而 bytecode 的结构化跳转可以有限地视为安全的 goto。

### 第 24 章 Calls and Functions

- 函数编译时：指令 + arity + name init（作为 `ObjFunction`，常量池引用）。
- **call frame**（`CallFrame`）：保存 ip、slots 根 —— 用一个数组实现手动管理。
- `OP_CALL`：检查被调用者是不是 function，检查 arity，新开 frame，跳进函数第一字节。
- return 时恢复 caller 的 frame、弹出 frame。
- Native function 处理不同的返回路径。

### 第 25 章 Closures

- 核心问题：函数返回后，被捕获的局部变量生命周期问题。
- **Upvalue**（捕获变量）：两层结构：
  - 编译器把被闭包捕获的变量从"栈槽"变成"upvalue"，指令：`OP_GET_UPVALUE` 等。
  - 运行时有 **open upvalue**（指向栈）与 **closed upvalue**（把值复制到堆中的 `ObjUpvalue`）。
- 当栈槽离开作用域时"close"所有指向它的 upvalue（值从栈搬进堆）。

**Design Note: Closing Over the Loop Variable** —— 经典 for 循环 + 闭包 bug：循环变量是在每次迭代新绑定还是共享？JavaScript 的 `let` 方案 vs `var` 方案 vs Python 的选择。Lox 每次迭代新绑定。

### 第 26 章 Garbage Collection

- **Mark-Sweep**：从 roots（栈、CallFrames、globals、开放 upvalue 等）出发标记 → 回收未标记的。
- **gray / black / white** 三色标记 + `grayStack`。
- 收集时机：在分配对象前触发（按内存阈值 / 或下一次 GC）。
- `freeObject` 分类释放（字符串、函数、实例、class、closure 等）。
- C 中做 GC 的麻烦：roots 在 C 栈里，要扫描 C 栈（用地址上界）。**注意**：C 编译器可能把局部变量优化进寄存器或移除，导致丢 roots —— 这就是为什么 `markValue` 前会把根缘先 push，并严禁临时变量驻留堆引用不清理。

**Design Note: Generational Collectors** —— 分代 GC 的思路（弱分代假说）；mark-sweep 虽简单但每次全堆扫描对大堆不友好。

### 第 27 章 Classes and Instances

- `ObjInstance` 持有 `ObjClass*` 和 `Table fields`。
- `class` 表达式：`OP_CLASS` 创建 ObjClass；`OP_DEFINE_GLOBAL` 绑定名字。
- 实例的字段操作：`OP_GET_PROPERTY`（先查字段，再查方法），不存在且不是类方法 → runtime error。
- `OP_SET_PROPERTY` 只允许在已声明的字段上设置（与 class 一致、不允许随意加字段）。

### 第 28 章 Methods and Initializers

- 方法定义：`OP_METHOD` 把一个 ObjFunction 绑成 **ObjBoundMethod**（receiver + function）放进行为表。
- **`init` 方法**：调用上与普通方法区别对待，返回 `this` 而不是 nil —— `new` 的语法糖实现。
- 加快调用时 `init` 的 arity 不匹配检查。

**Design Note: Novelty Budget** —— 设计语言时有一个"新奇预算"（novelty budget）：一次只能引入一种突破性的新概念，太多新东西没人学得会（考虑 Lox/Lox jlox中，语言设计的理论与产品学的跨界）。

### 第 29 章 Superclasses

- `class A < B` 时 emit `OP_INHERIT`：把超类的 methods **复制**到子类表。
- `super.method`：编译期就把方法的"receiver"换成了当前对象的 class 的超类方法表，指令为 `OP_GET_SUPER`。
- **super call**（`super.method(args)` 语法）：其实是 super get + call 的组合，`super_` 编译成 `OP_SUPER_INVOKE`（合并 性能优化）。
- super 的绑定通过 `ObjClass`+`receiver` 保存。

### 第 30 章 Optimization

最后一章：从"正确"走向"快"。

- 建立 benchmark 工程、用性能剖析找热点。
- 数值优化：NaN boxing 让 Value 只占 8 字节 vs 之前 tagged union 16 字节 → cache 友好。
- 栈上**缓存热点变量**，减少重复解引用。
- hash table 的优化：resize 时机、inline value 查找。
- 关键点：**优化前先 profile**，避免凭直觉改。
- 建议进阶方向：寄存器 VM、内联缓存（inline caching）、JIT compile threads、去 obfuscating。

---

## Backmatter

- **Appendix I: Lox Grammar** — 一页纸的语言完整语法（BNF 风格），实现任何特性的时刻可回来对照。
- **Appendix II: Generated Syntax Tree Classes** — jlox 自动生成的 AST 类的全列表。

---

## 心得提炼（个人总结）

1. **jlox vs clox 的递进是本书真正的主线**：同一个特性（变量、函数、类、闭包、继承、GC）你几乎都会实现两次，对比才会凸显"编译器其实只是另一张数据结构"的本质。
2. **单遍编译 vs 多遍编译**：jlox 需要单独的 Resolver 一遍遍历；clox 把解析、编译、语义分析揉在一遍里完成（省内存、省时间，但失误难修复）。这是"Java 的静态作用域 + Lox 声明前不能引用"共同促成的设计，值得反复咀嚼。
3. **一切高效来自于"提前做了能提前做的工作"**：Resolver 的距离优化、编译期的槽定位、字符串驻留、类方法表的提前解析 —— 都是把运行时查找挪到编译期。
4. **GC 是最难的一章**：C 手写 mark-sweep 需要严格管理 roots，任何 C 栈里遗留的 Value 引用都可能成为丢对象 bug 的来源。这章的调试经验极其宝贵。
5. **Pratt 解析值得背下来**：`{ prefix, infix, precedence }` 的三元组 + 递归循环 —— 15 行核心代码解决运算符优先级，适用于几乎所有表达式的语法。
6. 解释器的本质 = 数据结构 + 遍历；数组和哈希表（VM、GC）决定了速度的上限。

---

## 参考链接

- 在线全书：<https://craftinginterpreters.com/contents.html>
- 源码仓库：`./craftinginterpreters/`（submodule）
- 中文翻译：<https://github.com/GuoYaxiang/craftinginterpreters_zh>
- 作者博客：<http://journal.stuffwithstuff.com/>
