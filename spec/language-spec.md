# YongnianLang Language Specification

> **Version**: v4.2  
> **Updated**: 2026-02-10  
> **File Extension**: `.yong`

---

## ⚠️ On Extensibility — Read This Before the Spec

**YongnianLang is an open, evolving language.**

This document defines the language's current state, not its final form. YONG was designed to be extensible from day one — you don't need to wait for official updates. When you encounter a need not covered here, **extend it yourself**.

### Frozen (Invariants)

The following form the language's core skeleton and will never change:

| Frozen | Why |
|--------|-----|
| **Declarative philosophy** — write intent, not implementation | This is YONG's reason to exist |
| **Decorator syntax** — `@name(args)` | All extensions are injected through decorators; this is the unified entry point |
| **Two dialects** — App + Hardware | One for applications, one for hardware |
| **Data-first** — structs first | Data structures are the foundation of all behavior |
| **Unit safety** — compile-time physical quantity checks | Cross-dialect type safety guarantee |
| **Bit-accurate semantics** — same source, same result | The same `.yong` file must behave identically across all backends |

### Extensible (Open)

The following can be extended on demand without modifying the language core:

| Extensible | How | Example |
|------------|-----|---------|
| **New decorators** | Define a new `@name` | `@cache(ttl=60s)`, `@rate_limit(100/min)`, `@log` |
| **New types** | Add BaseType or UnitType | `image`, `audio`, `tensor`, `Energy<eV>` |
| **New config keys** | Add inside `config {}` blocks | `power_mode: low`, `clock: 100MHz`, `pipeline: deep` |
| **New learning rules** | Add after `connect ... with` | `with r_stdp`, `with bcm`, `with oja` |
| **New neuron types** | Add after `type=` | `type=izhikevich`, `type=adex`, `type=hodgkin_huxley` |
| **New compilation targets** | Compiler backend plugins | RISC-V RTL, WebAssembly, Quantum circuits |
| **New UI widgets** | `@ui(widget)` widget values | `@ui(slider)`, `@ui(map)`, `@ui(chart)` |
| **New policy actions** | Verbs after `allow/deny` | `allow export`, `allow share`, `allow archive` |
| **New migration operations** | Directives inside `migrate` blocks | `rename_table(...)`, `merge_columns(...)` |
| **New error codes** | E/W + new numbers | `E6xx` security audit, `E7xx` performance constraints |

### Extension Principles

Extending YONG requires following three rules:

1. **Declare, never implement** — New keywords/decorators must also be declarative; implementation is handled by the compiler
2. **Never break existing code** — Extensions are additive only. Existing `.yong` files must remain valid in new versions
3. **Units must have units** — New physical quantity types must carry unit dimensions

```yong
// ✓ Good extension — declarative, no implementation details
@cache(ttl=60s)
@api(GET, "/stats")
fn get_stats() -> Stats { ... }

// ✓ Good extension — new learning rule, declarative
connect hidden -> output with bcm(threshold=0.5, rate=0.01);

// ✓ Good extension — new neuron type
layer cortex(1000, type=adex, adapt_tau=100ms);

// ✗ Bad extension — exposes implementation details, violates declarative principle
layer cortex(1000, type=lif, verilog_module="my_custom_lif.sv");
```

> **Summary**: This document is YONG's "current snapshot." If you need a feature not covered here, add it following the core principles. Don't wait.

---

## 1. Overview

YongnianLang (永年语言, abbreviated YONG) is a declarative programming language. Programmers describe intent; the compiler materializes it to concrete platforms.

**Core Principles**:

| Principle | Meaning |
|-----------|---------|
| **Declarative** | Write *what*, not *how* |
| **Data-first** | Define data structures before behavior |
| **Unit safety** | Physical quantities carry units; incompatible units cannot be mixed |
| **AI-friendly** | Minimum tokens to express maximum complexity |

### 1.1 Hello World

```yong
@app(route="/")
component Hello {
    view { Header("Hello, Yongnian!") }
}
```

### 1.2 Todo App in 30 Lines

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

---

## 2. Two Dialects

YONG has two application domains, sharing the same syntactic philosophy:

```
YongnianLang (永年语言)
├── 🌐 App Dialect     — @ui, @api, @db, component, @auth, @migration
│   └── Compiles to: Web App / Mobile App / API Server
│
└── 🧠 Hardware Dialect — network, layer, connect, config
    └── Compiles to: FPGA RTL / ASIC RTL / SNN Simulator
```

**Shared Design**:
1. **Decorator-driven** — `@ui`, `@api`, `config {}` are all compiler directives
2. **No implementation details** — No `<div>`, no `always @(posedge clk)`
3. **Type-safe** — Type errors and unit incompatibility caught at compile time

---

## 3. Formal Grammar (EBNF)

> Compiler frontends, IDE plugins, and AI Agents must follow this as the source of truth.

### 3.1 Top-Level Structure

