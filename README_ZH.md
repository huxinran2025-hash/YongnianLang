# YONG — 一门语言，两个世界

> **声明意图，编译器物化。**

YONG（永年语言，YongnianLang）是一门声明式编程语言，用同一套语法同时编译到 **Web 应用** 和 **神经形态硬件**。

```
     30 行 YONG                          编译器生成的代码
┌──────────────────────┐        ┌──────────────────────────────────┐
│ @api(POST, "/todos") │   →    │ Express.js 路由 + 中间件         │
│ fn add_todo(...)     │        │ + 校验 + 错误处理 + 数据库 ORM   │
├──────────────────────┤        ├──────────────────────────────────┤
│ network MNIST {      │   →    │ 900+ 行 SystemVerilog RTL       │
│   layer input(784)   │        │ + LIF 神经元 + STDP 学习        │
│   connect -> output  │        │ + WTA 竞争 + AXI 总线           │
│ }                    │        │ + 测试平台                       │
└──────────────────────┘        └──────────────────────────────────┘
```

## 为什么需要 YONG？

| 痛点 | YONG 的解法 |
|------|------------|
| Web 应用需要 React + Node + SQL | **App 方言**: 一个 `.yong` 文件 → 全栈应用 |
| SNN 硬件需要 Verilog + CUDA | **Hardware 方言**: 一个 `.yong` 文件 → 可综合 RTL |
| 物理单位导致隐性 bug | **单位安全**: `Time<ms> + Energy<pJ>` → 编译错误 |
| 认证/安全总是后补 | **声明式 RBAC**: `@requires(Policy.delete)` → 自动注入 |
| 一个简单 CRUD 要 10 个文件 | **30 行**: 数据 + API + UI 写在一个文件里 |

## 快速一览

### App 方言 — 30 行全栈应用

```yong
@db(table="todos")
struct Todo {
    id: uid
    text: string
    done: bool
}

@api(GET, "/todos")
fn list_todos() -> list[Todo] {
    return db.todos.all()
}

@api(POST, "/todos")
fn add_todo(text: string) -> Todo {
    return db.todos.create({ text, done: false })
}

@app(route="/")
component TodoApp {
    state todos = query(list_todos)
    view {
        Header("My Todos")
        Input(placeholder="Add...", on_enter=add_todo)
        List(todos) -> |todo| {
            Row { Checkbox(todo.done) Text(todo.text) }
        }
    }
}
```

**编译产物**: HTML + CSS + JS + REST API + 数据库建表 + ORM

### Hardware 方言 — 15 行 SNN

```yong
network MNIST {
    layer input(784);
    layer hidden(400, type=lif, threshold=300, leak=1);
    layer output(10, type=lif, threshold=200, leak=2);

    connect input -> hidden with stdp(ltp=5, ltd=2);
    connect hidden -> output with stdp(ltp=10, ltd=4);

    config {
        weight_storage: bram
        wta_mode: enable
        homeostasis: enable
        threshold_mode: dual
    }
}
```

**编译产物**: SystemVerilog RTL（可综合，用于 FPGA/ASIC）或 Python SNN 仿真器

## 核心原则 (不变的)

以下原则永远不会改变：

| 原则 | 说明 |
|------|------|
| **声明式** | 写「做什么」，不写「怎么做」 |
| **数据优先** | 先定义 struct，再定义行为 |
| **单位安全** | 物理量自带单位；不兼容的单位 = 编译错误 |
| **AI 友好** | 最少 token 表达最复杂的系统 |
| **bit-accurate** | 同一份 `.yong` → 在每个后端行为一致 |
| **可扩展** | 核心不变，其余随意扩展 |

## 可扩展性

YONG 被设计为可生长的语言。核心语法冻结，但一切皆可扩展：

| 扩展什么 | 怎么扩展 | 示例 |
|---------|---------|------|
| 新装饰器 | 定义 `@name` | `@cache(ttl=60s)`, `@rate_limit(100/min)` |
| 新类型 | 注册到 TypeRegistry | `image`, `tensor`, `Energy<eV>` |
| 新神经元类型 | 注册 NeuronTypeDef | `type=izhikevich`, `type=adex` |
| 新学习规则 | 注册 LearningRuleDef | `with bcm`, `with oja` |
| 新后端 | 实现 Backend 接口 | RISC-V RTL, WebAssembly, 量子电路 |
| 新分析 Pass | 注册 AnalyzerPass | PerformancePass, StylePass |

扩展三原则：
1. **只声明，不实现** — 扩展必须是声明式的
2. **不能破坏现有代码** — 扩展只能是加法
3. **有单位的量必须带单位** — 新的物理量类型必须标注维度

## 文档

| 文档 | English | 中文 |
|------|---------|------|
| 语言规范 | [spec/language-spec.md](spec/language-spec.md) | [spec-zh/YONG语言规范.md](spec-zh/YONG语言规范.md) |
| 编译器规范 | [spec/compiler-spec.md](spec/compiler-spec.md) | [spec-zh/YONG编译器规范.md](spec-zh/YONG编译器规范.md) |

## 编译器架构

```
 .yong 源码
     │
     ▼
┌──────────┐
│ Stage 1  │  Tokenizer → Token 流
├──────────┤
│ Stage 2  │  Parser → AST
├──────────┤
│ Stage 3  │  Analyzer → Typed AST（所有错误在此报完）
├──────────┤
│ Stage 4  │  IR Generator → YONG IR（所有后端共享）
├──────────┤
│ Stage 5  │  Backend → 目标代码（可插拔）
└──────────┘
     │
     ├──→  Web App (HTML/CSS/JS + API)
     ├──→  FPGA RTL (SystemVerilog)
     ├──→  ASIC RTL (Verilog)
     ├──→  SNN Simulator (Python)
     └──→  你自己的后端
```

## 项目状态

| 组件 | 状态 |
|------|------|
| 语言规范 v4.2 | ✅ 完成 |
| 编译器规范 v1.0 | ✅ 完成 |
| EBNF 正式文法 | ✅ 完成 |
| 类型系统 + 单位安全 | ✅ 已定义 |


## 代码示例

查看 [examples/](examples/) 目录：

- [`hello.yong`](examples/hello.yong) — Hello World
- [`todo-app.yong`](examples/todo-app.yong) — 全栈 Todo App（30 行）
- [`snn-mnist.yong`](examples/snn-mnist.yong) — MNIST 分类器 SNN
- [`auth-api.yong`](examples/auth-api.yong) — 带 RBAC 的认证 API

## 参与贡献

YONG 处于早期阶段。我们欢迎以下方向的贡献：

- 🧪 **编译器实现** — 帮助构建 5 级管线
- 📝 **语言设计反馈** — 就规范提 Issue
- 🔌 **后端插件** — 实现新的编译目标
- 📚 **文档** — 改进规范、添加示例、翻译

## 作者

**Robert Hu (胡翔瑞)** — 中国·重庆 🇨🇳  
📧 [roberthxr@qq.com](mailto:roberthxr@qq.com)

## 许可证

[Apache-2.0](LICENSE)

---

*"声明意图，编译器物化。"*  
*"Declare intent, compiler materializes."*
