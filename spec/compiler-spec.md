# YONG Compiler Specification

> **Version**: v1.0  
> **Updated**: 2026-02-10  
> **Depends on**: [YongnianLang Language Specification v4.2](./language-spec.md)

---

## ⚠️ This is an Architecture Spec, Not a Concrete Implementation

This document defines the **skeleton and extension points** of the YONG compiler. It tells you what the compiler should look like, how data flows, and where to extend.  
It is **not** the source code documentation for any specific compiler implementation.

Like the language specification, the compiler is also extensible — the core pipeline is frozen, backends and plugins can be added freely.

---

## 1. Core Pipeline

```
 .yong source
     │
     ▼
┌──────────┐
│ Stage 1  │  Tokenizer — Source → Token stream
│  Lexer   │  Input: string    Output: List[Token]
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 2  │  Parser — Token stream → AST
│  Parser  │  Input: List[Token]    Output: AST (untyped)
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 3  │  Analyzer — AST → Typed AST
│ Analyzer │  Type inference + Unit checking + Security audit + Error reporting
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 4  │  IR Generator — Typed AST → YONG IR
│    IR    │  Unified intermediate representation (shared across backends)
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 5  │  Backend — YONG IR → Target code
│ Backend  │  Pluggable: one Backend plugin per target
└──────────┘
     │
     ├──→  Web App (HTML/CSS/JS + API Server)
     ├──→  FPGA RTL (SystemVerilog)
     ├──→  ASIC RTL (Verilog)
     ├──→  SNN Simulator (Python)
     ├──→  Mobile (React Native / Flutter)
     └──→  ... (your own backend)
```

### Frozen (Invariants)

| Invariant | Rationale |
|-----------|-----------|
| 5-stage pipeline structure | Each stage has clear responsibilities; stages communicate through standard data structures |
| Inter-stage data formats | Token / AST / Typed AST / IR interface definitions are immutable |
| Unidirectional data flow | Data flows only from Stage 1 to Stage 5, never backwards |
| All errors reported before Stage 3 completes | Backends perform no syntax or type checking |

### Extensible (Open)

| Extension Point | Method |
|-----------------|--------|
| New Backend | Register a `Backend` plugin |
| New decorators | Register a `DecoratorHandler` |
| New types | Register in `TypeRegistry` |
| New learning rules | Register in `LearningRuleRegistry` |
| New neuron types | Register in `NeuronTypeRegistry` |
| New lint / audit rules | Register in `AnalyzerPass` list |
| New IR optimizations | Register in `IRPass` list |

---

## 2. Stage 1: Tokenizer

### 2.1 Responsibilities

Split `.yong` source code strings into a Token sequence.

### 2.2 Token Structure

```
Token {
    type:    TokenType       // Classification (keyword/identifier/literal/symbol)
    value:   string          // Raw text
    line:    int             // Line number (1-indexed)
    column:  int             // Column number (1-indexed)
    span:    (start, end)    // Character offsets (for error indicators)
}
```

### 2.3 Token Classification

```
Keywords:
  App:      struct, enum, fn, component, state, view, policy,
            migrate, let, return, if, else, for, in, assert, emit,
            import, as, allow, deny, when, on, effect
  Hardware: network, layer, connect, config, with, enable, disable
  Shared:   true, false

Identifiers:
  /[a-zA-Z_][a-zA-Z0-9_]*/

Literals:
  NUMBER:       /[0-9]+(\.[0-9]+)?/
  STRING:       /"[^"]*"/
  HEX:          /0x[0-9a-fA-F_]+/
  UNIT_LITERAL: NUMBER + UNIT   (e.g.: 5ms, 10pJ, 100MHz)

Symbols:
  Structural: { } ( ) [ ]
  Operators:  + - * / % == != < > <= >= || && ! |> ? :
  Punctuation: ; , . -> = @

Comments:
  Line comment: // ...
  Block comment: /* ... */
```

### 2.4 Extension Point

**New keywords**: Simply add to the keyword table. The Tokenizer doesn't need to know keyword semantics.

---

## 3. Stage 2: Parser

### 3.1 Responsibilities

Parse the Token sequence into an AST (Abstract Syntax Tree). Grammar rules follow the [Language Specification §3 EBNF](./language-spec.md).