```ebnf
(* ========== Top Level ========== *)
Program         = { TopLevelDecl } ;
TopLevelDecl    = Import
                | StructDecl
                | EnumDecl
                | FnDecl
                | ComponentDecl
                | NetworkDecl
                | PolicyDecl
                | MigrationDecl ;

Import          = "import" ModulePath [ "as" IDENT ] ";" ;
ModulePath      = IDENT { "." IDENT } ;

(* ========== Universal Decorators ========== *)
Annotation      = "@" IDENT [ "(" AnnotationArgs ")" ] ;
AnnotationArgs  = AnnotationArg { "," AnnotationArg } ;
AnnotationArg   = STRING | NUMBER | IDENT | KeyValue ;
KeyValue        = IDENT "=" ( STRING | NUMBER | IDENT | "true" | "false" ) ;
```

### 3.2 App Dialect Grammar

```ebnf
(* ========== Data Structures ========== *)
StructDecl      = { Annotation } "struct" IDENT "{" { FieldDecl } "}" ;
FieldDecl       = { Annotation } IDENT ":" TypeExpr [ "=" Expr ] ;

EnumDecl        = "enum" IDENT "{" IDENT { "," IDENT } "}" ;

TypeExpr        = BaseType | GenericType | NullableType | UnitType ;
BaseType        = "int" | "float" | "bool" | "string"
                | "time" | "money" | "url" | "email" | "uid" ;
GenericType     = IDENT "[" TypeExpr { "," TypeExpr } "]" ;
NullableType    = TypeExpr "?" ;
UnitType        = IDENT "<" UNIT ">" ;
UNIT            = "ms" | "us" | "ns" | "s"
                | "pJ" | "nJ" | "fJ" | "J"
                | "mW" | "W"
                | "mV" | "V"
                | "Hz" | "kHz" | "MHz" ;

(* ========== Functions ========== *)
FnDecl          = { Annotation } "fn" IDENT "(" [ ParamList ] ")" [ "->" TypeExpr ] Block ;
ParamList       = Param { "," Param } ;
Param           = IDENT ":" TypeExpr ;

(* ========== Components ========== *)
ComponentDecl   = { Annotation } "component" IDENT "{" { ComponentBody } "}" ;
ComponentBody   = StateDecl | ViewDecl | EffectDecl | HandlerDecl ;
StateDecl       = "state" IDENT "=" Expr ;
ViewDecl        = "view" "{" { ViewElement } "}" ;
EffectDecl      = "effect" "(" EventRef ")" Block ;
HandlerDecl     = "on" EventRef Block ;

ViewElement     = WidgetCall | ConditionalView | ListExpr ;
WidgetCall      = IDENT "(" [ ArgList ] ")" [ "->" LambdaView ] ;
ConditionalView = "if" "(" Expr ")" "{" { ViewElement } "}" [ "else" "{" { ViewElement } "}" ] ;
ListExpr        = "List" "(" Expr ")" "->" "|" IDENT "|" "{" { ViewElement } "}" ;
LambdaView      = "|" IDENT "|" "{" { ViewElement } "}" ;

(* ========== Access Policies ========== *)
PolicyDecl      = "policy" IDENT "{" PolicyBody "}" ;
PolicyBody      = ResourceBind SubjectBind { PolicyRule } ;
ResourceBind    = "resource" ":" IDENT ;
SubjectBind     = "subject" ":" IDENT ;
PolicyRule      = ( "allow" | "deny" ) PolicyAction "when" Expr
                | ( "allow" | "deny" ) "*" ;
PolicyAction    = "read" | "create" | "update" | "delete" | "*" ;

(* ========== Data Migrations ========== *)
MigrationDecl   = "@migration" "(" MigrationArgs ")" "migrate" IDENT Block ;
MigrationArgs   = "version" "=" NUMBER "," "from" "=" NUMBER ;
```

### 3.3 Hardware Dialect Grammar

```ebnf
(* ========== Networks ========== *)
NetworkDecl     = "network" IDENT "{" { NetworkBody } "}" ;
NetworkBody     = LayerDecl | ConnectDecl | ConfigBlock ;

LayerDecl       = "layer" IDENT "(" NUMBER [ "," { KeyValue } ] ")" ";" ;
ConnectDecl     = "connect" IDENT "->" IDENT [ "with" LearningRule ] ";" ;
LearningRule    = "stdp" [ "(" { KeyValue } ")" ]
                | "anti_hebbian"
                | "none" ;

ConfigBlock     = "config" "{" { ConfigEntry } "}" ;
ConfigEntry     = IDENT ":" ConfigValue ;
ConfigValue     = NUMBER | STRING | IDENT | HexLiteral | "enable" | "disable"
                | UnitLiteral ;
HexLiteral      = "0x" HEX_DIGIT { HEX_DIGIT | "_" } ;
UnitLiteral     = NUMBER UNIT ;
```

### 3.4 Expressions and Statements

