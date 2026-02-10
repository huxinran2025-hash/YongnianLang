# YONG 编译器规范

> **版本**: v1.0  
> **更新**: 2026-02-10  
> **依赖**: [YongnianLang 语言规范 v4.2](./VGO语言规范.md)

---

## ⚠️ 这是架构规范，不是具体实现

本文档定义 YONG 编译器的**骨架和扩展点**。它告诉你编译器应该长什么样、数据怎么流动、从哪里扩展。   
它**不是**某个具体编译器的源码文档。

和语言规范一样，编译器也是可扩展的 — 核心管线不变，后端和插件随意加。

---

## 1. 核心管线

```
 .yong 源码
     │
     ▼
┌──────────┐
│ Stage 1  │  Tokenizer — 源码 → Token 流
│  词法    │  输入: 字符串    输出: List[Token]
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 2  │  Parser — Token 流 → AST
│  语法    │  输入: List[Token]    输出: AST (无类型)
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 3  │  Analyzer — AST → Typed AST
│  分析    │  类型推导 + 单位检查 + 权限审计 + 错误报告
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 4  │  IR Generator — Typed AST → YONG IR
│  中间    │  统一中间表示 (跨后端共用)
└──────────┘
     │
     ▼
┌──────────┐
│ Stage 5  │  Backend — YONG IR → 目标代码
│  后端    │  可插拔: 每个目标一个 Backend 插件
└──────────┘
     │
     ├──→  Web App (HTML/CSS/JS + API Server)
     ├──→  FPGA RTL (SystemVerilog)
     ├──→  ASIC RTL (Verilog)
     ├──→  SNN Simulator (Python)
     ├──→  Mobile (React Native / Flutter)
     └──→  ... (你自己的后端)
```

### 不变的 (Frozen)

| 不变 | 理由 |
|------|------|
| 5 级管线结构 | 每级职责清晰，级间通过标准数据结构传递 |
| 级间数据格式 | Token / AST / Typed AST / IR 的接口定义不变 |
| 单向数据流 | 数据只从 Stage 1 流向 Stage 5，不回流 |
| 错误在 Stage 3 之前全部报完 | Backend 不做语法和类型检查 |

### 可扩展的 (Extensible)

| 扩展点 | 方式 |
|--------|------|
| 新的 Backend | 注册一个 `Backend` 插件 |
| 新的装饰器 | 注册一个 `DecoratorHandler` |
| 新的类型 | 注册到 `TypeRegistry` |
| 新的学习规则 | 注册到 `LearningRuleRegistry` |
| 新的神经元类型 | 注册到 `NeuronTypeRegistry` |
| 新的 Lint / 审计规则 | 注册到 `AnalyzerPass` 列表 |
| 新的 IR 优化 | 注册到 `IRPass` 列表 |

---

## 2. Stage 1: Tokenizer

### 2.1 职责

将 `.yong` 源码字符串拆分成 Token 序列。

### 2.2 Token 结构

```
Token {
    type:    TokenType       // 分类 (关键字/标识符/字面量/符号)
    value:   string          // 原始文本
    line:    int             // 行号 (1-indexed)
    column:  int             // 列号 (1-indexed)
    span:    (start, end)    // 字符偏移量 (用于错误指示)
}
```

### 2.3 Token 分类

```
关键字 (Keywords):
  App:      struct, enum, fn, component, state, view, policy,
            migrate, let, return, if, else, for, in, assert, emit,
            import, as, allow, deny, when, on, effect
  Hardware: network, layer, connect, config, with, enable, disable
  共用:     true, false

标识符 (Identifiers):
  /[a-zA-Z_][a-zA-Z0-9_]*/

字面量 (Literals):
  NUMBER:       /[0-9]+(\.[0-9]+)?/
  STRING:       /"[^"]*"/
  HEX:          /0x[0-9a-fA-F_]+/
  UNIT_LITERAL: NUMBER + UNIT   (例: 5ms, 10pJ, 100MHz)

符号 (Symbols):
  结构: { } ( ) [ ]
  运算: + - * / % == != < > <= >= || && ! |> ? :
  标点: ; , . -> = @

注释 (Comments):
  行注释: // ...
  块注释: /* ... */
```

