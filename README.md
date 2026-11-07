# c_vm_craftinginterpreters

《Crafting Interpreters》（Robert Nystrom 著）学习仓库。

![Build Crafting Interpreters](https://github.com/edidada/c_vm_craftinginterpreters/actions/workflows/craftinginterpreters.yml/badge.svg)

- 📝 [notes/](./notes/README.md) — 全书读书笔记（中文），覆盖三大部（Welcome / jlox 树遍历解释器 / clox 字节码虚拟机）及全部 Design Notes。
- 📚 [craftinginterpreters/](./craftinginterpreters/) — 官方源码仓库，以 git submodule 引入（<https://github.com/munificent/craftinginterpreters>）。
- ▶️ [examples/](./examples/) — Lox demo 脚本与自建测试，供本地和 CI 编译验证使用。

## CI：编译并运行 submodule 代码

[GitHub Actions 工作流](./.github/workflows/craftinginterpreters.yml)在每次 push 时：

| Job | 编译 | 验证 |
|---|---|---|
| **clox** | `gcc` 编译 `craftinginterpreters/c/*.c` | 运行 demo + 3 个功能测试，diff 期望输出 |
| **jlox** | `javac` 编译 `craftinginterpreters/java` 的最终版源码 | 同上 |

## 本地运行（可选）

```bash
gcc -O2 -o build/clox craftinginterpreters/c/*.c
./build/clox examples/demo.lox

javac -d build/jlox craftinginterpreters/java/com/craftinginterpreters/lox/*.java
java -cp build/jlox com.craftinginterpreters.lox.Lox examples/demo.lox
```

## 获取 submodule

```bash
git clone --recurse-submodules https://github.com/edidada/c_vm_craftinginterpreters
# 或者已有的 clone 中：
git submodule update --init
```

## 在线阅读

<https://craftinginterpreters.com/contents.html>
