# YONG — One Language, Two Worlds

> **Declare intent. Compiler materializes.**

YONG (永年语言, YongnianLang) is a declarative programming language that compiles to **both web applications and neuromorphic hardware** from the same syntax.

```
     30 lines of YONG                    What the compiler generates
┌──────────────────────┐        ┌──────────────────────────────────┐
│ @api(POST, "/todos") │   →    │ Express.js route + middleware    │
│ fn add_todo(...)     │        │ + validation + error handling    │
│                      │        │ + database ORM                   │
├──────────────────────┤        ├──────────────────────────────────┤
│ network MNIST {      │   →    │ 900+ lines of SystemVerilog RTL  │
│   layer input(784)   │        │ + LIF neurons + STDP learning    │
│   connect -> output  │        │ + WTA competition + AXI bus      │
│ }                    │        │ + testbench                      │
└──────────────────────┘        └──────────────────────────────────┘
```

## Why YONG?

| Problem | YONG's Answer |
|---------|--------------|
| Web apps require React + Node + SQL | **App Dialect**: One `.yong` file → full-stack app |
| SNN hardware requires Verilog + CUDA | **Hardware Dialect**: One `.yong` file → synthesizable RTL |
| Physical units cause silent bugs | **Unit Safety**: `Time<ms> + Energy<pJ>` → compile error |
| Auth/security is an afterthought | **Declarative RBAC**: `@requires(Policy.delete)` → auto-injected |
| 10 files for a simple CRUD | **30 Lines**: Data + API + UI in one file |

## Quick Look

### App Dialect — Full-Stack in 30 Lines

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

### Hardware Dialect — SNN in 15 Lines

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

**Compiles to**: SystemVerilog RTL (synthesizable for FPGA/ASIC) or Python SNN simulator

## Core Principles (Frozen)

These will never change:

| Principle | Description |
|-----------|-------------|
| **Declarative** | Write *what*, not *how* |
| **Data-First** | Define structs before behavior |
| **Unit Safety** | Physical quantities carry units; incompatible units = compile error |
| **AI-Friendly** | Minimum tokens to express maximum complexity |
| **Bit-Accurate** | Same `.yong` → same behavior on every backend |
| **Extensible** | Frozen core, open everything else |

## Extensibility

YONG is designed to grow. The core syntax is frozen, but everything else is open:

| What | How | Example |
|------|-----|---------|
| New decorators | Define `@name` | `@cache(ttl=60s)`, `@rate_limit(100/min)` |
| New types | Register in TypeRegistry | `image`, `tensor`, `Energy<eV>` |
| New neuron types | Register NeuronTypeDef | `type=izhikevich`, `type=adex` |
| New learning rules | Register LearningRuleDef | `with bcm`, `with oja` |
| New backends | Implement Backend interface | RISC-V RTL, WebAssembly, Quantum |
| New analyzer passes | Register AnalyzerPass | PerformancePass, StylePass |

Three rules for extending:
1. **Declare, never implement** — Extensions must be declarative
2. **Never break existing code** — Extensions are additive only
3. **Units must have units** — New physical types must carry dimensions

## Documentation

| Document | English | 中文 |
|----------|---------|------|
| Language Specification | [spec/language-spec.md](spec/language-spec.md) | [spec-zh/YONG语言规范.md](spec-zh/YONG语言规范.md) |
| Compiler Specification | [spec/compiler-spec.md](spec/compiler-spec.md) | [spec-zh/YONG编译器规范.md](spec-zh/YONG编译器规范.md) |

## Compiler Architecture

```
 .yong source
     │
     ▼
┌──────────┐
│ Stage 1  │  Tokenizer → Token stream
├──────────┤
│ Stage 2  │  Parser → AST
├──────────┤
│ Stage 3  │  Analyzer → Typed AST (all errors caught here)
├──────────┤
│ Stage 4  │  IR Generator → YONG IR (shared across backends)
├──────────┤
│ Stage 5  │  Backend → Target code (pluggable)
└──────────┘
     │
     ├──→  Web App (HTML/CSS/JS + API)
     ├──→  FPGA RTL (SystemVerilog)
     ├──→  ASIC RTL (Verilog)
     ├──→  SNN Simulator (Python)
     └──→  Your own backend
```

## Project Status

| Component | Status |
|-----------|--------|
| Language Specification v4.2 | ✅ Complete |
| Compiler Specification v1.0 | ✅ Complete |
| EBNF Grammar | ✅ Complete |
| Type System + Unit Safety | ✅ Specified |


## Examples

See the [examples/](examples/) directory:

- [`hello.yong`](examples/hello.yong) — Hello World
- [`todo-app.yong`](examples/todo-app.yong) — Full-stack Todo App (30 lines)
- [`snn-mnist.yong`](examples/snn-mnist.yong) — MNIST classifier SNN
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

*"Declare intent, compiler materializes."*  
*"声明意图，编译器物化。"*
