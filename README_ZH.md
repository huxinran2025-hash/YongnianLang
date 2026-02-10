# YONG — CUDA 杀手

> **算力不等于电力。这是给 AI 写的语言，不是给人类的。**

YONG（永年语言）是一门声明式编程语言，为 **AI 生成代码**而设计，不是给人类学习的。同样的功能，AI 输出 30 个 Token 的 YONG，而不是 500 个 Token 的 Python。编译器负责把剩下的物化出来。

```
 AI 今天写的代码                          AI 应该写的代码
┌─────────────────────────────┐    ┌─────────────────────────────┐
│ from flask import Flask...  │    │ struct Todo {               │
│ from sqlalchemy import...   │    │   title: string             │
│ app = Flask(__name__)       │    │   done: bool                │
│ Base = declarative_base()   │    │ }                           │
│ class Todo(Base):           │    │ @api(POST, "/todos")        │
│   __tablename__ = 'todos'   │    │ fn create(req) -> Todo {    │
│   id = Column(Integer...)   │    │   return req |> save;       │
│   title = Column(String...) │    │ }                           │
│   done = Column(Boolean...) │    │                             │
│   ... 还有 40 行            │    │                             │
├─────────────────────────────┤    ├─────────────────────────────┤
│ ~500 tokens                 │    │ ~30 tokens                  │
│ 每 1000 个文件 ~$15         │    │ 每 1000 个文件 ~$1.50       │
└─────────────────────────────┘    └─────────────────────────────┘
```

**10 倍更少的 Token = 10 倍更少的 GPU 算力 = 10 倍更少的电力。**

## 为什么这是 CUDA 杀手

CUDA 的护城河不是硬件——是生态锁定。但有一个更大的浪费藏在明处：

**AI 写代码用的语言本身就是一种算力消耗。**

AI 生成的每一行 `import`、每一段 ORM 配置、每一个路由注册，都是在烧 GPU 的 Token。全球所有 AI 编码助手，每天数十亿 Token。

YONG 消灭这种浪费：

| | Python（AI 今天写的） | YONG（AI 应该写的） |
|---|---|---|
| Todo 应用 | ~500 tokens | **~30 tokens** |
| 用户认证 API | ~800 tokens | **~50 tokens** |
| SNN 芯片设计 | ~900 行 Verilog | **~30 行 YONG** |
| AI 生成成本 | 1× | **0.1×** |

而且还有第二杀：YONG 还能编译到 **SNN 神经形态硬件**，每次脉冲只要 28 pJ——比 GPU 推理高效 **100,000 倍**。

**一门语言，两次击杀。生成成本降 10 倍，执行成本降 100,000 倍。**

## 工作原理

**人类不写 YONG。人类不学 YONG。**

```
              传统流程:
用户 → "做个Todo应用" → AI → 500 tokens Python → 解释器 → 应用

              YONG 流程:
用户 → "做个Todo应用" → AI → 30 tokens YONG → 编译器 → 应用/芯片
```

编译器从声明推断一切：数据库建表、HTTP 路由、中间件、序列化、错误处理——或者 LIF 神经元、STDP 学习、WTA 竞争、BRAM 流水线。

**声明意图，编译器物化。**

## 架构

```
                          ┌─────────────────────────────────────┐
                          │         人类（自然语言描述）          │
                          └──────────────┬──────────────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────────────┐
                          │   VGO Brain 2.0（1.09亿参数 SNN）    │
                          │   自然语言 → YONG 代码               │
                          └──────────────┬──────────────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────────────┐
                          │         .yong 源文件                 │
                          │    （30 tokens，声明式意图）          │
                          └──────────────┬──────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
             ┌────────────┐     ┌──────────────┐     ┌──────────────┐
             │   词法分析   │     │   语法分析    │     │   IR 生成    │
             │  (Lexer)    │ ──▶ │   (AST)      │ ──▶ │ (类型化 IR)  │
             └────────────┘     └──────────────┘     └──────┬───────┘
                                                            │
                                         ┌──────────────────┼──────────────────┐
                                         ▼                                     ▼
                              ┌─────────────────────┐             ┌─────────────────────┐
                              │    Web 后端          │             │    RTL 后端          │
                              │  HTML+CSS+JS+API+DB │             │  Verilog → Yosys    │
                              │  （App 方言）        │             │  → FPGA/ASIC        │
                              └─────────────────────┘             │  （Hardware 方言）   │
                                                                  └─────────────────────┘
```

**5 级管线**: 词法分析 → 语法分析 → AST → 类型化 IR → 后端代码生成。
同一份 `.yong` 文件可以编译为 Web 应用**或者**可综合硅片。

## 快速一览

### App 方言 — 30 个 Token 搞定全栈

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

