# YONG Language Specification — Addendum v4.3

> Extensions to the YONG Language Specification v4.2, addressing community and AI peer review feedback.

---

## 1. Error Handling

YONG uses **Railway-Oriented Programming** — errors flow through pipes like data, without try/catch blocks.

### 1.1 Result Type

Every function implicitly returns `Result<T, E>`. The compiler wraps return values automatically.

```yong
fn divide(a: int, b: int) -> Result<int, DivisionError> {
    match b {
        0 => err(DivisionError { message: "Division by zero" })
        _ => ok(a / b)
    }
}
```

### 1.2 Pipe Error Operators

| Operator | Behavior | Example |
|----------|----------|---------|
| `\|>` | Continue on Ok, short-circuit on Err | `data \|> validate \|> save` |
| `\|!` | Handle error (catch branch) | `data \|> save \|! log_error` |
| `\|?` | Unwrap or return early | `db.find(id) \|? not_found("...")` |
| `\|recover` | Convert specific error to fallback | `fetch() \|recover Timeout => use_cache()` |

### 1.3 Error Types

Errors are structs with `@error` decorator. The compiler generates HTTP status mapping.

```yong
@error
struct ValidationError {
    field: string
    message: string
    code: int = 400     // maps to HTTP 400
}
```

### 1.4 Union Error Types

```yong
type ServiceError = ValidationError | DatabaseError | AuthError
```

### 1.5 Retry Decorator

```yong
@retry(max=3, backoff=exponential, on=DatabaseError)
fn save(data: T) -> Result<T, DatabaseError> { ... }
```

---

## 2. Escape Hatch — `@raw` Blocks

When YONG's declarative abstractions don't cover a specific need, `@raw` blocks allow inline target-language code.

### 2.1 Syntax

```yong
@raw(target: "javascript")
fn custom_payment(order: Order) -> Result<PaymentResult, PaymentError> {
    // Raw JavaScript code here
    const stripe = require('stripe')(process.env.STRIPE_KEY);
    const session = await stripe.checkout.sessions.create({...});
    return { id: session.id, url: session.url };
}
```

### 2.2 Rules

1. `@raw` blocks are **typed at their boundary** — input and output types must be declared in YONG.
2. The compiler **does not analyze** the contents of a `@raw` block.
3. The `target` parameter specifies which backend the raw code is for. If the `.yong` file is compiled to a different backend, the `@raw` block is a compile error.
4. `@raw` blocks are a **last resort**. The spec discourages their use and linting tools flag them.

### 2.3 Supported Targets

| Target | Use Case |
|--------|----------|
| `javascript` | Browser APIs, npm packages |
| `python` | ML libraries, data science |
| `verilog` | Custom RTL blocks |
| `sql` | Complex queries beyond ORM |

---

## 3. Middleware & Extension System

### 3.1 Middleware Decorator

```yong
@middleware
fn rate_limiter(req: Request, next: Handler) -> Response {
    return check_rate(req.ip)
        |? too_many_requests("Rate limit exceeded")
        |> next(req)
}

@api(POST, "/orders")
@use(rate_limiter, cors, logging)
fn create_order(req: OrderRequest) -> Order { ... }
```

### 3.2 Plugin System

```yong
// Plugins extend compiler behavior, not runtime behavior
@plugin("stripe")
import stripe { PaymentIntent, Customer }

// Plugin provides type definitions + compilation rules
// The compiler knows how to materialize Stripe API calls
```

---

## 4. Security Model

### 4.1 Authentication

```yong
@auth                           // requires any authenticated user
@auth(role: admin)              // requires admin role
@auth(role: [author, admin])    // requires author OR admin
@auth(role: admin, owner: author_id)  // admin, or owner of the resource
```

**Compile-time guarantees:**
- Every `@api` endpoint without `@auth` is flagged as `PUBLIC` in compilation output
- `auth.user` is only accessible inside `@auth`-decorated functions — compile error otherwise
- Role checks are inserted as the **first** middleware in the chain

