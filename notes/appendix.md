# Backmatter（附录）

## Appendix I: Lox Grammar

一页纸的语言完整语法（BNF 风格），实现任何特性的时刻可回来对照。要点记忆：

- 语句级：`print`、声明、块、`if`、`while`、`for`、`return`、表达式语句。
- 表达式优先级（低 → 高）：
  `comma`（clox 未实现）→ assignment → or → and → equality → comparison → term → factor → unary → call/property → primary。
- 类：`class Name < Super ? { methods }`。

## Appendix II: Generated Syntax Tree Classes

jlox 自动生成的 AST 类全列表（由 `tool/GenerateAst` 生成）：

- **Expr**：Assign(=)、Binary(=)、Call(=)、Get(=)、Grouping(=)、Literal(=)、Logical(=)、Set(=)、Super(=)、This(=)、Unary(=)、Variable(=)。
- **Stmt**：Block、Class、Expression、Function、If、Print、Return、Var、While。
