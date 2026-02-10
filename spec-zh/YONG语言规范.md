# YongnianLang 语言规范

> **版本**: v4.2  
> **更新**: 2026-02-10  
> **文件扩展名**: `.yong`

---

## ⚠️ 关于可扩展性 — 读本文档前请先了解

**YongnianLang 是一门开放的、正在演进的语言。**

本文档定义的是语言的当前状态，不是最终形态。YONG 从第一天起就被设计为可扩展的 — 你不需要等待官方更新，当你遇到本文档未覆盖的需求时，**自己扩展就行**。

### 不变的 (Frozen)

以下是语言的核心骨架，不会改变：

| 不变的东西 | 为什么不变 |
|-----------|-----------|
| **声明式哲学** — 写意图，不写实现 | 这是 YONG 存在的理由 |
| **装饰器语法** — `@name(args)` | 所有扩展都通过装饰器注入，这是统一入口 |
| **两大方言** — App + Hardware | 一个面向应用，一个面向硬件 |
| **数据优先** — struct 先行 | 数据结构是一切行为的基础 |
| **单位安全** — 编译期物理量检查 | 跨方言通用的类型安全保证 |
| **bit-accurate 语义** — 同源码同结果 | 同一份 .yong 在不同后端行为必须一致 |

### 开放的 (Extensible)

以下内容随时可以按需扩展，无需修改语言内核：

| 可扩展的东西 | 扩展方式 | 示例 |
|-------------|---------|------|
| **新装饰器** | 定义新的 `@name` | `@cache(ttl=60s)`, `@rate_limit(100/min)`, `@log` |
| **新类型** | 添加 BaseType 或 UnitType | `image`, `audio`, `tensor`, `Energy<eV>` |
| **新 config 键** | 在 `config {}` 块中添加 | `power_mode: low`, `clock: 100MHz`, `pipeline: deep` |
| **新学习规则** | 在 `connect ... with` 后添加 | `with r_stdp`, `with bcm`, `with oja` |
| **新神经元类型** | 在 `type=` 后添加 | `type=izhikevich`, `type=adex`, `type=hodgkin_huxley` |
| **新编译目标** | 编译器后端插件 | RISC-V RTL, WebAssembly, 量子电路 |
| **新 UI 控件** | `@ui(widget)` 的 widget 值 | `@ui(slider)`, `@ui(map)`, `@ui(chart)` |
| **新 policy 动作** | `allow/deny` 后的动词 | `allow export`, `allow share`, `allow archive` |
| **新迁移操作** | `migrate` 块内的指令 | `rename_table(...)`, `merge_columns(...)` |
| **新错误代码** | E/W + 新编号 | `E6xx` 安全审计, `E7xx` 性能约束 |

### 扩展原则

扩展 YONG 只需遵循三条规则：

1. **只声明，不实现** — 新关键字/装饰器也必须是声明式的，实现由编译器处理
2. **不破坏现有语法** — 扩展是加法，不是修改。已有的 `.yong` 文件在新版本中必须继续合法
3. **带单位的量必须有单位** — 新的物理量类型必须标注单位维度

```yong
// ✓ 好的扩展 — 声明式，不涉及实现细节
@cache(ttl=60s)
@api(GET, "/stats")
fn get_stats() -> Stats { ... }

// ✓ 好的扩展 — 新的学习规则，声明式
connect hidden -> output with bcm(threshold=0.5, rate=0.01);

// ✓ 好的扩展 — 新的神经元类型
layer cortex(1000, type=adex, adapt_tau=100ms);

// ✗ 坏的扩展 — 暴露实现细节，违反声明式原则
layer cortex(1000, type=lif, verilog_module="my_custom_lif.sv");
```

> **总结**: 本文档是 YONG 的「当前快照」。如果你需要的功能不在这里，按照核心原则自己加，不用等。

---

## 1. 概述

YongnianLang（永年语言，简称 YONG）是一门声明式编程语言。程序员描述意图，编译器负责物化到具体平台。

**核心原则**:

| 原则 | 含义 |
|------|------|
| **声明式** | 写「做什么」，不写「怎么做」 |
| **数据优先** | 先定义数据结构，再定义行为 |
| **单位安全** | 物理量自带单位，不兼容的单位无法混算 |
| **AI 友好** | 最少 token 表达最复杂的系统 |

### 1.1 Hello World