### 4.2 Secrets Management

```yong
config {
    // Secrets are NEVER embedded in compiled output
    // They are read from environment at runtime
    jwt_secret: env("JWT_SECRET")         // required, fail-fast on startup
    db_password: env("DB_PASSWORD")       // required
    stripe_key: env("STRIPE_KEY", "")     // optional, empty default

    // Compile error if a secret is used in a non-@auth context
}
```

### 4.3 Input Validation

All `@validate` annotations are enforced at **compile time** (generating validation middleware) and checked at **runtime** (before handler execution).

```yong
struct CreateUserRequest {
    email: string @validate(email)          // compile → regex validator
    name: string @validate(min=1, max=100)  // compile → length check
    age: int @validate(min=0, max=200)      // compile → range check
}
```

### 4.4 SQL Injection Prevention

YONG's ORM layer uses **parameterized queries exclusively**. There is no string interpolation in database operations. The compiler rejects any attempt to construct raw SQL with user input (unless inside a `@raw(target: "sql")` block, which is explicitly flagged as unsafe).

---

## 5. Type System

### 5.1 Inference Rules

| Context | Inference | Example |
|---------|-----------|---------|
| Struct fields | Type required | `name: string` |
| Function params | Type required | `fn foo(x: int)` |
| Function return | Inferred from body, or explicit | `fn foo() -> int` or `fn foo() { return 42 }` |
| Local variables | Inferred from assignment | `let x = 42  // x: int` |
| Pipe chains | Inferred from flow | `data \|> map(fn(x) { x + 1 })  // inferred` |

**Rule**: All public API boundaries (struct fields, function signatures) require explicit types. Internal logic may use inference.

### 5.2 Built-in Types

| Type | Description | Example |
|------|-------------|---------|
| `int` | 64-bit integer | `42` |
| `float` | 64-bit float | `3.14` |
| `bool` | Boolean | `true` |
| `string` | UTF-8 string | `"hello"` |
| `text` | Long-form text | (same as string, semantic hint) |
| `uid` | UUID v7 | Auto-generated |
| `timestamp` | ISO 8601 | `now()` |
| `list[T]` | Ordered collection | `list[int]` |
| `map[K, V]` | Key-value map | `map[string, int]` |
| `set[T]` | Unique collection | `set[string]` |
| `T?` | Optional (nullable) | `string?` |
| `Result<T, E>` | Ok or Error | `Result<User, AuthError>` |
| `Money<C>` | Currency-safe amount | `Money<CNY>`, `Money<USD>` |
| `Time<U>` | Unit-safe time | `Time<ms>`, `Time<s>` |
| `Energy<U>` | Unit-safe energy | `Energy<pJ>`, `Energy<mJ>` |

### 5.3 Unit Safety (Compile-time)

```yong
let t1: Time<ms> = 100
let t2: Time<s> = 2
let t3 = t1 + t2        // Compile error E202: cannot add Time<ms> + Time<s>
let t3 = t1 + t2.to_ms  // OK: 2100 ms
```

### 5.4 Generic Types

```yong
fn first<T>(items: list[T]) -> T? {
    match items {
        [head, ..] => some(head)
        [] => none
    }
}

struct Paginated<T> {
    items: list[T]
    page: int
    total: int
    has_next: bool
}
```

### 5.5 Union Types

```yong
type Shape = Circle | Rectangle | Triangle

fn area(s: Shape) -> float {
    match s {
        Circle { radius }    => 3.14159 * radius * radius
        Rectangle { w, h }   => w * h
        Triangle { b, h }    => 0.5 * b * h
    }
}
```

### 5.6 Enum Types

```yong
// Simple enum
enum Color { red, green, blue }

// Enum with associated data
enum Status {
    pending
    active(since: timestamp)
    suspended(reason: string, until: timestamp)
}
```

