# Part II. A Tree-Walk Interpreter（jlox，Java）

> 第 4–13 章。jlox 风格：教学清晰、树遍历求值。

## 第 4 章 Scanning

- Token 的结构：关键字、标识符、数字、字符串、双字符运算符（`!=`、`<=`）等。
- 手写 Scanner：一次文本 → token 列表，用 `switch-case` 完成 DFA 的状态转换。
- 关键点：字符串和数字字面量的解析；行尾追踪（用于报错信息）。
- 错误处理：Error 类只负责收集（line + message），区分 `ScanError`/`ParseError`。

**Design Note: Implicit Semicolons** —— 讨论省略分号还是让 newline 代替（像 JavaScript 的 ASI 的坑），Lox 保留分号但允许染色（染色使得省略换行作为分隔）。

## 第 5 章 Representing Code

- 引入 AST：用类表示语法树（每个节点都是 Expr 的子类）。
- **元编程**：写一个 `GenerateAst.java` 程序自动生成 Expr 类（Binary、Grouping、Literal、Unary、Assign、Variable、Logical、Call、Get、Set、This、Super…），避免手动重复代码。
- **Visitor 模式**：为每种节点添加访问逻辑，避免在每个类上写 `accept`。

**Design Note: Logic Versus History** —— 语言 `a + b = c` 的合法性讨论：语法设计受历史 (FORTRAN) 而非逻辑影响。

## 第 6 章 Parsing Expressions

- **递归下降解析**（Recursive Descent）：每个语法产生式对应一个函数；一个个 token 向前推进，失败时通过提前匹配 token 类型做**分支预测**。
- 关键技能：把优先级用**表达式语法分层**（assignment → expression → equality → comparison → term → factor → unary → primary）。
- 错误恢复：**panic mode + 同步**。在匹配错时跳过到下一条语句的开始 token（`;` 等），继续恢复。
- Pratt parsing（本章引入但实际在 clox 中使用更深入）。

## 第 7 章 Evaluating Expressions

- `Interpreter` 实现 Visitor，用**一个方法一个 case** 来 eval 表达式树。
- **truthiness**：Lox 中 `false` 和 `nil` 是假，其他一切（包括 `0`、`""`）都是真。
- 二元运算的**操作数类型检查**：这一步在运行时才发生（动态类型），错误处理用 `RuntimeError` 异常，在 REPL 外层捕获。
- `==` 的定义：Lox 中不同类型永不相等（`"1" != 1`）。

**Design Note: Static and Dynamic Typing** —— 静态类型 vs 动态类型的哲学讨论：安全性 vs 灵活性，"类型是最好的文档"。

## 第 8 章 Statements and State

- 增加**语句**：`expression statement`、`print statement`、声明。
- **Environment**：一个 `HashMap<String, Object>`，负责变量名到值的绑定；**作用域链**由嵌套 Environment 形成（`enclosing` 指针）。
- 赋值即"在环境里找到该名字并替换值"，然后返回右值。
- `ResolveError`/`RuntimeError` 的分工。
- REPL：解析到 EOF 就退出；支持" expressions + statements 混合"，没有分号的表达式也直接求值打印（结果用表达式语句兼 REPL 的糖）。

**Design Note: Implicit Variable Declaration** —— 赋值一个未声明变量是定义还是错误？Lox 选择报错（严格），避免打错变量名产生"幽灵变量"。

## 第 9 章 Control Flow

- 添加 `if/else`、逻辑运算符 `and/or`（**短路求值**：它们是语法糖级的"控制流表达式"）、`while`、`for`。
- `for` 被 desugar 成 `while` 的 Idea：语法糖让使用者写起来更舒服，但解释器内部其实没有 `for` 的独立表示。

**Design Note: Spoonfuls of Syntactic Sugar** —— 语言加糖能增加表达力但增加学习成本和实现复杂度。Lox 的原则：糖必须**可以被 desugar 成已有语法**（`for` desugar 成 `while`），这样就不需要两条执行路径。

## 第 10 章 Functions

- 函数声明、调用表达式 `callee(args)`。
- LoxFunction 自己持有 `closure`（环境）。
- 调用流程：call → 新环境（把参数绑定）→ 执行 body → `return` 值（没有 return 就是 `nil`）。
- Native functions（`clock()`）：用 Java 接口模拟宿主函数，区别于用户函数。

## 第 11 章 Resolving and Binding

- **语义分析 / Resolver** 第二遍遍历：绑定每个变量**使用**到它的**声明**位置。
- 核心数据：给每个 Variable/Assign 节点标注一个**深度数**，运行时不用沿环境链逐层查找，直接按数 hopping。
- 作用域：`beginScope()`/`endScope()` 镜像块的嵌套。
- Lox 的静态作用域规则 detail：**在声明前引用**（如 `var a = a;`）是不合法的 —— 解析为错误，虽然 JavaScript 里它会"成功"读到 `undefined`。
- `this` 也用同样机制：`this` 也是"把 `this` 绑定为一个特殊局部"。

## 第 12 章 Classes

- `class Name { method(); }`。
- 类 = 一组方法 + `init` 构造器；类不是值（不能赋值给变量？可以赋值，但 class 表达式只能用于声明…… 实际上 jlox 里 class 是 statement）。
- **方法绑定**：把 receiver 绑定为 `this` 并存进闭包 —— 这是"方法即闭包"的统一（函数式 + 面向对象互相解释）。
- 字段是实例上的属性，读写用 `Get`/`Set` 表达式。

**Design Note: Prototypes and Power** —— 基于类的语言 vs 基于原型（prototype-based，如 JavaScript）的语言：后者更强表达能力但风险更大，大部分程序员在类的模型里更轻松。

## 第 13 章 Inheritance

- `<` 进行继承：`class A < B { ... }` —— 把超类的方法复制进子类的方法表。
- `super.method()`：静态解析为超类方法表里的查找。
- 重要细节：**重写方法时要重新绑定 receiver**，且 super 的绑定要求方法所在对象仍然持有 `this`。