```yong
@app(route="/")
component Hello {
    view { Header("Hello, Yongnian!") }
}
```

### 1.2 30 行 Todo App

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

## 2. 两大方言

YONG 有两个应用方向，共享同一套语法哲学：

```
永年语言 (YongnianLang)
├── 🌐 App 方言     — @ui, @api, @db, component, @auth, @migration
│   └── 编译到: Web App / Mobile App / API Server
│
└── 🧠 Hardware 方言 — network, layer, connect, config
    └── 编译到: FPGA RTL / ASIC RTL / SNN 仿真器
```

**共同设计**:
1. **装饰器驱动** — `@ui`, `@api`, `config {}` 都是编译器指令
2. **无实现细节** — 没有 `<div>`，没有 `always @(posedge clk)`
3. **类型安全** — 编译期捕获类型错误和单位不兼容

---

## 3. 正式文法 (EBNF)

> 编译器前端、IDE 插件、AI Agent 都必须以此为准。

### 3.1 顶层结构

```ebnf
(* ========== 顶层 ========== *)
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

(* ========== 通用装饰器 ========== *)
Annotation      = "@" IDENT [ "(" AnnotationArgs ")" ] ;
AnnotationArgs  = AnnotationArg { "," AnnotationArg } ;
AnnotationArg   = STRING | NUMBER | IDENT | KeyValue ;
KeyValue        = IDENT "=" ( STRING | NUMBER | IDENT | "true" | "false" ) ;
```

### 3.2 App 方言文法

```ebnf
(* ========== 数据结构 ========== *)
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

(* ========== 函数 ========== *)
FnDecl          = { Annotation } "fn" IDENT "(" [ ParamList ] ")" [ "->" TypeExpr ] Block ;
ParamList       = Param { "," Param } ;
Param           = IDENT ":" TypeExpr ;

(* ========== 组件 ========== *)
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

(* ========== 权限策略 ========== *)
PolicyDecl      = "policy" IDENT "{" PolicyBody "}" ;
PolicyBody      = ResourceBind SubjectBind { PolicyRule } ;
ResourceBind    = "resource" ":" IDENT ;
SubjectBind     = "subject" ":" IDENT ;
PolicyRule      = ( "allow" | "deny" ) PolicyAction "when" Expr
                | ( "allow" | "deny" ) "*" ;
PolicyAction    = "read" | "create" | "update" | "delete" | "*" ;

(* ========== 数据迁移 ========== *)
MigrationDecl   = "@migration" "(" MigrationArgs ")" "migrate" IDENT Block ;
MigrationArgs   = "version" "=" NUMBER "," "from" "=" NUMBER ;
```

### 3.3 Hardware 方言文法

```ebnf
(* ========== 网络 ========== *)
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

### 3.4 表达式与语句

```ebnf
(* ========== 语句 ========== *)
Block           = "{" { Statement } "}" ;
Statement       = LetStmt | ReturnStmt | IfStmt | ForStmt | AssertStmt | EmitStmt | ExprStmt ;
LetStmt         = "let" IDENT [ ":" TypeExpr ] "=" Expr ";" ;
ReturnStmt      = "return" Expr ";" ;
IfStmt          = "if" "(" Expr ")" Block [ "else" Block ] ;
ForStmt         = "for" IDENT "in" Expr Block ;
AssertStmt      = "assert" "(" Expr "," STRING ")" ";" ;
EmitStmt        = "emit" IDENT "(" [ ArgList ] ")" ";" ;
ExprStmt        = Expr ";" ;

(* ========== 表达式 (优先级从低到高) ========== *)
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

(* ========== 词法 ========== *)
IDENT           = LETTER { LETTER | DIGIT | "_" } ;
NUMBER          = DIGIT { DIGIT } [ "." DIGIT { DIGIT } ] ;
STRING          = '"' { CHAR } '"' ;
LETTER          = "a".."z" | "A".."Z" | "_" ;
DIGIT           = "0".."9" ;
HEX_DIGIT       = DIGIT | "a".."f" | "A".."F" ;