```ebnf
(* ========== Statements ========== *)
Block           = "{" { Statement } "}" ;
Statement       = LetStmt | ReturnStmt | IfStmt | ForStmt | AssertStmt | EmitStmt | ExprStmt ;
LetStmt         = "let" IDENT [ ":" TypeExpr ] "=" Expr ";" ;
ReturnStmt      = "return" Expr ";" ;
IfStmt          = "if" "(" Expr ")" Block [ "else" Block ] ;
ForStmt         = "for" IDENT "in" Expr Block ;
AssertStmt      = "assert" "(" Expr "," STRING ")" ";" ;
EmitStmt        = "emit" IDENT "(" [ ArgList ] ")" ";" ;
ExprStmt        = Expr ";" ;

(* ========== Expressions (precedence low to high) ========== *)
Expr            = PipeExpr ;
PipeExpr        = TernaryExpr { "|>" FnCall } ;
TernaryExpr     = OrExpr [ "?" Expr ":" Expr ] ;
OrExpr          = AndExpr { "||" AndExpr } ;
AndExpr         = CompExpr { "&&" CompExpr } ;
CompExpr        = AddExpr { ( "==" | "!=" | "<" | ">" | "<=" | ">=" | "in" ) AddExpr } ;
AddExpr         = MulExpr { ( "+" | "-" ) MulExpr } ;
MulExpr         = UnaryExpr { ( "*" | "/" | "%" ) UnaryExpr } ;
UnaryExpr       = [ "!" | "-" ] PrimaryExpr ;
PrimaryExpr     = NUMBER | STRING | "true" | "false"
                | IDENT [ "." IDENT ] [ "(" [ ArgList ] ")" ]
                | "(" Expr ")"
                | UnitLiteral
                | ArrayLiteral ;
ArrayLiteral    = "[" [ Expr { "," Expr } ] "]" ;
ArgList         = Expr { "," Expr } ;
FnCall          = IDENT "(" [ ArgList ] ")" ;

(* ========== Lexical ========== *)
IDENT           = LETTER { LETTER | DIGIT | "_" } ;
NUMBER          = DIGIT { DIGIT } [ "." DIGIT { DIGIT } ] ;
STRING          = '"' { CHAR } '"' ;
LETTER          = "a".."z" | "A".."Z" | "_" ;
DIGIT           = "0".."9" ;
HEX_DIGIT       = DIGIT | "a".."f" | "A".."F" ;

(* ========== Comments ========== *)
LineComment     = "//" { CHAR } NEWLINE ;
BlockComment    = "/*" { CHAR } "*/" ;
```

---

## 4. Type System

### 4.1 Base Types

| Type | Description | Example |
|------|-------------|---------|
| `int` | Integer | `42` |
| `float` | Floating point | `3.14` |
| `bool` | Boolean | `true`, `false` |
| `string` | String | `"hello"` |
| `uid` | Unique identifier (UUID) | Auto-generated |
| `time` | Timestamp | `created_at: time` |
| `money` | Monetary amount | `balance: money` |
| `email` | Email address | Compiler auto-attaches format validation |
| `url` | URL address | Compiler auto-attaches format validation |
| `void` | No return value | `fn foo() -> void` |

### 4.2 Compound Types

```yong
list[T]                   // Ordered collection
map[K, V]                 // Key-value mapping
T?                        // Nullable type (must null-check before use)
```

### 4.3 Physical Unit Types

```yong
// Time
Time<ms>    Time<us>    Time<ns>    Time<s>

// Energy
Energy<pJ>  Energy<nJ>  Energy<fJ>  Energy<J>

// Power
Power<mW>   Power<W>

// Voltage
Voltage<mV> Voltage<V>

// Frequency
Frequency<Hz>   Frequency<kHz>  Frequency<MHz>
```

### 4.4 Unit Safety

Incompatible physical units produce compile-time errors:

```yong
let x: Time<ms> = 5ms;
let y: Energy<pJ> = 10pJ;
let z = x + y;      // ✗ E202: Unit incompatible: cannot add 'Time<ms>' and 'Energy<pJ>'
```

Same-dimension units allow implicit conversion:

```yong
let a: Time<ms> = 5ms;
let b: Time<us> = 3000us;
let c = a + b;       // ✓ Result: Time<ms> = 8ms (compiler auto-converts)
```

### 4.5 Field Annotations

| Annotation | Meaning |
|------------|---------|
| `@unique` | Unique constraint on field value |
| `@hidden` | Not serialized to API responses (for passwords and other sensitive fields) |
| `@ui(widget)` | Specifies UI widget type |

---

## 5. App Dialect

### 5.1 Data Definition

```yong
@db(table="todos")
struct Todo {
    id: uid
    @ui(text, title)
    text: string
    @ui(checkbox)
    done: bool
    priority: int = 0
}
```

`@db(table=...)` declares data persistence. The compiler auto-generates database tables, serialization, and ORM mappings.

### 5.2 API Definition

```yong
@api(GET, "/todos")
fn list_todos() -> list[Todo] {
    return db.todos.all()
}

@api(POST, "/todos")
fn add_todo(text: string) -> Todo {
    return db.todos.create({ text, done: false })
}

@api(PUT, "/todos/:id")
fn update_todo(id: uid, done: bool) -> Todo {
    return db.todos.update(id, { done })
}

@api(DELETE, "/todos/:id")
fn delete_todo(id: uid) -> void {
    db.todos.delete(id)
}
```