### 2.4 扩展点

**新关键字**: 添加到关键字表即可。Tokenizer 不需要知道关键字的语义。

---

## 3. Stage 2: Parser

### 3.1 职责

将 Token 序列解析为 AST（抽象语法树）。语法规则遵循 [语言规范 §3 EBNF](./VGO语言规范.md)。

### 3.2 AST 节点类型

```
AST (抽象语法树)
├── Program
│   └── declarations: List[Declaration]
│
├── Declaration (声明)
│   ├── ImportDecl        { path, alias? }
│   ├── StructDecl        { name, annotations[], fields[] }
│   ├── EnumDecl          { name, variants[] }
│   ├── FnDecl            { name, annotations[], params[], return_type?, body }
│   ├── ComponentDecl     { name, annotations[], states[], views[], handlers[] }
│   ├── NetworkDecl       { name, layers[], connections[], config? }
│   ├── PolicyDecl        { name, resource, subject, rules[] }
│   └── MigrationDecl     { version, from, name, body }
│
├── Statement (语句)
│   ├── LetStmt           { name, type?, value }
│   ├── ReturnStmt        { value }
│   ├── IfStmt            { condition, then_block, else_block? }
│   ├── ForStmt           { variable, iterable, body }
│   ├── AssertStmt        { condition, message }
│   ├── EmitStmt          { event_name, args[] }
│   └── ExprStmt          { expr }
│
├── Expression (表达式)
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
├── Hardware-specific (硬件专用)
│   ├── LayerNode         { name, size, neuron_type?, params{} }
│   ├── ConnectNode       { source, target, learning_rule, params{} }
│   └── ConfigNode        { entries{} }
│
├── Annotation (装饰器)
│   └── AnnotationNode    { name, args[] }
│
└── Type (类型)
    ├── SimpleType        { name }                // int, string, uid
    ├── GenericType       { name, type_args[] }   // list[Todo]
    ├── NullableType      { inner }               // string?
    └── UnitType          { base, unit }          // Time<ms>
```

### 3.3 Parser 的唯一职责

Parser **只管语法结构**，不做任何语义判断：
- ✅ `@foo(bar=baz)` → 解析为 `AnnotationNode("foo", [("bar", "baz")])`
- ✅ `connect A -> B with xyz` → 解析为 `ConnectNode("A", "B", "xyz")`
- ❌ 不检查 `@foo` 是否是有效装饰器
- ❌ 不检查 `xyz` 是否是有效学习规则
- ❌ 不检查 `A` 和 `B` 是否已定义

这些都是 Stage 3 (Analyzer) 的事。

### 3.4 错误恢复

Parser 遇到语法错误时应该：
1. 记录错误（位置 + 消息）
2. 同步到下一个安全点（`;` 或 `}`）
3. 继续解析
4. 一次性报告所有语法错误（而不是遇到第一个就停）

---

## 4. Stage 3: Analyzer

### 4.1 职责

对 AST 做语义分析，产出 Typed AST。**所有错误必须在这里报完。**

### 4.2 分析 Pass

Analyzer 由一系列有序的 Pass 组成。每个 Pass 做一件事：

```
Pass 1: SymbolResolution     — 建立符号表，检查 undefined 引用
Pass 2: TypeInference        — 推导表达式类型
Pass 3: UnitCheck            — 物理单位兼容性检查
Pass 4: SecurityAudit        — @auth/@requires/@hidden 完备性检查
Pass 5: HardwareConstraint   — 神经元数、BRAM 容量、目标兼容性检查
Pass 6: MigrationValidation  — 迁移版本链完整性检查
```

### 4.3 符号表

```
SymbolTable {
    // 作用域栈
    scopes: Stack[Scope]

    // 全局注册表
    types:          Map[string, TypeDef]       // 所有已知类型
    functions:      Map[string, FnDef]         // 所有函数签名
    structs:        Map[string, StructDef]     // 所有结构体
    policies:       Map[string, PolicyDef]     // 所有权限策略
    layers:         Map[string, LayerDef]      // 所有网络层 (Hardware)
    decorators:     Map[string, DecoratorDef]  // 所有已知装饰器

    // 方法
    resolve(name) -> Symbol | Error
    define(name, symbol) -> void | Error
    enter_scope() -> void
    exit_scope() -> void
}
```