### 3.2 AST Node Types

```
AST (Abstract Syntax Tree)
├── Program
│   └── declarations: List[Declaration]
│
├── Declaration
│   ├── ImportDecl        { path, alias? }
│   ├── StructDecl        { name, annotations[], fields[] }
│   ├── EnumDecl          { name, variants[] }
│   ├── FnDecl            { name, annotations[], params[], return_type?, body }
│   ├── ComponentDecl     { name, annotations[], states[], views[], handlers[] }
│   ├── NetworkDecl       { name, layers[], connections[], config? }
│   ├── PolicyDecl        { name, resource, subject, rules[] }
│   └── MigrationDecl     { version, from, name, body }
│
├── Statement
│   ├── LetStmt           { name, type?, value }
│   ├── ReturnStmt        { value }
│   ├── IfStmt            { condition, then_block, else_block? }
│   ├── ForStmt           { variable, iterable, body }
│   ├── AssertStmt        { condition, message }
│   ├── EmitStmt          { event_name, args[] }
│   └── ExprStmt          { expr }
│
├── Expression
│   ├── LiteralExpr       { value, type }        // 42, "hello", true, 5ms
│   ├── IdentExpr         { name }               // x
│   ├── MemberExpr        { object, field }       // user.name
│   ├── CallExpr          { callee, args[] }      // foo(1, 2)
│   ├── BinaryExpr        { left, op, right }     // a + b
│   ├── UnaryExpr         { op, operand }         // !x
│   ├── TernaryExpr       { cond, then, else }    // a ? b : c
│   ├── PipeExpr          { left, fn_call }       // x |> sort()
│   └── ArrayExpr         { elements[] }          // [1, 2, 3]
│
├── Hardware-specific
│   ├── LayerNode         { name, size, neuron_type?, params{} }
│   ├── ConnectNode       { source, target, learning_rule, params{} }
│   └── ConfigNode        { entries{} }
│
├── Annotation
│   └── AnnotationNode    { name, args[] }
│
└── Type
    ├── SimpleType        { name }                // int, string, uid
    ├── GenericType       { name, type_args[] }   // list[Todo]
    ├── NullableType      { inner }               // string?
    └── UnitType          { base, unit }          // Time<ms>
```

### 3.3 Parser's Sole Responsibility

The Parser **only handles syntactic structure** and performs no semantic judgment:
- ✅ `@foo(bar=baz)` → Parse as `AnnotationNode("foo", [("bar", "baz")])`
- ✅ `connect A -> B with xyz` → Parse as `ConnectNode("A", "B", "xyz")`
- ❌ Does not check if `@foo` is a valid decorator
- ❌ Does not check if `xyz` is a valid learning rule
- ❌ Does not check if `A` and `B` are defined

These are all Stage 3 (Analyzer)'s responsibilities.

### 3.4 Error Recovery

When the Parser encounters a syntax error, it should:
1. Record the error (position + message)
2. Synchronize to the next safe point (`;` or `}`)
3. Continue parsing
4. Report all syntax errors at once (rather than stopping at the first one)

---

## 4. Stage 3: Analyzer

### 4.1 Responsibilities

Perform semantic analysis on the AST, producing a Typed AST. **All errors must be reported here.**

### 4.2 Analysis Passes

The Analyzer consists of a series of ordered Passes, each doing one thing:

```
Pass 1: SymbolResolution     — Build symbol table, check undefined references
Pass 2: TypeInference        — Infer expression types
Pass 3: UnitCheck            — Physical unit compatibility checking
Pass 4: SecurityAudit        — @auth/@requires/@hidden completeness checking
Pass 5: HardwareConstraint   — Neuron count, BRAM capacity, target compatibility checking
Pass 6: MigrationValidation  — Migration version chain integrity checking
```

### 4.3 Symbol Table

```
SymbolTable {
    // Scope stack
    scopes: Stack[Scope]

    // Global registries
    types:          Map[string, TypeDef]       // All known types
    functions:      Map[string, FnDef]         // All function signatures
    structs:        Map[string, StructDef]     // All structures
    policies:       Map[string, PolicyDef]     // All access policies
    layers:         Map[string, LayerDef]      // All network layers (Hardware)
    decorators:     Map[string, DecoratorDef]  // All known decorators

    // Methods
    resolve(name) -> Symbol | Error
    define(name, symbol) -> void | Error
    enter_scope() -> void
    exit_scope() -> void
}
```

