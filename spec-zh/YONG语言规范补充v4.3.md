# YONG 语言规范 — 补充 v4.3

> 针对 YONG 语言规范 v4.2 的扩展，回应社区和 AI 同行评审反馈。

---

## 1. 错误处理

YONG 使用**铁路导向编程（Railway-Oriented Programming）**——错误通过管道像数据一样流动，不需要 try/catch。

### 1.1 Result 类型

每个函数隐式返回 `Result<T, E>`。编译器自动包装返回值。

```yong
fn divide(a: int, b: int) -> Result<int, DivisionError> {
    match b {
        0 => err(DivisionError { message: "除以零" })
        _ => ok(a / b)
    }
}
```

### 1.2 管道错误运算符

| 运算符 | 行为 | 示例 |
|--------|------|------|
| `\|>` | Ok 时继续，Err 时短路 | `data \|> validate \|> save` |
| `\|!` | 处理错误（catch 分支）| `data \|> save \|! log_error` |
| `\|?` | 解包或提前返回 | `db.find(id) \|? not_found("...")` |
| `\|recover` | 将特定错误转为备选值 | `fetch() \|recover Timeout => use_cache()` |

### 1.3 错误类型

错误是带 `@error` 装饰器的结构体。编译器自动生成 HTTP 状态码映射。

```yong
@error
struct ValidationError {
    field: string
    message: string
    code: int = 400     // 映射到 HTTP 400
}
```

### 1.4 联合错误类型

```yong
type ServiceError = ValidationError | DatabaseError | AuthError
```

### 1.5 重试装饰器

```yong
@retry(max=3, backoff=exponential, on=DatabaseError)
fn save(data: T) -> Result<T, DatabaseError> { ... }
```

---

## 2. 逃生舱口 — `@raw` 块

当 YONG 的声明式抽象无法覆盖特定需求时，`@raw` 块允许内嵌目标语言代码。

### 2.1 语法

```yong
@raw(target: "javascript")
fn custom_payment(order: Order) -> Result<PaymentResult, PaymentError> {
    // 这里是原生 JavaScript 代码
    const stripe = require('stripe')(process.env.STRIPE_KEY);
    const session = await stripe.checkout.sessions.create({...});
    return { id: session.id, url: session.url };
}
```

### 2.2 规则

1. `@raw` 块在**边界处是有类型的**——输入和输出类型必须用 YONG 声明。
2. 编译器**不分析** `@raw` 块的内容。
3. `target` 参数指定原始代码对应的后端目标。如果 `.yong` 文件被编译到不同的后端，`@raw` 块会产生编译错误。
4. `@raw` 块是**最后手段**。规范不鼓励使用，lint 工具会标记它们。

### 2.3 支持的目标

| 目标 | 用途 |
|------|------|
| `javascript` | 浏览器 API、npm 包 |
| `python` | ML 库、数据科学 |
| `verilog` | 自定义 RTL 模块 |
| `sql` | 超出 ORM 能力的复杂查询 |

---

## 3. 中间件与扩展系统

### 3.1 中间件装饰器

```yong
@middleware
fn rate_limiter(req: Request, next: Handler) -> Response {
    return check_rate(req.ip)
        |? too_many_requests("请求过于频繁")
        |> next(req)
}

@api(POST, "/orders")
@use(rate_limiter, cors, logging)
fn create_order(req: OrderRequest) -> Order { ... }
```

---

## 4. 安全模型

### 4.1 认证

```yong
@auth                           // 需要任何已认证用户
@auth(role: admin)              // 需要管理员角色
@auth(role: [author, admin])    // 需要作者或管理员
@auth(role: admin, owner: author_id)  // 管理员，或资源所有者
```

**编译时保证:**
- 每个没有 `@auth` 的 `@api` 端点在编译输出中标记为 `PUBLIC`
- `auth.user` 只在 `@auth` 装饰的函数中可访问——否则编译错误
- 角色检查作为中间件链的**第一个**插入

### 4.2 密钥管理

```yong
config {
    // 密钥永远不会嵌入编译输出
    // 它们在运行时从环境变量读取
    jwt_secret: env("JWT_SECRET")         // 必需，启动时失败
    db_password: env("DB_PASSWORD")       // 必需
    stripe_key: env("STRIPE_KEY", "")     // 可选，空默认值
}
```