### 4.4 错误报告格式

遵循 [语言规范 §9](./VGO语言规范.md) 定义的错误代码体系：

```
error[E205]: Dimension mismatch: layer 'output' expects 400, got 784
  --> src/model.yong:12:5
   |
12 |     connect input -> output with stdp;
   |     ^^^^^^^ 'input' has 784 outputs but 'output' expects 400 inputs
   |
   = help: add a hidden layer between them, or resize 'input' to 400
```

### 4.5 扩展点

**新的 Analyzer Pass**: 注册一个实现 `AnalyzerPass` 接口的对象：

```
interface AnalyzerPass {
    name:     string
    order:    int              // 执行顺序 (越小越先)
    run(ast: AST, symbols: SymbolTable) -> List[Diagnostic]
}
```

示例：
- `PerformancePass` — 警告可能的性能问题 (W3xx)
- `DeprecationPass` — 警告已废弃的语法
- `StylePass` — 代码风格检查

---

## 5. Stage 4: IR (中间表示)

### 5.1 职责

将 Typed AST 转换为统一的中间表示。IR 是所有 Backend 共享的输入。

### 5.2 IR 设计原则

| 原则 | 说明 |
|------|------|
| **方言无关** | IR 不区分 App 还是 Hardware，都是统一的节点和边 |
| **类型完备** | 每个 IR 节点都带完整的类型信息 |
| **声明解糖** | 装饰器已展开为具体的 IR 属性 |
| **优化友好** | 可以在 IR 上做 pass 级别的优化 |

### 5.3 IR 节点类型

```
YONG IR
├── DataNode         — 数据定义 (struct → 表/存储)
│   { name, fields[], storage_hint?, constraints[] }
│
├── FunctionNode     — 函数/接口
│   { name, params[], return_type, body: List[IRStmt],
│     properties { route?, method?, auth?, transaction? } }
│
├── ComponentNode    — UI 组件
│   { name, states[], view_tree, event_bindings[] }
│
├── NetworkNode      — SNN 网络
│   { layers[], connections[], config{} }
│
├── PolicyNode       — 权限策略 (已解糖)
│   { resource_type, subject_type, rules[] }
│
├── MigrationNode    — 数据迁移 (已解糖)
│   { version, operations[] }
│
└── IRStmt           — IR 级语句
    ├── IRAssign      { target, value }
    ├── IRCall        { callee, args[] }
    ├── IRBranch      { condition, then_block, else_block }
    ├── IRLoop        { variable, iterable, body }
    ├── IRReturn      { value }
    ├── IRAssert      { condition, message }
    └── IREmit        { event, args[] }
```

### 5.4 装饰器解糖

装饰器在 AST → IR 阶段被「解糖」为 IR 属性：

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

### 5.5 IR 优化 Pass

```
interface IRPass {
    name:     string
    order:    int
    run(ir: List[IRNode]) -> List[IRNode]
}
```

内置 Pass:
- `DeadCodeElimination` — 删除不可达代码
- `ConstantFolding` — 常量折叠
- `ResourceEstimation` — 硬件资源预估 (仅 Hardware)

自定义 Pass:
- `EnergyOptimization` — 基于能量预算的优化
- `TopologySimplification` — 合并等价网络层

---

## 6. Stage 5: Backend

### 6.1 职责

将 YONG IR 转换为目标代码。每个目标对应一个独立的 Backend 插件。

### 6.2 Backend 接口