### 4.4 Error Reporting Format

Follows the error code system defined in [Language Specification §9](./language-spec.md):

```
error[E205]: Dimension mismatch: layer 'output' expects 400, got 784
  --> src/model.yong:12:5
   |
12 |     connect input -> output with stdp;
   |     ^^^^^^^ 'input' has 784 outputs but 'output' expects 400 inputs
   |
   = help: add a hidden layer between them, or resize 'input' to 400
```

### 4.5 Extension Point

**New Analyzer Pass**: Register an object implementing the `AnalyzerPass` interface:

```
interface AnalyzerPass {
    name:     string
    order:    int              // Execution order (lower = earlier)
    run(ast: AST, symbols: SymbolTable) -> List[Diagnostic]
}
```

Examples:
- `PerformancePass` — Warn about potential performance issues (W3xx)
- `DeprecationPass` — Warn about deprecated syntax
- `StylePass` — Code style checking

---

## 5. Stage 4: IR (Intermediate Representation)

### 5.1 Responsibilities

Convert the Typed AST into a unified intermediate representation. IR is the shared input for all Backends.

### 5.2 IR Design Principles

| Principle | Description |
|-----------|-------------|
| **Dialect-agnostic** | IR doesn't distinguish App from Hardware; both are unified nodes and edges |
| **Type-complete** | Every IR node carries complete type information |
| **Declaration desugared** | Decorators are expanded into concrete IR properties |
| **Optimization-friendly** | Pass-level optimizations can be performed on the IR |

### 5.3 IR Node Types

```
YONG IR
├── DataNode         — Data definition (struct → table/storage)
│   { name, fields[], storage_hint?, constraints[] }
│
├── FunctionNode     — Function/interface
│   { name, params[], return_type, body: List[IRStmt],
│     properties { route?, method?, auth?, transaction? } }
│
├── ComponentNode    — UI component
│   { name, states[], view_tree, event_bindings[] }
│
├── NetworkNode      — SNN network
│   { layers[], connections[], config{} }
│
├── PolicyNode       — Access policy (desugared)
│   { resource_type, subject_type, rules[] }
│
├── MigrationNode    — Data migration (desugared)
│   { version, operations[] }
│
└── IRStmt           — IR-level statements
    ├── IRAssign      { target, value }
    ├── IRCall        { callee, args[] }
    ├── IRBranch      { condition, then_block, else_block }
    ├── IRLoop        { variable, iterable, body }
    ├── IRReturn      { value }
    ├── IRAssert      { condition, message }
    └── IREmit        { event, args[] }
```

### 5.4 Decorator Desugaring

Decorators are "desugared" into IR properties during the AST → IR phase:

```
AST:
  @api(POST, "/todos")
  @requires(TodoPolicy.create)
  @transaction
  fn add_todo(text: string) -> Todo { ... }

IR:
  FunctionNode {
    name: "add_todo"
    params: [("text", String)]
    return_type: Todo
    properties: {
      route: "/todos"
      method: POST
      auth: { policy: "TodoPolicy", action: "create" }
      transaction: { isolation: "READ_COMMITTED" }
    }
    body: [...]
  }
```

### 5.5 IR Optimization Passes

```
interface IRPass {
    name:     string
    order:    int
    run(ir: List[IRNode]) -> List[IRNode]
}
```

Built-in Passes:
- `DeadCodeElimination` — Remove unreachable code
- `ConstantFolding` — Fold constants
- `ResourceEstimation` — Hardware resource estimation (Hardware only)

Custom Passes:
- `EnergyOptimization` — Optimization based on energy budgets
- `TopologySimplification` — Merge equivalent network layers

---

## 6. Stage 5: Backend

### 6.1 Responsibilities

Convert YONG IR into target code. Each target corresponds to an independent Backend plugin.

### 6.2 Backend Interface

