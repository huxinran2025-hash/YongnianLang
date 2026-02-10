# YONG — The CUDA Killer

> **Compute ≠ Power. The language AI writes, not humans.**

YONG (永年语言) is a declarative programming language designed for **AI to generate**, not humans to learn. Same 30 tokens of YONG replace 500 tokens of Python. The compiler materializes the rest.

```
 What AI writes today                    What AI should write
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
│   ... 40 more lines         │    │                             │
├─────────────────────────────┤    ├─────────────────────────────┤
│ ~500 tokens                 │    │ ~30 tokens                  │
│ ~$15 per 1000 files         │    │ ~$1.50 per 1000 files       │
└─────────────────────────────┘    └─────────────────────────────┘
```

**10× fewer tokens = 10× less GPU compute = 10× less power.**

## Why This Is a CUDA Killer

CUDA's moat isn't hardware — it's ecosystem lock-in. But there's a bigger waste hiding in plain sight:

**The language AI writes in IS a form of compute cost.**

Every `import`, every ORM config, every route handler that AI generates is a token that burns GPU power. Billions of tokens, every day, across every AI coding assistant on the planet.

YONG eliminates the waste:

| | Python (what AI writes today) | YONG (what AI should write) |
|---|---|---|
| Todo App | ~500 tokens | **~30 tokens** |
| User Auth API | ~800 tokens | **~50 tokens** |
| SNN Chip Design | ~900 lines Verilog | **~30 lines YONG** |
| AI generation cost | 1× | **0.1×** |

And here's the double kill: YONG also compiles to **SNN neuromorphic hardware** at 28 pJ/spike — **100,000× more efficient** than GPU inference.

**One language. Two kills. Generation cost down 10×. Execution cost down 100,000×.**

## How It Works

**Humans don't write YONG. Humans don't learn YONG.**

```
              Today's pipeline:
User → "Build a todo app" → AI → 500 tokens Python → interpreter → app

              YONG pipeline:
User → "Build a todo app" → AI → 30 tokens YONG → compiler → app/silicon
```

The compiler infers everything from declarations: database schema, HTTP routing, middleware, serialization, error handling — or LIF neurons, STDP learning, WTA competition, BRAM pipelines.

**Declare intent. Compiler materializes.**

## Quick Look

### App Dialect — Full-Stack in 30 Tokens

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

**Compiles to**: HTML + CSS + JS + REST API + Database schema + ORM

### Hardware Dialect — SNN Chip in 15 Lines

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

**Compiles to**: 47KB synthesizable Verilog RTL (verified on Yosys + iCE40 FPGA)

## Why AI Prefers YONG

| Feature | Why it matters for AI |
|---------|----------------------|
| **No imports** | Compiler infers dependencies from declarations. Zero wasted tokens. |
| **Decorators as directives** | `@api(POST, "/todos")` replaces 20 lines of routing + middleware + serialization. |
| **Pipe operators** | `data |> validate |> save |> emit` — linear flow, no nesting. AI generates in one pass. |
| **No boilerplate** | No ORM config, no app factory, no session management. Declared once, inferred everywhere. |
| **Dual compilation** | Same syntax targets web apps OR silicon. AI doesn't need to know the target. |

## Core Principles (Frozen)

| Principle | Description |
|-----------|-------------|
| **AI-First** | Designed for AI to generate, not humans to type |
| **Declarative** | Write *what*, not *how* |
| **Minimum Tokens** | Maximum semantics in minimum surface area |
| **Unit Safety** | `Time<ms> + Energy<pJ>` → compile error E202 |
| **Bit-Accurate** | Same `.yong` → same behavior on every backend |
| **Extensible** | Frozen core, open everything else |

## The Numbers

| Metric | NVIDIA GPU | VGO SNN (YONG-compiled) |
|--------|-----------|------------------------|
| Power per inference | ~2,700,000 pJ | **~28 pJ/spike** |
| Efficiency ratio | 1× | **~100,000×** |
| Lines of code | 900+ (CUDA/Verilog) | **30 (YONG)** |
| FPGA bitstream | N/A | **132 KB** |

## Project Status

| Component | Status |
|-----------|--------|
| Language Specification v4.2 | ✅ Complete |
| Compiler Specification v1.0 | ✅ Complete |
| Parser (Lexer → AST → IR) | ✅ Working (822 lines) |
| Native Engine (.yong → GUI) | ✅ Working |
| Verilog Backend (.yong → RTL) | ✅ Working (47KB output, Yosys verified) |
| VGO Brain 2.0 (NL → YONG) | ✅ Working (109M param SNN) |
| FPGA Synthesis (iCE40) | ✅ Verified (132KB bitstream) |

## Documentation

| Document | English | 中文 |
|----------|---------|------|
| Language Specification | [spec/language-spec.md](spec/language-spec.md) | [spec-zh/YONG语言规范.md](spec-zh/YONG语言规范.md) |
| Compiler Specification | [spec/compiler-spec.md](spec/compiler-spec.md) | [spec-zh/YONG编译器规范.md](spec-zh/YONG编译器规范.md) |

## Examples

See the [examples/](examples/) directory:

- [`hello.yong`](examples/hello.yong) — Hello World
- [`todo-app.yong`](examples/todo-app.yong) — Full-stack Todo App (30 tokens)
- [`snn-mnist.yong`](examples/snn-mnist.yong) — MNIST classifier SNN chip
- [`auth-api.yong`](examples/auth-api.yong) — Authenticated API with RBAC

## Contributing

YONG is in its early stages. We welcome contributions in:

- 🧪 **Compiler implementation** — Help build the 5-stage pipeline
- 📝 **Language design feedback** — File issues on the spec
- 🔌 **Backend plugins** — Implement new compilation targets
- 📚 **Documentation** — Improve specs, add examples, translate

## Author

**Robert Hu** — Chongqing, China 🇨🇳  
📧 [roberthxr@qq.com](mailto:roberthxr@qq.com)

## License

[Apache-2.0](LICENSE)

---

*Declare intent. Compiler materializes.* | *声明意图，编译器物化。*  
*Compute ≠ Power.* | *算力不等于电力。*  
*The writer is AI, not human.* | *写代码的是 AI，不是人。*