```
interface Backend {
    // 元信息
    name:           string          // "fpga_rtl", "web_app", "snn_sim"
    target_id:      string          // 用于 config { target: xxx }
    version:        string
    file_extension: string          // ".sv", ".js", ".py"

    // 能力声明
    supports_app_dialect:      bool
    supports_hardware_dialect: bool

    // 生成
    generate(ir: List[IRNode], config: CompilerConfig) -> GeneratedOutput

    // 验证
    validate(ir: List[IRNode]) -> List[Diagnostic]
}

GeneratedOutput {
    files:    Map[string, string]    // 文件名 → 内容
    metadata: Map[string, Any]       // 资源报告、时序估计等
}
```

### 6.3 内置 Backend

| Backend | target_id | 输入 IR | 输出 |
|---------|-----------|---------|------|
| `WebAppBackend` | `web` | FunctionNode, DataNode, ComponentNode, PolicyNode | HTML + CSS + JS + API Server |
| `FPGABackend` | `fpga` | NetworkNode | SystemVerilog + TCL |
| `ASICBackend` | `asic` | NetworkNode | Verilog (可综合) |
| `SimBackend` | `sim` | NetworkNode | Python SNN 仿真器 |
| `MobileBackend` | `mobile` | FunctionNode, DataNode, ComponentNode | React Native / Flutter |

### 6.4 Backend 注册

```
CompilerRegistry {
    backends:           Map[string, Backend]
    decorator_handlers: Map[string, DecoratorHandler]
    type_registry:      TypeRegistry
    learning_rules:     LearningRuleRegistry
    neuron_types:       NeuronTypeRegistry
    analyzer_passes:    List[AnalyzerPass]
    ir_passes:          List[IRPass]

    // 注册方法
    register_backend(backend: Backend)
    register_decorator(name: string, handler: DecoratorHandler)
    register_type(name: string, type_def: TypeDef)
    register_learning_rule(name: string, rule_def: LearningRuleDef)
    register_neuron_type(name: string, neuron_def: NeuronTypeDef)
    register_analyzer_pass(pass: AnalyzerPass)
    register_ir_pass(pass: IRPass)
}
```

### 6.5 扩展 Backend 示例

```python
# 想支持 RISC-V? 写一个 Backend 插件就行:

class RISCVBackend(Backend):
    name = "riscv_rtl"
    target_id = "riscv"
    version = "0.1.0"
    file_extension = ".sv"
    supports_app_dialect = False
    supports_hardware_dialect = True

    def generate(self, ir, config):
        # 把 NetworkNode 转成 RISC-V 加速器 RTL
        ...

    def validate(self, ir):
        # 检查 RISC-V 专有约束
        ...

# 注册:
registry.register_backend(RISCVBackend())
```

---

## 7. 插件扩展接口

### 7.1 装饰器插件

```
interface DecoratorHandler {
    name:       string          // "@cache", "@rate_limit", ...
    dialect:    "app" | "hardware" | "both"
    targets:    List[string]    // 可附加到哪些节点类型: ["fn", "struct", "component"]

    // AST → IR 解糖
    desugar(annotation: AnnotationNode, target: ASTNode) -> IRProperties

    // 语义检查
    validate(annotation: AnnotationNode, target: ASTNode, symbols: SymbolTable) -> List[Diagnostic]
}
```

示例:
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

### 7.2 学习规则插件

```
interface LearningRuleDef {
    name:       string          // "stdp", "bcm", "oja", "r_stdp"
    params:     List[ParamDef]  // 参数定义 (名称/类型/默认值/范围)

    // 语义定义 (伪代码形式, 用于 conformance test)
    weight_update(weight, pre, post, params) -> weight

    // IR 生成: 返回学习规则的 IR 表示
    to_ir(params) -> IRNode
}
```

### 7.3 神经元类型插件

```
interface NeuronTypeDef {
    name:       string          // "lif", "izhikevich", "adex"
    params:     List[ParamDef]  // 参数定义

    // 语义定义
    dynamics(state, input, params) -> (state, spike)

    // IR 生成
    to_ir(params) -> IRNode
}
```

---

## 8. 编译流程 API

### 8.1 一行编译

```python
# 最简用法
output = yong_compile("source.yong", target="web")

# 等价于:
tokens   = Tokenizer(source).tokenize()          # Stage 1
ast      = Parser(tokens).parse()                 # Stage 2
typed    = Analyzer(ast, registry).analyze()       # Stage 3
ir       = IRGenerator(typed).generate()           # Stage 4
output   = registry.get_backend("web").generate(ir) # Stage 5
```