### 5.3 Component Definition

```yong
@app(route="/")
component TodoApp {
    state todos = query(list_todos)
    state filter = "all"

    view {
        Header("My Todos")
        Input(placeholder="What needs to be done?", on_enter=add_todo)

        Tabs(["all", "active", "completed"], selected=filter)

        List(todos |> filter_by(filter)) -> |todo| {
            Row {
                Checkbox(todo.done, on_change=update_todo(todo.id, !todo.done))
                Text(todo.text, style=todo.done ? "strike" : "normal")
                Button("×", on_click=delete_todo(todo.id))
            }
        }
    }
}
```

### 5.4 Pipe Operator

```yong
users
    |> filter(u => u.active)
    |> map(u => u.name)
    |> sort()
```

### 5.5 Authentication and Authorization

```yong
// -- User model --
@db(table="users")
@auth(identity)                        // Mark as authentication subject
struct User {
    id: uid
    email: email @unique
    password_hash: string @hidden       // @hidden = excluded from API responses
    role: Role
    created_at: time
}

enum Role { admin, editor, viewer }

// -- Auth configuration --
config auth {
    provider: jwt                       // jwt | session | oauth2
    secret_env: "JWT_SECRET"            // Read secret from environment variable
    token_expiry: 24h
    refresh_enabled: true
    password_hash: bcrypt(rounds=12)
}

// -- Access policy --
policy TodoPolicy {
    resource: Todo
    subject:  User

    allow read   when subject.role in [admin, editor, viewer]
    allow create when subject.role in [admin, editor]
    allow update when subject.role == admin
                   or resource.owner_id == subject.id
    allow delete when subject.role == admin

    deny *                              // Default deny
}

// -- Apply policy to API --
@api(DELETE, "/todos/:id")
@requires(TodoPolicy.delete)            // Compiler injects auth middleware
fn delete_todo(id: uid) -> void {
    db.todos.delete(id)
}
```

Compiler behavior:
- `@auth(identity)` → Generates JWT issuance/verification logic
- `policy` → Generates RBAC check middleware
- `@requires` → Auto-injected into API endpoints
- `@hidden` → Stripped from all GET responses
- `@api` without `@requires` → Compile warning W201

### 5.6 Data Migrations

```yong
@migration(version=2, from=1)
migrate AddPriorityToTodo {
    alter_table("todos") {
        add_column("priority", int, default=0)
        add_index("idx_priority", ["priority"])
    }
}

@migration(version=3, from=2)
migrate AddOwnerToTodo {
    alter_table("todos") {
        add_column("owner_id", uid, references="users.id")
    }
}
```

Compiler behavior:
- Generates SQL ALTER TABLE (or target database equivalent)
- Auto-generates rollback scripts (down migration)
- Maintains `_yong_migrations` table for version tracking
- Auto-checks and executes pending migrations on startup
- Version conflicts → Compile error E401

### 5.7 Transactions

```yong
@api(POST, "/transfer")
@requires(TransferPolicy.execute)
@transaction
fn transfer(from_id: uid, to_id: uid, amount: money) -> void {
    let from = db.accounts.get(from_id)
    let to   = db.accounts.get(to_id)

    assert(from.balance >= amount, "Insufficient funds")  // Failure → auto ROLLBACK

    db.accounts.update(from_id, { balance: from.balance - amount })
    db.accounts.update(to_id,   { balance: to.balance + amount })

    emit TransferCompleted(from_id, to_id, amount)        // Delivered only after COMMIT
}
```

**Transaction Semantics**:

| Behavior | Rule |
|----------|------|
| Scope | `@transaction` function body = one transaction boundary |
| Isolation level | Default `READ_COMMITTED`; override via `@transaction(isolation=serializable)` |
| Failure rollback | `assert` failure or uncaught exception → auto `ROLLBACK` |
| Event delivery | `emit` events delivered only after successful `COMMIT` (outbox pattern) |
| Nesting | Default uses `SAVEPOINT`; `@transaction(nested=false)` disables nesting |

### 5.8 Event Safety Constraints

All `on_*` event handlers (`on_enter`, `on_click`, `on_change`) must ultimately call `@api` functions that satisfy `@requires` authorization checks. The compiler statically verifies authorization completeness across the call chain at compile time.

---

## 6. Hardware Dialect

### 6.1 Network Declaration

```yong
network MyNetwork {
    layer input(64);
    layer neurons(100, type=lif, threshold=200, leak=2);
    connect input -> neurons with stdp;

    config {
        weight_storage: bram
        wta_mode: enable
        homeostasis: enable
        energy_tracking: enable
    }
}
```

### 6.2 Keywords