(* ========== 注释 ========== *)
LineComment     = "//" { CHAR } NEWLINE ;
BlockComment    = "/*" { CHAR } "*/" ;
```

---

## 4. 类型系统

### 4.1 基础类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `int` | 整数 | `42` |
| `float` | 浮点数 | `3.14` |
| `bool` | 布尔 | `true`, `false` |
| `string` | 字符串 | `"hello"` |
| `uid` | 唯一标识（UUID） | 自动生成 |
| `time` | 时间戳 | `created_at: time` |
| `money` | 货币金额 | `balance: money` |
| `email` | 电子邮件 | 编译器自动附加格式校验 |
| `url` | URL 地址 | 编译器自动附加格式校验 |
| `void` | 无返回值 | `fn foo() -> void` |

### 4.2 复合类型

```yong
list[T]                   // 有序集合
map[K, V]                 // 键值映射
T?                        // 可空类型（使用前必须 null-check）
```

### 4.3 物理单位类型

```yong
// 时间
Time<ms>    Time<us>    Time<ns>    Time<s>

// 能量
Energy<pJ>  Energy<nJ>  Energy<fJ>  Energy<J>

// 功率
Power<mW>   Power<W>

// 电压
Voltage<mV> Voltage<V>

// 频率
Frequency<Hz>   Frequency<kHz>  Frequency<MHz>
```

### 4.4 单位安全

不兼容的物理单位在编译期报错：

```yong
let x: Time<ms> = 5ms;
let y: Energy<pJ> = 10pJ;
let z = x + y;      // ✗ E202: Unit incompatible: cannot add 'Time<ms>' and 'Energy<pJ>'
```

同维度单位允许隐式转换：

```yong
let a: Time<ms> = 5ms;
let b: Time<us> = 3000us;
let c = a + b;       // ✓ 结果: Time<ms> = 8ms (编译器自动转换)
```

### 4.5 字段注解

| 注解 | 含义 |
|------|------|
| `@unique` | 字段值唯一约束 |
| `@hidden` | 不序列化到 API 响应（用于密码等敏感字段） |
| `@ui(widget)` | 指定 UI 控件类型 |

---

## 5. App 方言

### 5.1 数据定义

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

`@db(table=...)` 声明数据持久化。编译器自动生成数据库表、序列化和 ORM 映射。

### 5.2 API 定义

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

### 5.3 组件定义

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

### 5.4 管道操作符

```yong
users
    |> filter(u => u.active)
    |> map(u => u.name)
    |> sort()
```

### 5.5 认证与权限

```yong
// -- 用户模型 --
@db(table="users")
@auth(identity)                        // 标记为认证主体
struct User {
    id: uid
    email: email @unique
    password_hash: string @hidden       // @hidden = 不出现在 API 响应中
    role: Role
    created_at: time
}

enum Role { admin, editor, viewer }

// -- 认证配置 --
config auth {
    provider: jwt                       // jwt | session | oauth2
    secret_env: "JWT_SECRET"            // 从环境变量读取密钥
    token_expiry: 24h
    refresh_enabled: true
    password_hash: bcrypt(rounds=12)
}

// -- 权限策略 --
policy TodoPolicy {
    resource: Todo
    subject:  User

    allow read   when subject.role in [admin, editor, viewer]
    allow create when subject.role in [admin, editor]
    allow update when subject.role == admin
                   or resource.owner_id == subject.id
    allow delete when subject.role == admin

    deny *                              // 默认拒绝
}

