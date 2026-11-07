# Part I. Welcome（欢迎）

> 第 1–3 章。

## 第 1 章 Introduction

- 为什么学解释器：**解释器是"图灵柱"** —— 能在单一程序里压缩大量计算逻辑。
- 三个理由：处理其他语言（SQL、日志 DSL 等）、提升对 C#/Java 等语言的理解、以及纯粹的智力快感。
- 本书的语言 Lox 是动态类型、GC、有类的脚本语言。
- 教学顺序：从孤儿语言特性到完整实现，先功后名。

**Design Note: What's in a Name?** —— 给语言命名是个 Marketing 问题，名字会随时间改变对语言的感觉（Lox 一词无所指也没关系，像 Java 也只是为了好玩）。

## 第 2 章 A Map of the Territory

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

## 第 3 章 The Lox Language

Lox 语言特性（相当于"产品需求文档"）：

- 动态类型、原生支持 GC、支持函数式（一等函数）+ 面向对象。
- 语法上极简：C 风格注释、分号结尾、块、if/while/for、函数、类（单继承）、`return`。
- 只有三种副作用允许变量绑定：声明、赋值、变量绑定到函数参数。
- 一个类的最小序列：`data + behavior`（字段 + 方法）。
- 语义上特别注意：变量作用域是**块级**（C 风格），但**没有变量提升**。
- `print` 是内建语句而不是函数（简化设计）。

**Design Note: Implicit Variable Declaration** —— 为什么 Lox 需要显式 `var`。省略它靠约定（如 Python 的赋值即声明），但语义会产生全局作用域陷阱，Lox 选择更严格。