---

## 6. Determinism Guarantees

### 6.1 Bit-Accurate Promise

The same `.yong` file compiled with the same compiler version produces:
- **Identical** web application behavior across Node.js, Deno, Bun
- **Identical** Verilog RTL bitstream across synthesis tools (within tool rounding)

### 6.2 IO Boundaries

Functions with side effects are marked by the compiler:

| Decorator/Pattern | Side Effect | Deterministic? |
|-------------------|-------------|----------------|
| `@api` | Network IO | No |
| `db.*` | Database IO | No |
| `emit()` | Event dispatch | No |
| `now()` | System clock | No |
| Pure functions | None | **Yes** |

The compiler tracks effect propagation — a pure function cannot call an effectful function without explicit annotation.

---

## 7. Versioning Strategy

### 7.1 Semantic Versioning

YONG follows `MAJOR.MINOR.PATCH`:

| Change | Level | Example |
|--------|-------|---------|
| New keyword / operator | MAJOR | Adding `async` keyword |
| New decorator | MINOR | Adding `@cache` |
| Bug fix in compiler | PATCH | Fixing edge case in type inference |

### 7.2 Frozen Core

The following are **frozen** (will not change after v5.0):
- Pipe operators: `|>`, `|!`, `|?`, `|recover`
- Core decorators: `@api`, `@db`, `@app`, `@auth`, `@error`
- Type syntax: `T?`, `list[T]`, `Result<T, E>`, `Money<C>`, `Time<U>`, `Energy<U>`
- Struct definition syntax
- Function definition syntax

### 7.3 Extension Points (Open)

- New decorators (via plugins)
- New compilation targets (backend plugins)
- New `@validate` rules
- New `@raw` target languages
- Custom middleware

### 7.4 Compiler Directive

```yong
@yong(version: "4.3")  // at top of file, ensures compiler compatibility
```

---

## 8. Explanatory Mode / Pretty Printer

### 8.1 `yong explain`

Translates YONG into verbose, annotated target-language code for human review:

```bash
$ yong explain todo-app.yong --target=python

# Output: fully commented Python equivalent
# from flask import Flask, request, jsonify
# from sqlalchemy import create_engine, Column, String, Boolean
# # YONG: @db(table="todos") → creates SQLAlchemy model
# class Todo(Base):
#     __tablename__ = 'todos'
#     ...
```

### 8.2 `yong fmt`

Standard formatter for `.yong` files:

```bash
$ yong fmt --check .       # verify formatting
$ yong fmt --write .       # auto-format in place
```

### 8.3 `yong check`

Dialect-aware linting:

```bash
$ yong check --target=web todo-app.yong    # web-specific rules
$ yong check --target=rtl snn-mnist.yong   # hardware-specific rules

# Error: @api decorator not available in RTL target
# Warning: network block not available in Web target
```

---

## 9. Compiler Error Messages (AI-Friendly)

Errors are structured and include suggested fixes:

```
E101: Type mismatch at line 12
  Expected: Money<CNY>
  Got:      int
  
  Suggestion: wrap with Money<CNY>(amount)
  
  12 |   total: order.subtotal + discount_amount
                                  ^^^^^^^^^^^^^^^
                                  This is `int`, expected `Money<CNY>`

Fix: total: order.subtotal + Money<CNY>(discount_amount)
```

For AI consumers, errors are also available in JSON:

```json
{
  "code": "E101",
  "severity": "error",
  "line": 12,
  "message": "Type mismatch",
  "expected": "Money<CNY>",
  "got": "int",
  "suggestion": "wrap with Money<CNY>(amount)",
  "fix": "total: order.subtotal + Money<CNY>(discount_amount)"
}
```

---

*YONG Language Specification Addendum v4.3*
*Authored: 2026-02-10*
*Status: Draft — addresses AI peer review feedback*