| Keyword | Meaning |
|---------|---------|
| `layer name(N)` | Define an input layer of width N |
| `layer name(N, type=lif, ...)` | Define N LIF neurons with configurable threshold and leak |
| `connect A -> B with stdp` | Fully-connected A to B with STDP learning |
| `connect A -> B with none` | Fully-connected A to B with fixed weights |
| `weight_storage: bram` | Weights stored in Block RAM (for large networks) |
| `weight_storage: lut` | Weights stored in registers (for small networks, default) |
| `wta_mode: enable` | Enable Winner-Take-All competition |
| `homeostasis: enable` | Enable adaptive threshold regulation |
| `energy_tracking: enable` | Enable spike counting / energy statistics |
| `threshold_mode: dual` | Separate training and inference thresholds |

### 6.3 STDP Parameter Configuration

```yong
connect input -> neurons with stdp(
    ltp=8,              // LTP increment (default=8, range 0-255)
    ltd=3,              // LTD increment (default=3, range 0-255)
    w_min=30,           // Weight lower bound (default=30)
    w_max=250,          // Weight upper bound (default=250)
    div=3               // Divider (default=3 → learn every 4 steps, 0=learn every step)
);
```

### 6.4 Homeostasis Parameter Configuration

```yong
config {
    homeostasis: enable
    target_rate: 20         // Target firing rate (default=20)
    revival_threshold: 50   // Revive after this many consecutive non-wins (default=50)
    w_sum_max: 220          // Weight sum scaling threshold (default=220)
}
```

### 6.5 Multi-Layer Network Example

```yong
network DeepSNN {
    layer input(784);                                   // 28×28 image input
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

---

## 7. Executable Semantic Definitions

> The following definitions are normative. The same `.yong` file compiled to any backend
> must produce identical behavior. Pseudocode uses Python-style notation.

### 7.1 LIF Neuron Model

**Applies to**: `layer name(N, type=lif, threshold=T, leak=L)`

```python
class LIFNeuron:
    """
    Leaky Integrate-and-Fire neuron. Fixed-point arithmetic, 16-bit membrane potential.

    Parameters:
      threshold:  Base threshold (default=200)
      leak:       Leak amount per timestep (default=2)
      refrac:     Refractory period length (default=3 timesteps)
    """

    def __init__(self, threshold=200, leak=2, refrac=3):
        self.membrane = 0           # uint16, membrane potential
        self.refrac_cnt = 0         # uint16, refractory period counter
        self.spike_count = 0        # uint16, cumulative spike count
        self.spike_window = 0       # uint16, current window spike count (homeostasis)
        self.thresh_adj = 500       # uint16, threshold offset (homeostasis, center=500)
        self.no_win_cnt = 0         # uint8,  consecutive non-win count (WTA revival)

    def integrate(self, weighted_sum, leak):
        """Integration phase: accumulate input current, subtract leak."""
        if self.membrane + weighted_sum > leak:
            self.membrane = self.membrane + weighted_sum - leak
        else:
            self.membrane = 0

    def fire(self, eff_threshold, is_wta_winner) -> bool:
        """Fire decision."""
        if self.refrac_cnt > 0:
            self.refrac_cnt -= 1
            return False

        if self.membrane >= eff_threshold:
            if is_wta_winner:
                self.membrane = 0
                self.refrac_cnt = self.refrac
                self.spike_count += 1
                self.spike_window += 1
                return True
            else:
                # WTA loser: no fire, halve membrane potential (soft inhibition)
                self.membrane >>= 1
                return False
        return False
```

### 7.2 STDP Learning Rule

**Applies to**: `connect A -> B with stdp`

#### 7.2.1 Trigger Conditions

```python
def should_run_stdp(learn_en, div_counter, has_winner):
    """
    Three prerequisites for STDP execution:
      1. learn_en == True   (learning enabled)
      2. div_counter == 0   (divider expired)
      3. has_winner == True  (WTA produced at least one winner)
    """
    return learn_en and div_counter == 0 and has_winner
```

#### 7.2.2 Weight Update Formula

```python
def stdp_update(weight: uint8, input_active: bool,
                ltp=8, ltd=3, w_min=30, w_max=250) -> uint8:
    """
    Single synapse weight update (Winner-Take-All STDP variant).

    Only WTA winner weights are updated.
    Update direction determined by whether input is active.

      - Input active → LTP (Long-Term Potentiation): weight += ltp
      - Input silent → LTD (Long-Term Depression): weight -= ltd

    Result clamped to [w_min, w_max] range.
    """
    if input_active:
        new_weight = min(weight + ltp, w_max)
    else:
        new_weight = max(weight - ltd, w_min)

    return clamp(new_weight, w_min, w_max)
```

#### 7.2.3 Divider Control

```python
class STDPDivider:
    """
    Divider controls STDP execution frequency to prevent weight oscillation.

    div_max=0 → learn every timestep
    div_max=3 → learn every 4 timesteps (default)
    """
    def __init__(self, div_max=3):
        self.counter = 0
        self.div_max = div_max

    def tick(self) -> bool:
        """Called once per timestep. Returns True if STDP should execute this step."""
        if self.counter == 0:
            self.counter = self.div_max
            return True
        else:
            self.counter -= 1
            return False