### 8.2 分步编译 (用于 IDE / Debug)

```python
# IDE 场景: 只做到 Stage 3 就够了 (语法高亮 + 错误提示)
tokens = Tokenizer(source).tokenize()
ast    = Parser(tokens).parse()
diags  = Analyzer(ast, registry).analyze()
# → diags 发给 IDE
```

### 8.3 多目标编译

```python
# 同一份源码, 编译到多个目标
ir = compile_to_ir("network.yong")     # Stage 1-4

fpga_out = registry.get_backend("fpga").generate(ir)   # → .sv
sim_out  = registry.get_backend("sim").generate(ir)     # → .py
asic_out = registry.get_backend("asic").generate(ir)    # → .v
```

---

## 9. Conformance 要求

### 9.1 Backend Conformance

任何 Backend 插件要宣称 "YONG Compatible"，必须:

1. 实现 `Backend` 接口的所有方法
2. 通过 [语言规范 §11](./VGO语言规范.md) 定义的 Conformance Test
3. 对同一份 `.yong` + 同一份 IR，多次生成的输出必须确定性一致
4. Hardware Backend 必须满足 **bit-accurate** 要求

### 9.2 Analyzer Conformance

- 遵循 [语言规范 §9](./VGO语言规范.md) 的错误代码体系
- 同一份源码 + 同一版本编译器 → 同一组错误/警告
- 不允许静默吞掉错误

### 9.3 IR Conformance

- 同一份 AST → 同一份 IR（确定性）
- IR 节点必须携带完整类型信息
- 装饰器在 IR 阶段必须已解糖

---

## 10. 与现有实现的关系

### 10.1 `vgo_parser.py` (现状)

现有的 `src/vgo_computing/tools/vgo_parser.py` 是一个早期原型:

| 规范要求 | 现状 | 差距 |
|---------|------|------|
| 5 级管线 | Tokenizer + Parser + CodeGen (3级, 无 Analyzer 和 IR) | 缺 Stage 3, 4 |
| 可插拔后端 | `to_python()` / `to_verilog()` 硬编码在一个类里 | 需重构为 Backend 插件 |
| 支持两大方言 | 只支持 Hardware (network/layer/connect/config) | 缺 App 方言 |
| 符号表 + 类型检查 | 无 | 需新建 |
| 错误代码体系 | `raise SyntaxError(...)` 裸异常 | 需实现 §9 格式 |
| IR | 无 | 需新建 |

### 10.2 演进路线

```
Phase 1 (当前): 原型 — vgo_parser.py, 混合 3 级管线, 只支持 SNN
     ↓
Phase 2: 重构 — 拆分为 5 级, 引入 AnalyzerPass 和 Backend 接口
     ↓
Phase 3: 扩展 — 添加 App 方言 Parser + WebAppBackend
     ↓
Phase 4: 完善 — IR 优化 Pass, 多目标, IDE 集成
```

每个 Phase 都可以独立交付，互不阻塞。

---

## 11. 关键字一览

| 概念 | 说明 |
|------|------|
| `Token` | 词法单元（关键字/标识符/字面量/符号） |
| `AST` | 抽象语法树，Parser 的输出 |
| `Typed AST` | 带类型注解的 AST，Analyzer 的输出 |
| `IR` | 中间表示，所有 Backend 共享的输入 |
| `Backend` | 目标代码生成器（可插拔） |
| `AnalyzerPass` | 语义分析的一个子步骤（可插拔） |
| `IRPass` | IR 优化的一个子步骤（可插拔） |
| `DecoratorHandler` | 装饰器的解糖和校验逻辑（可插拔） |
| `CompilerRegistry` | 全局注册表，所有插件的注册入口 |
| `Diagnostic` | 编译错误/警告，遵循语言规范 §9 格式 |

---

*YONG 编译器规范 v1.0*  
*核心管线不变，插件随意扩展。*  
*"Frozen pipeline, extensible everything else."*