```
interface Backend {
    // Metadata
    name:           string          // "fpga_rtl", "web_app", "snn_sim"
    target_id:      string          // Used in config { target: xxx }
    version:        string
    file_extension: string          // ".sv", ".js", ".py"

    // Capability declaration
    supports_app_dialect:      bool
    supports_hardware_dialect: bool

    // Generation
    generate(ir: List[IRNode], config: CompilerConfig) -> GeneratedOutput

    // Validation
    validate(ir: List[IRNode]) -> List[Diagnostic]
}

GeneratedOutput {
    files:    Map[string, string]    // filename → content
    metadata: Map[string, Any]       // resource reports, timing estimates, etc.
}
```

### 6.3 Built-in Backends

| Backend | target_id | Input IR | Output |
|---------|-----------|----------|--------|
| `WebAppBackend` | `web` | FunctionNode, DataNode, ComponentNode, PolicyNode | HTML + CSS + JS + API Server |
| `FPGABackend` | `fpga` | NetworkNode | SystemVerilog + TCL |
| `ASICBackend` | `asic` | NetworkNode | Verilog (synthesizable) |
| `SimBackend` | `sim` | NetworkNode | Python SNN simulator |
| `MobileBackend` | `mobile` | FunctionNode, DataNode, ComponentNode | React Native / Flutter |

### 6.4 Backend Registration

```
CompilerRegistry {
    backends:           Map[string, Backend]
    decorator_handlers: Map[string, DecoratorHandler]
    type_registry:      TypeRegistry
    learning_rules:     LearningRuleRegistry
    neuron_types:       NeuronTypeRegistry
    analyzer_passes:    List[AnalyzerPass]
    ir_passes:          List[IRPass]

    // Registration methods
    register_backend(backend: Backend)
    register_decorator(name: string, handler: DecoratorHandler)
    register_type(name: string, type_def: TypeDef)
    register_learning_rule(name: string, rule_def: LearningRuleDef)
    register_neuron_type(name: string, neuron_def: NeuronTypeDef)
    register_analyzer_pass(pass: AnalyzerPass)
    register_ir_pass(pass: IRPass)
}
```

### 6.5 Extending Backend — Example

```python
# Want to support RISC-V? Just write a Backend plugin:

class RISCVBackend(Backend):
    name = "riscv_rtl"
    target_id = "riscv"
    version = "0.1.0"
    file_extension = ".sv"
    supports_app_dialect = False
    supports_hardware_dialect = True

    def generate(self, ir, config):
        # Convert NetworkNode to RISC-V accelerator RTL
        ...

    def validate(self, ir):
        # Check RISC-V specific constraints
        ...

# Register:
registry.register_backend(RISCVBackend())
```

---

## 7. Plugin Extension Interfaces

### 7.1 Decorator Plugin

```
interface DecoratorHandler {
    name:       string          // "@cache", "@rate_limit", ...
    dialect:    "app" | "hardware" | "both"
    targets:    List[string]    // Which node types it can attach to: ["fn", "struct", "component"]

    // AST → IR desugaring
    desugar(annotation: AnnotationNode, target: ASTNode) -> IRProperties

    // Semantic validation
    validate(annotation: AnnotationNode, target: ASTNode, symbols: SymbolTable) -> List[Diagnostic]
}
```

Example:
```python
class CacheDecorator(DecoratorHandler):
    name = "cache"
    dialect = "app"
    targets = ["fn"]

    def desugar(self, annotation, target):
        ttl = annotation.get_arg("ttl")  # e.g. "60s"
        return { "cache": { "ttl": ttl, "strategy": "lru" } }

    def validate(self, annotation, target, symbols):
        errors = []
        if target.return_type is None:
            errors.append(Diagnostic("E105", "Cannot cache void function"))
        return errors
```

### 7.2 Learning Rule Plugin

```
interface LearningRuleDef {
    name:       string          // "stdp", "bcm", "oja", "r_stdp"
    params:     List[ParamDef]  // Parameter definitions (name/type/default/range)

    // Semantic definition (pseudocode form, for conformance testing)
    weight_update(weight, pre, post, params) -> weight

    // IR generation: return learning rule's IR representation
    to_ir(params) -> IRNode
}
```

### 7.3 Neuron Type Plugin

```
interface NeuronTypeDef {
    name:       string          // "lif", "izhikevich", "adex"
    params:     List[ParamDef]  // Parameter definitions

    // Semantic definition
    dynamics(state, input, params) -> (state, spike)

    // IR generation
    to_ir(params) -> IRNode
}
```