```

#### 7.2.4 Consistency Requirement

> **Bit-accuracy**: Any backend (FPGA / ASIC / simulator) must produce exactly the same
> weight matrix for the same input sequence and same STDP parameters. Floating-point
> approximation is not allowed. All operations use fixed-point integers.

### 7.3 WTA (Winner-Take-All)

**Applies to**: `wta_mode: enable`

#### 7.3.1 Selection Algorithm

```python
def wta_select(membranes, N, K=2, lfsr_noise, start_idx=0):
    """
    Select Top-K from N neurons (default K=2).

    Parameters:
      membranes:  Membrane potential array for each neuron
      N:          Total number of neurons
      K:          Number of winners (default=2, i.e., champion + runner-up both fire)
      lfsr_noise: Random number generator output (for tie-breaking)
      start_idx:  Scan start index (rotates +1 after each spike, prevents starvation)

    Returns: List of winner indices (length=K)
    """
    winners = []  # (value, index) sorted by value descending, capacity=K

    for offset in range(N):
        i = (start_idx + offset) % N

        if neuron_disabled(i) or refrac_cnt[i] > 0:
            continue

        noisy_val = membranes[i] + (lfsr_noise & 0xF)  # Low 4-bit noise

        # Insert into sorted top-k winners list
        insert_into_topk(winners, (noisy_val, i), K)

    return [idx for (val, idx) in winners]
```

#### 7.3.2 Tie-Breaking

Ties are broken through the following mechanisms (priority high to low):

1. **LFSR noise injection**: `noisy_val = membrane + lfsr[3:0]` — even with identical membrane potentials, noise likely breaks ties
2. **Start rotation (Anti-Starvation)**: `start_idx` increments by 1 after each spike, ensuring long-term fairness
3. **Deterministic resolution**: If the first two mechanisms fail to break a tie, the neuron encountered first in scan order wins

#### 7.3.3 Winner Tax

```python
def apply_winner_tax(winner_idx, thresh_adj, win_tax=5):
    """
    Prevent winner monopolization. Raise winner's threshold each time it wins WTA.

    Parameters:
      win_tax: Tax coefficient (default=5)
      Actual increment = win_tax × 4
    """
    delta = win_tax << 2                           # Default 5×4=20
    thresh_adj[winner_idx] = min(
        thresh_adj[winner_idx] + delta,
        60000                                      # Saturation upper bound
    )
```

### 7.4 Homeostasis

**Applies to**: `homeostasis: enable`

#### 7.4.1 Execution Timing

```
Processing phases per timestep:

  INTEGRATE → WTA → FIRE → LEARN(STDP) → HOMEOSTASIS → WEIGHT_SCALE
                                          ↑
                                       Executed here
```

#### 7.4.2 Threshold Adjustment

```python
def homeostasis_update(neuron_idx, spike_window, thresh_adj, target_rate=20):
    """
    Executed independently for each neuron.

    Rules:
      - Firing > target_rate → increase threshold (+4, inhibit)
      - Firing < target_rate/2 → decrease threshold (-8, excite)

    Asymmetry note:
      Threshold decrease step (8) > increase step (4).
      Design intent: activating silent neurons is more important than inhibiting active ones.
    """
    if target_rate == 0:
        return

    if spike_window[neuron_idx] > target_rate:
        thresh_adj[neuron_idx] = min(thresh_adj[neuron_idx] + 4, 60000)
    elif spike_window[neuron_idx] < (target_rate >> 1):
        thresh_adj[neuron_idx] = max(thresh_adj[neuron_idx] - 8, 8)

    spike_window[neuron_idx] = 0  # Reset window
```

#### 7.4.3 Effective Threshold Calculation

```python
def get_effective_threshold(thresh_adj, base_threshold):
    """
    thresh_adj centered at 500:
      adj > 500 → effective threshold above base (inhibition)
      adj < 500 → effective threshold below base (excitation)
      adj = 500 → effective threshold = base
    """
    if thresh_adj >= 500:
        return base_threshold + (thresh_adj - 500)
    else:
        delta = 500 - thresh_adj
        return max(base_threshold - delta, 1)  # Minimum threshold = 1
```

#### 7.4.4 Dead Neuron Revival

```python
def revival_check(neuron_idx, no_win_cnt, thresh_adj, is_winner):
    """
    In WTA mode, a neuron that hasn't won for revival_threshold consecutive
    steps is deemed "dead" and revived.

    Revival mechanism: drastically lower thresh_adj → easily activated in next WTA round.
    """
    if is_winner:
        no_win_cnt[neuron_idx] = 0
        return

    no_win_cnt[neuron_idx] = min(no_win_cnt[neuron_idx] + 1, 255)

    if no_win_cnt[neuron_idx] >= REVIVAL_THRESHOLD:   # Default=50
        no_win_cnt[neuron_idx] = 0
        thresh_adj[neuron_idx] = 200                  # Far below center 500 → easy to activate
