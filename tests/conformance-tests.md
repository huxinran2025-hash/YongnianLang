# YONG Conformance Tests

> Golden output tests ensuring YONG compilation is deterministic and correct.

---

## Test Structure

Each test case consists of:
1. **Input**: A `.yong` source file
2. **Expected Output**: The compiler's expected behavior (success with output, or error with code)
3. **Category**: Which aspect of the language is tested

## Test Categories

### T1: Struct & Type Declaration

```
Test: T1-01 — Basic struct declaration
Input: struct User { id: uid; name: string; age: int }
Expected: PASS — valid struct with 3 fields, types resolved

Test: T1-02 — Struct with optional field
Input: struct Profile { bio: string?; avatar: string? }
Expected: PASS — optional fields accepted

Test: T1-03 — Struct with validation
Input: struct Email { address: string @validate(email) }
Expected: PASS — @validate(email) generates regex validator

Test: T1-04 — Invalid type reference
Input: struct Order { user: NonExistent }
Expected: ERROR E301 — unknown type 'NonExistent'
```

### T2: API Declaration

```
Test: T2-01 — GET endpoint
Input: @api(GET, "/users") fn list_users() -> list[User] { return db.users.all() }
Expected: PASS — generates GET route with JSON serialization

Test: T2-02 — POST with body parsing
Input: @api(POST, "/users") fn create(name: string, email: string) -> User { ... }
Expected: PASS — generates POST route with body validation

Test: T2-03 — Missing route
Input: @api(GET) fn broken() -> void {}
Expected: ERROR E102 — @api requires (method, path)

Test: T2-04 — Invalid HTTP method
Input: @api(PATCH, "/users/{id}") fn update(id: uid) -> User { ... }
Expected: PASS — PATCH is valid HTTP method
```

### T3: Pipe Operators

```
Test: T3-01 — Basic pipe chain
Input: data |> validate |> save
Expected: PASS — sequential function application

Test: T3-02 — Error short-circuit
Input: data |> validate |> save (where validate returns Err)
Expected: PASS — save is NOT called, Err propagates

Test: T3-03 — Error handler
Input: data |> save |! log_error
Expected: PASS — log_error called on Err from save

Test: T3-04 — Unwrap operator
Input: db.find(id) |? not_found("...")
Expected: PASS — returns early with not_found error if None
```

### T4: Authentication

```
Test: T4-01 — Protected endpoint
Input: @api(GET, "/me") @auth fn profile() -> User { return db.users.find(auth.user.id) }
Expected: PASS — auth middleware injected before handler

Test: T4-02 — Role-based access
Input: @auth(role: admin) fn admin_only() -> void {}
Expected: PASS — role check middleware added

Test: T4-03 — auth.user without @auth
Input: @api(GET, "/test") fn test() -> User { return db.users.find(auth.user.id) }
Expected: ERROR E401 — 'auth.user' accessed without @auth decorator

Test: T4-04 — Public endpoint warning
Input: @api(POST, "/payments") fn pay(amount: Money<CNY>) -> void { ... }
Expected: WARNING W101 — endpoint without @auth is PUBLIC
```

### T5: Unit Safety

```
Test: T5-01 — Same unit addition
Input: let t = Time<ms>(100) + Time<ms>(200)
Expected: PASS — result is Time<ms>(300)

Test: T5-02 — Mixed unit addition
Input: let t = Time<ms>(100) + Time<s>(2)
Expected: ERROR E202 — cannot add Time<ms> + Time<s>

Test: T5-03 — Unit conversion
Input: let t = Time<ms>(100) + Time<s>(2).to_ms
Expected: PASS — result is Time<ms>(2100)

Test: T5-04 — Money cross-currency
Input: let m = Money<CNY>(100) + Money<USD>(15)
Expected: ERROR E203 — cannot add Money<CNY> + Money<USD>
```

### T6: SNN / Hardware Dialect

```
Test: T6-01 — Basic network declaration
Input: network Test { layer input(784); layer output(10, type=lif); connect input -> output with stdp(ltp=5, ltd=2); }
Expected: PASS — generates valid Verilog RTL

Test: T6-02 — Missing connection
Input: network Test { layer input(784); layer output(10, type=lif); }
Expected: WARNING W201 — layer 'input' has no outgoing connections

Test: T6-03 — Invalid target
Input: config { target: invalid_fpga }
Expected: ERROR E501 — unknown target 'invalid_fpga'

Test: T6-04 — Weight storage mismatch
Input: network Large { layer huge(100000); config { weight_storage: bram; target: ice40_hx8k } }
Expected: ERROR E502 — 100000 neurons exceed iCE40 HX8K BRAM capacity
```

### T7: Error Handling

```
Test: T7-01 — @error struct
Input: @error struct MyError { message: string; code: int = 400 }
Expected: PASS — generates error type with HTTP status mapping

Test: T7-02 — Union error type
Input: type ServiceError = ValidationError | DatabaseError
Expected: PASS — union type accepted

Test: T7-03 — Retry decorator
Input: @retry(max=3, backoff=exponential, on=DatabaseError) fn save(x: T) -> Result<T, DatabaseError> { ... }
Expected: PASS — generates retry wrapper

Test: T7-04 — Recover operator
Input: fetch() |recover Timeout => use_cache()
Expected: PASS — Timeout caught, use_cache called
```

### T8: Escape Hatch

```
Test: T8-01 — @raw with matching target
Input: @raw(target: "javascript") fn custom() -> string { return "hello"; }
Target: --target=web
Expected: PASS — raw JavaScript embedded in output

Test: T8-02 — @raw with wrong target
Input: @raw(target: "javascript") fn custom() -> string { ... }
Target: --target=rtl
Expected: ERROR E601 — @raw(target: "javascript") incompatible with RTL backend

Test: T8-03 — @raw lint warning
Input: @raw(target: "sql") fn query() -> list[User] { ... }
Expected: WARNING W301 — @raw blocks bypass YONG safety guarantees
```

---

## Running Tests

```bash
$ yong test conformance/          # run all conformance tests
$ yong test conformance/T5-*.yong # run unit safety tests only
$ yong test --golden-update       # update expected outputs
```

---

## Test Count Summary

| Category | Tests | Coverage |
|----------|-------|----------|
| T1: Struct & Types | 4 | Declaration, Optional, Validation, Error |
| T2: API | 4 | GET, POST, Error, PATCH |
| T3: Pipes | 4 | Chain, Short-circuit, Handler, Unwrap |
| T4: Auth | 4 | Protected, RBAC, Safety, Warning |
| T5: Unit Safety | 4 | Same, Mixed, Conversion, Currency |
| T6: Hardware | 4 | Basic, Warning, Target, Capacity |
| T7: Error Handling | 4 | Struct, Union, Retry, Recover |
| T8: Escape Hatch | 3 | Match, Mismatch, Warning |
| **Total** | **31 conformance tests** | |

---

*Conformance Test Suite v0.1 — 2026-02-10*