// -- 在 API 上应用权限 --
@api(DELETE, "/todos/:id")
@requires(TodoPolicy.delete)            // 编译器注入鉴权中间件
fn delete_todo(id: uid) -> void {
    db.todos.delete(id)
}
```

编译器行为：
- `@auth(identity)` → 生成 JWT 签发/验证逻辑
- `policy` → 生成 RBAC 检查中间件
- `@requires` → 自动注入到 API 端点
- `@hidden` → 从所有 GET 响应中剥离该字段
- 缺少 `@requires` 的 `@api` → 编译警告 W201

### 5.6 数据迁移

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

编译器行为：
- 生成 SQL ALTER TABLE（或目标数据库等效语句）
- 自动生成回滚脚本（down migration）
- 维护 `_yong_migrations` 表追踪版本
- 启动时自动检查并执行未应用的迁移
- 版本冲突 → 编译错误 E401

### 5.7 事务

```yong
@api(POST, "/transfer")
@requires(TransferPolicy.execute)
@transaction
fn transfer(from_id: uid, to_id: uid, amount: money) -> void {
    let from = db.accounts.get(from_id)
    let to   = db.accounts.get(to_id)

    assert(from.balance >= amount, "Insufficient funds")  // 失败 → 自动 ROLLBACK

    db.accounts.update(from_id, { balance: from.balance - amount })
    db.accounts.update(to_id,   { balance: to.balance + amount })

    emit TransferCompleted(from_id, to_id, amount)        // COMMIT 后才投递
}
```

**事务语义**:

| 行为 | 规则 |
|------|------|
| 作用域 | `@transaction` 函数体 = 一个事务边界 |
| 隔离级别 | 默认 `READ_COMMITTED`，可通过 `@transaction(isolation=serializable)` 覆盖 |
| 失败回滚 | `assert` 失败或未捕获异常 → 自动 `ROLLBACK` |
| 事件投递 | `emit` 的事件在 `COMMIT` 成功后才投递（outbox 模式） |
| 嵌套 | 默认使用 `SAVEPOINT`；`@transaction(nested=false)` 禁止嵌套 |

### 5.8 事件安全约束

所有 `on_*` 事件处理器（`on_enter`, `on_click`, `on_change`）最终调用的 `@api` 函数，必须满足 `@requires` 权限检查。编译器在编译期静态验证调用链上的权限完备性。

---

## 6. Hardware 方言

### 6.1 网络声明

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

### 6.2 关键字

| 关键字 | 含义 |
|--------|------|
| `layer name(N)` | 定义一个 N 宽度的输入层 |
| `layer name(N, type=lif, ...)` | 定义 N 个 LIF 神经元层，可配置阈值和泄漏率 |
| `connect A -> B with stdp` | A 到 B 的全连接，启用 STDP 学习 |
| `connect A -> B with none` | A 到 B 的全连接，固定权重 |
| `weight_storage: bram` | 权重存储在 Block RAM（大规模网络） |
| `weight_storage: lut` | 权重存储在寄存器（小规模网络，默认） |
| `wta_mode: enable` | 启用 Winner-Take-All 竞争机制 |
| `homeostasis: enable` | 启用自适应阈值调节 |
| `energy_tracking: enable` | 启用脉冲计数 / 能耗统计 |
| `threshold_mode: dual` | 分离训练阈值和推理阈值 |

### 6.3 STDP 参数配置

```yong
connect input -> neurons with stdp(
    ltp=8,              // LTP 增量 (默认=8, 范围 0-255)
    ltd=3,              // LTD 增量 (默认=3, 范围 0-255)
    w_min=30,           // 权重下界 (默认=30)
    w_max=250,          // 权重上界 (默认=250)
    div=3               // 分频 (默认=3 → 每4步学习一次, 0=每步学习)
);
```

### 6.4 Homeostasis 参数配置

```yong
config {
    homeostasis: enable
    target_rate: 20         // 目标发放率 (默认=20)
    revival_threshold: 50   // 连续未赢此步数后复活 (默认=50)
    w_sum_max: 220          // 权重总和缩放阈值 (默认=220)
}
```

### 6.5 多层网络示例

```yong
network DeepSNN {
    layer input(784);                                   // 28×28 图像输入
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

## 7. 可执行语义定义

> 以下定义是规范性的（normative）。同一份 `.yong` 文件编译到任何后端，
> 必须产生一致的行为。伪代码使用 Python 风格描述。

### 7.1 LIF 神经元模型

**适用**: `layer name(N, type=lif, threshold=T, leak=L)`

```python
class LIFNeuron:
    """
    Leaky Integrate-and-Fire 神经元。定点算术，16-bit 膜电位。

    参数:
      threshold:  基础阈值 (默认=200)
      leak:       每 timestep 泄漏量 (默认=2)
      refrac:     不应期长度 (默认=3 timesteps)
    """

    def __init__(self, threshold=200, leak=2, refrac=3):
        self.membrane = 0           # uint16, 膜电位
        self.refrac_cnt = 0         # uint16, 不应期计数器
        self.spike_count = 0        # uint16, 脉冲累计计数
        self.spike_window = 0       # uint16, 当前窗口脉冲计数 (homeostasis)
        self.thresh_adj = 500       # uint16, 阈值偏移 (homeostasis, 中心值=500)
        self.no_win_cnt = 0         # uint8,  连续未赢计数 (WTA revival)

    def integrate(self, weighted_sum, leak):
        """积分阶段: 累加输入电流，减去泄漏。"""
        if self.membrane + weighted_sum > leak:
            self.membrane = self.membrane + weighted_sum - leak
        else:
            self.membrane = 0

    def fire(self, eff_threshold, is_wta_winner) -> bool:
        """放电判定。"""
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
                # WTA loser: 不放电，膜电位减半（软抑制）
                self.membrane >>= 1
                return False
        return False
```

### 7.2 STDP 学习规则

**适用**: `connect A -> B with stdp`

#### 7.2.1 触发条件

```python
def should_run_stdp(learn_en, div_counter, has_winner):
    """
    STDP 执行的三个前提:
      1. learn_en == True   (学习使能)
      2. div_counter == 0   (分频到期)
      3. has_winner == True  (WTA 产生了至少一个赢家)
    """
    return learn_en and div_counter == 0 and has_winner
```

#### 7.2.2 权重更新公式

```python
def stdp_update(weight: uint8, input_active: bool,
                ltp=8, ltd=3, w_min=30, w_max=250) -> uint8:
    """
    单个突触权重更新 (Winner-Take-All STDP 变体)。

    只有 WTA 赢家的权重被更新。
    更新方向由「输入是否活跃」决定。

      - 输入活跃 → LTP (长时程增强): weight += ltp
      - 输入沉默 → LTD (长时程抑制): weight -= ltd

    结果钳位在 [w_min, w_max] 范围内。
    """
    if input_active:
        new_weight = min(weight + ltp, w_max)
    else:
        new_weight = max(weight - ltd, w_min)

    return clamp(new_weight, w_min, w_max)
```

#### 7.2.3 分频控制

```python
class STDPDivider:
    """
    分频器控制 STDP 执行频率，避免权重振荡。

    div_max=0 → 每个 timestep 都学习
    div_max=3 → 每 4 个 timestep 学习一次 (默认)
    """
    def __init__(self, div_max=3):
        self.counter = 0
        self.div_max = div_max

    def tick(self) -> bool:
        """每个 timestep 调用一次。返回 True 表示本次应执行 STDP。"""
        if self.counter == 0:
            self.counter = self.div_max
            return True
        else:
            self.counter -= 1
            return False
```

#### 7.2.4 一致性要求

> **Bit-accuracy**: 任何后端（FPGA / ASIC / 仿真器）对相同的输入序列和相同的 STDP 参数，
> 必须产生完全相同的权重矩阵。不允许浮点近似。所有运算使用定点整数。

### 7.3 WTA (Winner-Take-All)

**适用**: `wta_mode: enable`

#### 7.3.1 选择算法

```python
def wta_select(membranes, N, K=2, lfsr_noise, start_idx=0):
    """
    从 N 个神经元中选出 Top-K (默认 K=2)。

    参数:
      membranes:  各神经元膜电位数组
      N:          神经元总数
      K:          赢家数目 (默认=2, 即冠军+亚军都可放电)
      lfsr_noise: 随机数生成器输出 (用于打破并列)
      start_idx:  扫描起始索引 (每次有 spike 后旋转 +1, 防止饥饿)

    返回: 赢家索引列表 (长度=K)
    """
    winners = []  # (value, index) 按 value 降序，容量=K

    for offset in range(N):
        i = (start_idx + offset) % N

        if neuron_disabled(i) or refrac_cnt[i] > 0:
            continue

        noisy_val = membranes[i] + (lfsr_noise & 0xF)  # 低 4-bit 噪声

        # 插入到有序 winners 列表中
        insert_into_topk(winners, (noisy_val, i), K)

    return [idx for (val, idx) in winners]
```

#### 7.3.2 并列裁决

并列通过以下机制打破（优先级从高到低）：

1. **LFSR 噪声注入**: `noisy_val = membrane + lfsr[3:0]`，即使膜电位相同，噪声大概率打破并列
2. **起始旋转 (Anti-Starvation)**: `start_idx` 每次有 spike 后 +1，保证长期公平
3. **确定性裁决**: 如前两项均未打破并列，扫描顺序中先遇到的 neuron 优先

#### 7.3.3 赢家税 (Winner Tax)

```python
def apply_winner_tax(winner_idx, thresh_adj, win_tax=5):
    """
    防止赢家垄断。每次 neuron 赢得 WTA 时上调其阈值。

    参数:
      win_tax: 税值系数 (默认=5)
      实际增量 = win_tax × 4
    """
    delta = win_tax << 2                           # 默认 5×4=20
    thresh_adj[winner_idx] = min(
        thresh_adj[winner_idx] + delta,
        60000                                      # 饱和上界
    )
```

### 7.4 Homeostasis

**适用**: `homeostasis: enable`

#### 7.4.1 执行时序

```
每个 timestep 的处理阶段:

  INTEGRATE → WTA → FIRE → LEARN(STDP) → HOMEOSTASIS → WEIGHT_SCALE
                                          ↑
                                       在此执行
```

#### 7.4.2 阈值调节

```python
def homeostasis_update(neuron_idx, spike_window, thresh_adj, target_rate=20):
    """
    独立对每个 neuron 执行。

    规则:
      - 发放 > target_rate → 提高阈值 (+4, 抑制)
      - 发放 < target_rate/2 → 降低阈值 (-8, 激励)

    不对称性说明:
      降阈值步长 (8) > 升阈值步长 (4)。
      设计意图: 激活沉默 neuron 比抑制活跃 neuron 更重要。
    """
    if target_rate == 0:
        return

    if spike_window[neuron_idx] > target_rate:
        thresh_adj[neuron_idx] = min(thresh_adj[neuron_idx] + 4, 60000)
    elif spike_window[neuron_idx] < (target_rate >> 1):
        thresh_adj[neuron_idx] = max(thresh_adj[neuron_idx] - 8, 8)

    spike_window[neuron_idx] = 0  # 重置窗口
```

#### 7.4.3 有效阈值计算

```python
def get_effective_threshold(thresh_adj, base_threshold):
    """
    thresh_adj 以 500 为中心:
      adj > 500 → 有效阈值高于 base (抑制)
      adj < 500 → 有效阈值低于 base (激励)
      adj = 500 → 有效阈值 = base
    """
    if thresh_adj >= 500:
        return base_threshold + (thresh_adj - 500)
    else:
        delta = 500 - thresh_adj
        return max(base_threshold - delta, 1)  # 最低阈值 = 1
```

#### 7.4.4 死神经元复活 (Revival)

```python
def revival_check(neuron_idx, no_win_cnt, thresh_adj, is_winner):
    """
    WTA 模式下，连续 revival_threshold 步未赢的 neuron 被判定为「死亡」并复活。

    复活机制: 大幅降低 thresh_adj → 下一轮 WTA 极易被激活。
    """
    if is_winner:
        no_win_cnt[neuron_idx] = 0
        return

    no_win_cnt[neuron_idx] = min(no_win_cnt[neuron_idx] + 1, 255)

    if no_win_cnt[neuron_idx] >= REVIVAL_THRESHOLD:   # 默认=50
        no_win_cnt[neuron_idx] = 0
        thresh_adj[neuron_idx] = 200                  # 远低于中心值 500 → 易激活
```

#### 7.4.5 权重缩放

```python
def weight_scaling(neuron_idx, weights, N_INPUTS, w_sum_max=220):
    """
    防止总权重无限增长。

    触发: sum(weights) > w_sum_max × 64
    动作: 所有权重右移 4 位 (÷16)
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

## 8. 编译器规则

> 以下规则是编译器实现者必须遵守的约束。

### 8.1 核心原则

```
程序员写:     weight_storage: bram
编译器生成:   Block RAM 实例化 + 读写流水线 + 地址生成

程序员写:     @api(POST, "/todos")
编译器生成:   路由注册 + 中间件 + 错误处理 + 序列化

程序员写:     connect input -> neurons with stdp
编译器生成:   所有 STDP 逻辑 + 权重钳位 + 分频控制
```

### 8.2 Hardware 方言规则

| 规则 | 说明 |
|------|------|
| BRAM 读延迟 | 当 `weight_storage: bram` 时，编译器必须处理至少 2 clock 的读延迟 |
| 索引位宽 | `neuron_idx = ceil(log2(N))`, `input_idx = ceil(log2(N_INPUTS)) + 1` |
| 资源适配 | 编译器根据 `target` 自动选择并行或串行实现 |
| 寄存器映射 | 编译器自动生成控制/状态寄存器及其总线接口 |

### 8.3 App 方言规则

| 规则 | 说明 |
|------|------|
| 权限显式 | 没有 `@requires` 的 `@api` → 警告 W201 |
| 敏感字段 | 密码类字段没有 `@hidden` → 错误 E302 |
| 事务纯净 | `@transaction` 内不允许异步操作 → 错误 E303 |
| 策略引用 | `@requires` 引用不存在的 `policy` → 错误 E301 |

---

## 9. 编译错误与警告

### 9.1 错误代码体系

```
格式: [类别字母][3位数字]

E = Error   (编译终止)
W = Warning (可继续)

  E1xx — 语法错误 (Syntax)
  E2xx — 类型错误 (Type)
  E3xx — 语义/安全错误 (Semantic/Security)
  E4xx — 迁移/数据错误 (Data)
  E5xx — 硬件约束错误 (Hardware)
  W1xx — 语法警告
  W2xx — 安全警告
  W3xx — 性能警告
```

### 9.2 错误目录

#### 语法错误 (E1xx)

| 代码 | 消息模板 | 描述 |
|------|---------|------|
| E101 | `Unexpected token '{token}' at {line}:{col}` | 不符合 EBNF 的 token |
| E102 | `Missing ';' after statement at {line}:{col}` | 缺少分号 |
| E103 | `Undefined keyword '{keyword}'` | 无效关键字 |
| E104 | `Duplicate field '{name}' in struct '{struct}'` | 结构体重复字段 |
| E105 | `Invalid annotation '@{name}' for target '{target}'` | 装饰器不适用于当前上下文 |
| E106 | `Unmatched brace at {line}:{col}` | 括号不匹配 |

#### 类型错误 (E2xx)

| 代码 | 消息模板 | 描述 |
|------|---------|------|
| E201 | `Type mismatch: cannot assign '{rhs}' to '{lhs}'` | 类型不兼容 |
| E202 | `Unit incompatible: cannot add '{unit_a}' and '{unit_b}'` | 物理单位不兼容 |
| E203 | `Type '{type}' is not iterable` | 对非集合类型使用 List/for |
| E204 | `Unknown type '{type}'` | 未定义的类型 |
| E205 | `Dimension mismatch: layer '{layer}' expects {expected}, got {actual}` | 层间维度不匹配 |
| E206 | `Nullable type '{type}?' requires null-check before use` | 空值安全检查缺失 |

#### 语义/安全错误 (E3xx)

| 代码 | 消息模板 | 描述 |
|------|---------|------|
| E301 | `@requires references undefined policy '{policy}'` | 引用不存在的策略 |
| E302 | `Sensitive field '{field}' must be annotated with @hidden` | 敏感字段未标 @hidden |
| E303 | `Async operation inside @transaction is forbidden` | 事务内不允许异步 |
| E304 | `Undefined layer '{name}' referenced in connect` | 引用未定义的层 |
| E305 | `Circular dependency: {path}` | 循环依赖 |
| E306 | `Component '{name}' must have exactly one view block` | 组件 view 数量错误 |

#### 迁移错误 (E4xx)

| 代码 | 消息模板 | 描述 |
|------|---------|------|
| E401 | `Migration version {ver} conflicts with existing` | 版本冲突 |
| E402 | `Migration chain broken: version {from} not found` | 迁移链断裂 |
| E403 | `Cannot drop column '{col}' — referenced by policy '{policy}'` | 被策略引用的列 |

#### 硬件约束错误 (E5xx)

| 代码 | 消息模板 | 描述 |
|------|---------|------|
| E501 | `Target '{target}' does not support {feature}` | 目标不支持此特性 |
| E502 | `N_NEURONS={n} exceeds target maximum ({max})` | 神经元数超限 |
| E503 | `BRAM usage {actual}Kb exceeds available {avail}Kb` | BRAM 资源不足 |
| E504 | `Weight storage requires N_INPUTS × N_NEURONS ≤ {max}` | 权重存储超限 |
| E505 | `energy_budget({budget}pJ) exceeded by estimated {actual}pJ` | 能量预算超限 |

#### 警告 (Wxx)

| 代码 | 消息模板 | 描述 |
|------|---------|------|
| W201 | `API endpoint '{ep}' lacks @requires authorization` | API 缺少权限策略 |
| W202 | `Migration version gap: {from} → {to}` | 迁移版本不连续 |
| W301 | `Estimated cycle count {n} may cause timing issues` | FSM 可能超时 |
| W302 | `Enable mask leaves {n} neurons permanently disabled` | 禁用了过多 neuron |

### 9.3 错误输出格式

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

## 10. 编译器架构

```
.yong 源码
    ↓
┌─────────────────┐
│    Frontend      │  词法/语法分析 (EBNF §3) → AST
└─────────────────┘
    ↓
┌─────────────────┐
│   Type Checker   │  类型推导 + 单位检查 + 安全审计 (§9)
└─────────────────┘
    ↓
┌─────────────────┐
│    YONG IR       │  统一中间表示 (图 + 事件 + 数据流)
└─────────────────┘
    ↓
┌─────────────────┐
│   Optimizer      │  拓扑优化 / 能量优化 / 资源映射
└─────────────────┘
    ↓
┌─────────────────┐
│  Conformance     │  对 §7 语义做 bit-accurate 验证
└─────────────────┘
    ↓
┌────────┬──────────┬──────────┬──────────┬──────────┐
│ Web App│ FPGA RTL │ ASIC RTL │ SNN Sim  │  Mobile  │
└────────┴──────────┴──────────┴──────────┴──────────┘
```

---

## 11. Conformance Test 要求

> 任何编译器后端必须通过以下测试才能声称 "YongnianLang v4.1 Compatible"。

### 11.1 Hardware 方言

| 测试 ID | 描述 | 验证方法 |
|---------|------|---------|
| HW-001 | STDP bit-accuracy | 相同输入 + 参数 → 权重矩阵 bit-for-bit 相同 |
| HW-002 | WTA Top-K 正确性 | N 个 neuron, 多轮 timestep → winner 序列匹配参考实现 |
| HW-003 | Homeostasis 收敛 | 长时运行后所有 neuron 发放率在 target_rate ±50% |
| HW-004 | Revival 触发 | 人为沉默一个 neuron → revival_threshold 步后必须复活 |
| HW-005 | Weight clamping | 权重永远在 [w_min, w_max] 内 |
| HW-006 | LIF 膜电位 | 泄漏、积分、不应期行为匹配 §7.1 定义 |

### 11.2 App 方言

| 测试 ID | 描述 | 验证方法 |
|---------|------|---------|
| APP-001 | CRUD 正确性 | POST/GET/PUT/DELETE → 数据一致 |
| APP-002 | @hidden 字段隐藏 | GET 响应中不包含 @hidden 字段 |
| APP-003 | @requires 拦截 | 无 token / 错误角色 → 401/403 |
| APP-004 | @transaction 回滚 | assert 失败 → 所有变更回滚 |
| APP-005 | @migration 幂等性 | 同一迁移执行两次 → 无副作用 |
| APP-006 | 事件投递顺序 | emit 在 COMMIT 后投递, ROLLBACK 后不投递 |

---

## 12. 关键字一览

### App 方言

| 关键字 | 用途 |
|--------|------|
| `struct` | 数据结构定义 |
| `enum` | 枚举类型 |
| `fn` | 函数定义 |
| `component` | UI 组件 |
| `state` | 组件状态 |
| `view` | 视图声明 |
| `policy` | 权限策略 |
| `migrate` | 数据迁移 |
| `let` | 变量绑定 |
| `return` | 返回值 |
| `if` / `else` | 条件分支 |
| `for ... in` | 迭代 |
| `assert` | 断言（事务中失败触发回滚） |
| `emit` | 事件发射 |

### Hardware 方言

| 关键字 | 用途 |
|--------|------|
| `network` | 网络定义 |
| `layer` | 层定义 |
| `connect` | 层间连接 |
| `config` | 配置块 |

### 装饰器

| 装饰器 | 方言 | 用途 |
|--------|------|------|
| `@db(table=...)` | App | 数据持久化 |
| `@api(METHOD, path)` | App | API 端点 |
| `@app(route=...)` | App | 页面路由 |
| `@auth(identity)` | App | 认证主体 |
| `@requires(Policy.action)` | App | 权限检查 |
| `@transaction` | App | 事务边界 |
| `@migration(version=N, from=M)` | App | 数据迁移 |
| `@hidden` | App | 字段隐藏 |
| `@unique` | App | 唯一约束 |
| `@ui(widget)` | App | UI 控件映射 |

---

*YongnianLang v4.1 语言规范*  
*"声明意图，编译器物化。"*  
*"Declare intent, compiler materializes."*