```

#### 7.4.5 Weight Scaling

```python
def weight_scaling(neuron_idx, weights, N_INPUTS, w_sum_max=220):
    """
    Prevent total weight from growing without bound.

    Trigger: sum(weights) > w_sum_max × 64
    Action: Right-shift all weights by 4 bits (÷16)
    """
    if w_sum_max == 0:
        return

    target_sum = w_sum_max << 6
    actual_sum = sum(weights[neuron_idx, j] for j in range(N_INPUTS))

    if actual_sum > target_sum:
        for j in range(N_INPUTS):
            weights[neuron_idx, j] >>= 4
```

---

## 8. Compiler Rules

> The following rules are constraints that compiler implementers must follow.

### 8.1 Core Principle

```
Programmer writes:     weight_storage: bram
Compiler generates:    Block RAM instantiation + read/write pipeline + address generation

Programmer writes:     @api(POST, "/todos")
Compiler generates:    Route registration + middleware + error handling + serialization

Programmer writes:     connect input -> neurons with stdp
Compiler generates:    All STDP logic + weight clamping + divider control
```

### 8.2 Hardware Dialect Rules

| Rule | Description |
|------|-------------|
| BRAM read latency | When `weight_storage: bram`, compiler must handle at least 2 clock read latency |
| Index bit-width | `neuron_idx = ceil(log2(N))`, `input_idx = ceil(log2(N_INPUTS)) + 1` |
| Resource adaptation | Compiler auto-selects parallel or serial implementation based on `target` |
| Register mapping | Compiler auto-generates control/status registers and their bus interfaces |

### 8.3 App Dialect Rules

| Rule | Description |
|------|-------------|
| Explicit permissions | `@api` without `@requires` → Warning W201 |
| Sensitive fields | Password-like fields without `@hidden` → Error E302 |
| Transaction purity | No async operations inside `@transaction` → Error E303 |
| Policy references | `@requires` referencing non-existent `policy` → Error E301 |

---

## 9. Compile Errors and Warnings

### 9.1 Error Code System

```
Format: [Category Letter][3-digit number]

E = Error   (compilation terminates)
W = Warning (can continue)

  E1xx — Syntax errors
  E2xx — Type errors
  E3xx — Semantic/Security errors
  E4xx — Migration/Data errors
  E5xx — Hardware constraint errors
  W1xx — Syntax warnings
  W2xx — Security warnings
  W3xx — Performance warnings
```

### 9.2 Error Catalog

#### Syntax Errors (E1xx)

| Code | Message Template | Description |
|------|-----------------|-------------|
| E101 | `Unexpected token '{token}' at {line}:{col}` | Token does not conform to EBNF |
| E102 | `Missing ';' after statement at {line}:{col}` | Missing semicolon |
| E103 | `Undefined keyword '{keyword}'` | Invalid keyword |
| E104 | `Duplicate field '{name}' in struct '{struct}'` | Duplicate struct field |
| E105 | `Invalid annotation '@{name}' for target '{target}'` | Decorator not applicable to current context |
| E106 | `Unmatched brace at {line}:{col}` | Unmatched braces |

#### Type Errors (E2xx)

| Code | Message Template | Description |
|------|-----------------|-------------|
| E201 | `Type mismatch: cannot assign '{rhs}' to '{lhs}'` | Type incompatibility |
| E202 | `Unit incompatible: cannot add '{unit_a}' and '{unit_b}'` | Physical unit incompatibility |
| E203 | `Type '{type}' is not iterable` | Using List/for on non-collection type |
| E204 | `Unknown type '{type}'` | Undefined type |
| E205 | `Dimension mismatch: layer '{layer}' expects {expected}, got {actual}` | Inter-layer dimension mismatch |
| E206 | `Nullable type '{type}?' requires null-check before use` | Missing null safety check |

#### Semantic/Security Errors (E3xx)

| Code | Message Template | Description |
|------|-----------------|-------------|
| E301 | `@requires references undefined policy '{policy}'` | Referencing non-existent policy |
| E302 | `Sensitive field '{field}' must be annotated with @hidden` | Sensitive field not marked @hidden |
| E303 | `Async operation inside @transaction is forbidden` | Async not allowed inside transactions |
| E304 | `Undefined layer '{name}' referenced in connect` | Referencing undefined layer |
| E305 | `Circular dependency: {path}` | Circular dependency |
| E306 | `Component '{name}' must have exactly one view block` | Incorrect number of component views |

#### Migration Errors (E4xx)

| Code | Message Template | Description |
|------|-----------------|-------------|
| E401 | `Migration version {ver} conflicts with existing` | Version conflict |
| E402 | `Migration chain broken: version {from} not found` | Broken migration chain |
| E403 | `Cannot drop column '{col}' — referenced by policy '{policy}'` | Column referenced by policy |

#### Hardware Constraint Errors (E5xx)

| Code | Message Template | Description |
|------|-----------------|-------------|
| E501 | `Target '{target}' does not support {feature}` | Target doesn't support this feature |
| E502 | `N_NEURONS={n} exceeds target maximum ({max})` | Neuron count exceeded |
| E503 | `BRAM usage {actual}Kb exceeds available {avail}Kb` | Insufficient BRAM resources |
| E504 | `Weight storage requires N_INPUTS × N_NEURONS ≤ {max}` | Weight storage exceeded |
| E505 | `energy_budget({budget}pJ) exceeded by estimated {actual}pJ` | Energy budget exceeded |

#### Warnings (Wxx)

| Code | Message Template | Description |
|------|-----------------|-------------|
| W201 | `API endpoint '{ep}' lacks @requires authorization` | API missing auth policy |
| W202 | `Migration version gap: {from} → {to}` | Non-contiguous migration versions |
| W301 | `Estimated cycle count {n} may cause timing issues` | FSM may timeout |
| W302 | `Enable mask leaves {n} neurons permanently disabled` | Too many neurons disabled |

### 9.3 Error Output Format

```
error[E202]: Unit incompatible: cannot add 'Time<ms>' and 'Energy<pJ>'
  --> src/example.yong:42:15
   |