### Hardware 方言 — 15 行 SNN 芯片

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
        target: zynq7020
    }
}
```

**编译产物**: 47KB 可综合 Verilog RTL（Yosys + iCE40 FPGA 验证通过）

## 为什么 AI 喜欢 YONG

| 特性 | 对 AI 生成的意义 |
|------|-----------------|
| **没有 import** | 编译器从声明推断依赖。零浪费 token。 |
| **装饰器即指令** | `@api(POST, "/todos")` 替代 20 行路由+中间件+序列化 |
| **管道操作符** | `data |> validate |> save |> emit` — 线性流动，无嵌套。AI 一次生成。 |
| **没有模板代码** | 没有 ORM 配置，没有 app factory，没有 session。声明即推断。 |
| **双编译目标** | 同一语法编译到 Web 或芯片。AI 不需要知道目标是什么。 |

## 核心原则（不变的）

| 原则 | 说明 |
|------|------|
| **AI 优先** | 为 AI 生成而设计，不是给人类打字用的 |
| **声明式** | 写「做什么」，不写「怎么做」 |
| **最少 Token** | 用最小表面积承载最大语义 |
| **单位安全** | `Time<ms> + Energy<pJ>` → 编译错误 E202 |
| **bit-accurate** | 同一份 `.yong` → 在每个后端行为一致 |
| **可扩展** | 核心不变，其余随意扩展 |

## 数据对比

| 指标 | 英伟达 GPU | VGO SNN（YONG 编译）|
|------|-----------|-------------------|
| 每次推理功耗 | ~2,700,000 pJ | **~28 pJ/spike** |
| 能效比 | 1× | **~100,000×** |
| 代码行数 | 900+（CUDA/Verilog）| **30（YONG）** |
| FPGA 比特流 | N/A | **132 KB** |

> 📊 **完整测量方法、硬件规格和复现脚本**: [benchmarks/README.md](benchmarks/README.md)

## AI 同行评审（2026年2月）

我们把 YONG 代码发给了 5 个主流 AI 模型——**它们之前从未见过 YONG**。

**第一轮** — 纯代码，零上下文：**5/5 立即理解。**

**第二轮** — 完整 README 审阅：

| AI | 核心评价 |
|---|---|
| **Claude** | *"Token 经济学论点非常准确。你不是在争夺人类心智——你是在争夺 AI 的 Token 预算。完全不同的赛道。"* |
| **DeepSeek** | *"出色的、必要的、高风险/高回报的思想实验变成了现实。"* |
| **Gemini** | *"YONG 看起来像一个完美的中间表示（IR）——人类描述，AI 生成 YONG，然后编译到 Rust、Go 或 TypeScript。"* |
| **ChatGPT 5.2** | *"README 不是画饼——有具体的状态声明。这是正确的可信度信号。"* |
| **Grok** | *"瓶颈不再是晶体管或瓦特——而是 Token 和不必要的语法。"* |

这些 AI 就是 YONG 的目标用户。它们的即时理解验证了核心设计论点。

## 项目状态

| 组件 | 状态 |
|------|------|
| 语言规范 v4.3 | ✅ 完成（v4.2 核心 + v4.3 补充）|
| 编译器规范 v1.0 | ✅ 完成 |
| 解析器（Lexer → AST → IR）| ✅ 运行中（822 行）|
| 原生引擎（.yong → GUI）| ✅ 运行中 |
| Verilog 后端（.yong → RTL）| ✅ 运行中（47KB 输出，Yosys 验证）|
| VGO Brain 2.0（自然语言 → YONG）| ✅ 运行中（109M 参数 SNN）|
| FPGA 综合（iCE40）| ✅ 验证通过（132KB 比特流）|
| VSCode 扩展 | ✅ 语法高亮 |
| 微调数据集 | 🔄 16 个种子对（目标: 1000+）|
| 一致性测试 | 🔄 31 个测试用例 |

## 文档

| 文档 | English | 中文 |
|------|---------|------|
| 语言规范 | [spec/language-spec.md](spec/language-spec.md) | [spec-zh/YONG语言规范.md](spec-zh/YONG语言规范.md) |
| 规范补充 v4.3 | [spec/language-spec-addendum-v4.3.md](spec/language-spec-addendum-v4.3.md) | [spec-zh/YONG语言规范补充v4.3.md](spec-zh/YONG语言规范补充v4.3.md) |
| 编译器规范 | [spec/compiler-spec.md](spec/compiler-spec.md) | [spec-zh/YONG编译器规范.md](spec-zh/YONG编译器规范.md) |
| 基准测试方法论 | [benchmarks/README.md](benchmarks/README.md) | — |

## 代码示例

查看 [examples/](examples/) 目录：

- [`hello.yong`](examples/hello.yong) — Hello World
- [`todo-app.yong`](examples/todo-app.yong) — 全栈 Todo App（30 tokens）
- [`snn-mnist.yong`](examples/snn-mnist.yong) — MNIST 分类器 SNN 芯片
- [`auth-api.yong`](examples/auth-api.yong) — 带 RBAC 的认证 API
- [`error-handling.yong`](examples/error-handling.yong) — 铁路导向错误处理
- [`blog-engine.yong`](examples/blog-engine.yong) — 完整 CMS（文章、评论、标签、认证、搜索）
- [`ecommerce-api.yong`](examples/ecommerce-api.yong) — 电商 API（状态机、事务、Webhook）
- [`snn-speech.yong`](examples/snn-speech.yong) — SNN 语音识别（储备池计算）

## 工具链

- **VSCode 扩展**: [vscode-yong/](vscode-yong/) — `.yong` 文件语法高亮
- **微调数据集**: [datasets/](datasets/) — 自然语言→YONG 训练对
- **一致性测试**: [tests/](tests/) — 31 个金标测试用例

## 参与贡献

YONG 处于早期阶段。我们欢迎以下方向的贡献：

- 🧪 **编译器实现** — 帮助构建 5 级管线
- 📝 **语言设计反馈** — 就规范提 Issue
- 🔌 **后端插件** — 实现新的编译目标
- 📚 **文档** — 改进规范、添加示例、翻译

## 作者

**Robert Hu** — 中国·重庆 🇨🇳  
📧 [roberthxr@qq.com](mailto:roberthxr@qq.com)

## 许可证

[Apache-2.0](LICENSE)

---

*声明意图，编译器物化。* | *Declare intent. Compiler materializes.*  
*算力不等于电力。* | *Compute ≠ Power.*  
*写代码的是 AI，不是人。* | *The writer is AI, not human.*