---

## 8. Compilation Flow API

### 8.1 One-Line Compilation

```python
# Simplest usage
output = yong_compile("source.yong", target="web")

# Equivalent to:
tokens   = Tokenizer(source).tokenize()          # Stage 1
ast      = Parser(tokens).parse()                 # Stage 2
typed    = Analyzer(ast, registry).analyze()       # Stage 3
ir       = IRGenerator(typed).generate()           # Stage 4
output   = registry.get_backend("web").generate(ir) # Stage 5
```

### 8.2 Staged Compilation (for IDE / Debug)

```python
# IDE scenario: Stages 1-3 are sufficient (syntax highlighting + error hints)
tokens = Tokenizer(source).tokenize()
ast    = Parser(tokens).parse()
diags  = Analyzer(ast, registry).analyze()
# → diags sent to IDE
```

### 8.3 Multi-Target Compilation

```python
# Same source, compile to multiple targets
ir = compile_to_ir("network.yong")     # Stages 1-4

fpga_out = registry.get_backend("fpga").generate(ir)   # → .sv
sim_out  = registry.get_backend("sim").generate(ir)     # → .py
asic_out = registry.get_backend("asic").generate(ir)    # → .v
```

---

## 9. Conformance Requirements

### 9.1 Backend Conformance

Any Backend plugin claiming "YONG Compatible" must:

1. Implement all methods of the `Backend` interface
2. Pass the Conformance Tests defined in [Language Specification §11](./language-spec.md)
3. For the same `.yong` + same IR, multiple generations must produce deterministically identical output
4. Hardware Backends must satisfy **bit-accurate** requirements

### 9.2 Analyzer Conformance

- Follow the error code system in [Language Specification §9](./language-spec.md)
- Same source + same compiler version → same set of errors/warnings
- Silent error suppression is not allowed

### 9.3 IR Conformance

- Same AST → same IR (deterministic)
- IR nodes must carry complete type information
- Decorators must be desugared at the IR stage

---

## 10. Relationship to Existing Implementation

### 10.1 `vgo_parser.py` (Current State)

The existing `src/vgo_computing/tools/vgo_parser.py` is an early prototype:

| Spec Requirement | Current State | Gap |
|-----------------|---------------|-----|
| 5-stage pipeline | Tokenizer + Parser + CodeGen (3 stages, no Analyzer or IR) | Missing Stages 3, 4 |
| Pluggable backends | `to_python()` / `to_verilog()` hardcoded in one class | Needs refactoring into Backend plugins |
| Two dialect support | Only Hardware (network/layer/connect/config) | Missing App dialect |
| Symbol table + Type checking | None | Needs creation |
| Error code system | `raise SyntaxError(...)` raw exceptions | Needs §9 format implementation |
| IR | None | Needs creation |

### 10.2 Evolution Path

```
Phase 1 (current): Prototype — vgo_parser.py, mixed 3-stage pipeline, SNN only
     ↓
Phase 2: Refactor — Split into 5 stages, introduce AnalyzerPass and Backend interface
     ↓
Phase 3: Extend — Add App dialect Parser + WebAppBackend
     ↓
Phase 4: Complete — IR optimization passes, multi-target, IDE integration
```

Each Phase can be delivered independently without blocking others.

---

## 11. Glossary

| Concept | Description |
|---------|-------------|
| `Token` | Lexical unit (keyword/identifier/literal/symbol) |
| `AST` | Abstract Syntax Tree, Parser output |
| `Typed AST` | AST with type annotations, Analyzer output |
| `IR` | Intermediate Representation, shared input for all Backends |
| `Backend` | Target code generator (pluggable) |
| `AnalyzerPass` | A sub-step of semantic analysis (pluggable) |
| `IRPass` | A sub-step of IR optimization (pluggable) |
| `DecoratorHandler` | Desugaring and validation logic for decorators (pluggable) |
| `CompilerRegistry` | Global registry, entry point for all plugin registration |
| `Diagnostic` | Compile error/warning, follows Language Specification §9 format |

---

*YONG Compiler Specification v1.0*  
*Frozen pipeline, extensible everything else.*  
*"核心管线不变，插件随意扩展。"*
