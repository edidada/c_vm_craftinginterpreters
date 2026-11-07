# 总览与心得

> 《Crafting Interpreters》— Robert Nystrom 著，免费在线阅读： <https://craftinginterpreters.com/>
> 原始代码仓库：<https://github.com/munificent/craftinginterpreters>

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

## 心得提炼（个人总结）

1. **jlox vs clox 的递进是本书真正的主线**：同一个特性（变量、函数、类、闭包、继承、GC）你几乎都会实现两次，对比才会凸显"编译器其实只是另一张数据结构"的本质。
2. **单遍编译 vs 多遍编译**：jlox 需要单独的 Resolver 一遍遍历；clox 把解析、编译、语义分析揉在一遍里完成（省内存、省时间，但失误难以恢复）。这是"Java 的静态作用域 + Lox 声明前不能引用"共同促成的设计，值得反复咀嚼。
3. **一切高效来自于"提前做了能提前做的工作"**：Resolver 的距离优化、编译期的槽定位、字符串驻留、类方法表的提前解析 —— 都是把运行时查找挪到编译期。
4. **GC 是最难的一章**：C 手写 mark-sweep 需要严格管理 roots，任何 C 栈里遗留的 Value 引用都可能成为丢对象 bug 的来源。这章的调试经验极其宝贵。
5. **Pratt 解析值得背下来**：`{ prefix, infix, precedence }` 的三元组 + 递归循环 —— 15 行核心代码解决运算符优先级，适用于几乎所有表达式的语法。
6. 解释器的本质 = 数据结构 + 遍历；数组和哈希表（VM、GC）决定了速度的上限。

## 参考链接

- 在线全书：<https://craftinginterpreters.com/contents.html>
- 源码仓库：`./craftinginterpreters/`（submodule）
- 中文翻译：<https://github.com/GuoYaxiang/craftinginterpreters_zh>
- 作者博客：<http://journal.stuffwithstuff.com/>