42 |     let z = x + y;
   |               ^ cannot add 'Time<ms>' and 'Energy<pJ>'
   |
   = note: Time and Energy are different physical dimensions
   = help: consider using separate variables
```

---

## 10. Compiler Architecture

```
.yong source
    ↓
┌─────────────────┐
│    Frontend      │  Lexer/Parser (EBNF §3) → AST
└─────────────────┘
    ↓
┌─────────────────┐
│   Type Checker   │  Type inference + Unit checking + Security audit (§9)
└─────────────────┘
    ↓
┌─────────────────┐
│    YONG IR       │  Unified intermediate representation (graph + events + data flow)
└─────────────────┘
    ↓
┌─────────────────┐
│   Optimizer      │  Topology optimization / Energy optimization / Resource mapping
└─────────────────┘
    ↓
┌─────────────────┐
│  Conformance     │  Bit-accurate verification against §7 semantics
└─────────────────┘
    ↓
┌────────┬──────────┬──────────┬──────────┬──────────┐
│ Web App│ FPGA RTL │ ASIC RTL │ SNN Sim  │  Mobile  │
└────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 11. Conformance Test Requirements

> Any compiler backend must pass the following tests to claim "YongnianLang v4.2 Compatible."

### 11.1 Hardware Dialect

| Test ID | Description | Verification Method |
|---------|-------------|-------------------|
| HW-001 | STDP bit-accuracy | Same input + parameters → bit-for-bit identical weight matrix |
| HW-002 | WTA Top-K correctness | N neurons, multiple timestep rounds → winner sequence matches reference |
| HW-003 | Homeostasis convergence | After long run, all neuron firing rates within target_rate ±50% |
| HW-004 | Revival trigger | Artificially silence a neuron → must revive after revival_threshold steps |
| HW-005 | Weight clamping | Weights always within [w_min, w_max] |
| HW-006 | LIF membrane potential | Leak, integration, refractory behavior matches §7.1 definition |

### 11.2 App Dialect

| Test ID | Description | Verification Method |
|---------|-------------|-------------------|
| APP-001 | CRUD correctness | POST/GET/PUT/DELETE → data consistency |
| APP-002 | @hidden field hiding | GET responses exclude @hidden fields |
| APP-003 | @requires interception | No token / wrong role → 401/403 |
| APP-004 | @transaction rollback | assert failure → all changes rolled back |
| APP-005 | @migration idempotency | Same migration executed twice → no side effects |
| APP-006 | Event delivery ordering | emit delivered after COMMIT, not delivered after ROLLBACK |

---

## 12. Keyword Reference

### App Dialect

| Keyword | Purpose |
|---------|---------|
| `struct` | Data structure definition |
| `enum` | Enumeration type |
| `fn` | Function definition |
| `component` | UI component |
| `state` | Component state |
| `view` | View declaration |
| `policy` | Access policy |
| `migrate` | Data migration |
| `let` | Variable binding |
| `return` | Return value |
| `if` / `else` | Conditional branching |
| `for ... in` | Iteration |
| `assert` | Assertion (failure triggers rollback in transactions) |
| `emit` | Event emission |

### Hardware Dialect

| Keyword | Purpose |
|---------|---------|
| `network` | Network definition |
| `layer` | Layer definition |
| `connect` | Inter-layer connection |
| `config` | Configuration block |

### Decorators

| Decorator | Dialect | Purpose |
|-----------|---------|---------|
| `@db(table=...)` | App | Data persistence |
| `@api(METHOD, path)` | App | API endpoint |
| `@app(route=...)` | App | Page routing |
| `@auth(identity)` | App | Authentication subject |
| `@requires(Policy.action)` | App | Authorization check |
| `@transaction` | App | Transaction boundary |
| `@migration(version=N, from=M)` | App | Data migration |
| `@hidden` | App | Field hiding |
| `@unique` | App | Unique constraint |
| `@ui(widget)` | App | UI widget mapping |

---

*YongnianLang v4.2 Language Specification*  
*"Declare intent, compiler materializes."*  
*"声明意图，编译器物化。"*