### 4.3 SQL 注入防护

YONG 的 ORM 层**专用参数化查询**。数据库操作中没有字符串拼接。编译器拒绝任何用用户输入构造原始 SQL 的尝试。

---

## 5. 类型系统

### 5.1 推断规则

| 上下文 | 推断 | 示例 |
|--------|------|------|
| 结构体字段 | 类型必需 | `name: string` |
| 函数参数 | 类型必需 | `fn foo(x: int)` |
| 函数返回值 | 从函数体推断，或显式声明 | `fn foo() -> int` |
| 局部变量 | 从赋值推断 | `let x = 42  // x: int` |
| 管道链 | 从流推断 | `data \|> map(fn(x) { x + 1 })` |

**规则**: 所有公开 API 边界（结构体字段、函数签名）需要显式类型。内部逻辑可以使用推断。

### 5.2 内置类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `int` | 64位整数 | `42` |
| `float` | 64位浮点 | `3.14` |
| `bool` | 布尔 | `true` |
| `string` | UTF-8 字符串 | `"hello"` |
| `uid` | UUID v7 | 自动生成 |
| `timestamp` | ISO 8601 | `now()` |
| `list[T]` | 有序集合 | `list[int]` |
| `T?` | 可选（可空）| `string?` |
| `Result<T, E>` | Ok 或 Error | `Result<User, AuthError>` |
| `Money<C>` | 货币安全金额 | `Money<CNY>` |
| `Time<U>` | 单位安全时间 | `Time<ms>` |
| `Energy<U>` | 单位安全能量 | `Energy<pJ>` |

### 5.3 单位安全（编译时）

```yong
let t1: Time<ms> = 100
let t2: Time<s> = 2
let t3 = t1 + t2        // 编译错误 E202: 不能相加 Time<ms> + Time<s>
let t3 = t1 + t2.to_ms  // OK: 2100 ms
```

### 5.4 泛型

```yong
fn first<T>(items: list[T]) -> T? {
    match items {
        [head, ..] => some(head)
        [] => none
    }
}
```

### 5.5 联合类型

```yong
type Shape = Circle | Rectangle | Triangle
```

---

## 6. 版本策略

### 6.1 语义化版本

| 变更 | 级别 | 示例 |
|------|------|------|
| 新关键字 / 运算符 | MAJOR | 添加 `async` |
| 新装饰器 | MINOR | 添加 `@cache` |
| 编译器修复 | PATCH | 修复类型推断边界 |

### 6.2 冻结核心

以下在 v5.0 后**不再变更**：
- 管道运算符: `|>`, `|!`, `|?`, `|recover`
- 核心装饰器: `@api`, `@db`, `@app`, `@auth`, `@error`
- 类型语法: `T?`, `list[T]`, `Result<T, E>`, `Money<C>`, `Time<U>`, `Energy<U>`

### 6.3 扩展点（开放）

- 新装饰器（通过插件）
- 新编译目标（后端插件）
- 新 `@validate` 规则
- 新 `@raw` 目标语言

### 6.4 编译器指令

```yong
@yong(version: "4.3")  // 文件顶部，确保编译器兼容性
```

---

## 7. 解释模式 / 美化打印器

### 7.1 `yong explain`

将 YONG 翻译为详细注释的目标语言代码，供人类审阅。

### 7.2 `yong fmt`

`.yong` 文件标准格式化工具。

### 7.3 `yong check`

方言感知的 lint：

```bash
$ yong check --target=web todo-app.yong    # Web 专用规则
$ yong check --target=rtl snn-mnist.yong   # 硬件专用规则
```

---

## 8. AI 友好的编译器错误消息

错误结构化且包含建议修复：

```json
{
  "code": "E101",
  "severity": "error",
  "line": 12,
  "message": "类型不匹配",
  "expected": "Money<CNY>",
  "got": "int",
  "suggestion": "用 Money<CNY>(amount) 包装",
  "fix": "total: order.subtotal + Money<CNY>(discount_amount)"
}
```

---

*YONG 语言规范补充 v4.3*
*编写时间: 2026-02-10*
*状态: 草案 — 回应 AI 同行评审反馈*
