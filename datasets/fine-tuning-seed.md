# YONG Fine-Tuning Dataset — Seed Pairs

> Natural Language → YONG code pairs for training AI models to generate YONG.
> These pairs serve as the seed dataset. Expand to 1000+ for production fine-tuning.

Format: `instruction` (what human says) → `output` (what AI should generate)

---

## App Dialect — Web Development

### 1. Hello World
```json
{
  "instruction": "Create a hello world web page",
  "output": "@app(route=\"/\")\ncomponent Hello {\n    view {\n        Title(\"Hello, World!\")\n    }\n}"
}
```

### 2. Todo App
```json
{
  "instruction": "Build a todo app with add, list, and complete functionality",
  "output": "@db(table=\"todos\")\nstruct Todo {\n    id: uid\n    text: string\n    done: bool\n}\n\n@api(GET, \"/todos\")\nfn list_todos() -> list[Todo] {\n    return db.todos.all()\n}\n\n@api(POST, \"/todos\")\nfn add_todo(text: string) -> Todo {\n    return db.todos.create({ text, done: false })\n}\n\n@api(PUT, \"/todos/{id}/complete\")\nfn complete_todo(id: uid) -> Todo {\n    return db.todos.find(id) |> merge({ done: true }) |> save\n}\n\n@app(route=\"/\")\ncomponent TodoApp {\n    state todos = query(list_todos)\n    view {\n        Header(\"My Todos\")\n        Input(placeholder=\"Add...\", on_enter=add_todo)\n        List(todos) -> |todo| {\n            Row { Checkbox(todo.done, on_change=complete_todo) Text(todo.text) }\n        }\n    }\n}"
}
```

### 3. User Registration API
```json
{
  "instruction": "Create a user registration endpoint with email validation and password hashing",
  "output": "@db(table=\"users\")\nstruct User {\n    id: uid\n    email: string @unique @validate(email)\n    password: string @hash(bcrypt)\n    name: string\n    created_at: timestamp = now()\n}\n\n@api(POST, \"/register\")\nfn register(email: string, password: string, name: string) -> Result<User, ValidationError> {\n    return { email, password, name }\n        |> validate\n        |> save\n}"
}
```

### 4. Blog Post CRUD
```json
{
  "instruction": "Create a blog with posts, authors, and publish/draft status",
  "output": "@db(table=\"posts\")\nstruct Post {\n    id: uid\n    author_id: uid @ref(User)\n    title: string @validate(min=1, max=200)\n    slug: string @unique @derive(title |> slugify)\n    content: text @markdown\n    status: enum { draft, published } = draft\n    published_at: timestamp?\n    created_at: timestamp = now()\n}\n\n@api(GET, \"/posts\")\nfn list_posts(page: int = 1) -> Paginated<Post> {\n    return db.posts\n        |> where(status == \"published\")\n        |> order_by(published_at, desc)\n        |> paginate(page, 20)\n        |> include(author)\n}\n\n@api(POST, \"/posts\")\n@auth(role: author)\nfn create_post(title: string, content: string) -> Post {\n    return { title, content, author_id: auth.user.id } |> validate |> save\n}\n\n@api(POST, \"/posts/{id}/publish\")\n@auth(role: author, owner: author_id)\nfn publish_post(id: uid) -> Post {\n    return db.posts.find(id)\n        |> merge({ status: \"published\", published_at: now() })\n        |> save\n}"
}
```

### 5. JWT Authentication
```json
{
  "instruction": "Create login and protected endpoint with JWT tokens",
  "output": "@api(POST, \"/login\")\nfn login(email: string, password: string) -> Result<AuthToken, AuthError> {\n    return db.users.find_by(email: email)\n        |? err(AuthError { message: \"User not found\" })\n        |> verify_password(password)\n        |> generate_jwt(expires: 24h)\n}\n\n@api(GET, \"/me\")\n@auth\nfn get_profile() -> User {\n    return db.users.find(auth.user.id)\n}"
}
```

### 6. REST API with Pagination and Search
```json
{
  "instruction": "Create a product catalog API with search, filtering, and pagination",
  "output": "@db(table=\"products\")\nstruct Product {\n    id: uid\n    name: string\n    price: Money<CNY>\n    category: string\n    active: bool = true\n}\n\n@api(GET, \"/products\")\nfn list_products(page: int = 1, category: string?, q: string?) -> Paginated<Product> {\n    return db.products\n        |> where(active == true)\n        |> where_if(category, category == category)\n        |> full_text_search_if(q, fields: [name])\n        |> order_by(name, asc)\n        |> paginate(page, 20)\n}"
}
```

### 7. File Upload API
```json
{
  "instruction": "Create an image upload endpoint that validates file type and size",
  "output": "@api(POST, \"/upload\")\n@auth\nfn upload_image(file: File) -> Result<ImageUrl, UploadError> {\n    return file\n        |> validate_type([\"image/png\", \"image/jpeg\", \"image/webp\"])\n        |> validate_size(max: 5MB)\n        |> resize(max_width: 1920)\n        |> store(bucket: \"uploads\")\n}"
}
```

### 8. Webhook Handler
```json
{
  "instruction": "Create a Stripe payment webhook handler",
  "output": "@api(POST, \"/webhooks/stripe\")\n@webhook(secret: config.stripe_webhook_secret)\nfn handle_stripe(event: WebhookEvent) -> void {\n    match event.type {\n        \"payment_intent.succeeded\" => {\n            db.orders.find_by(payment_id: event.data.id)\n                |> merge({ status: \"paid\" })\n                |> save\n                |> emit(\"order.paid\")\n        }\n        \"payment_intent.failed\" => {\n            db.orders.find_by(payment_id: event.data.id)\n                |> merge({ status: \"failed\" })\n                |> save\n        }\n    }\n}"
}
```

---

## Hardware Dialect — SNN / Neuromorphic

### 9. MNIST Classifier
```json
{
  "instruction": "Create an SNN for MNIST digit classification",
  "output": "network MNIST {\n    layer input(784);\n    layer hidden(400, type=lif, threshold=300, leak=1);\n    layer output(10, type=lif, threshold=200, leak=2);\n\n    connect input -> hidden with stdp(ltp=5, ltd=2);\n    connect hidden -> output with stdp(ltp=10, ltd=4);\n\n    config {\n        weight_storage: bram\n        wta_mode: enable\n        homeostasis: enable\n        target: zynq7020\n    }\n}"
}
```

### 10. Multi-Layer SNN with Inhibition
```json
{
  "instruction": "Create a 3-layer SNN with lateral inhibition for pattern recognition",
  "output": "network PatternNet {\n    layer input(256);\n    layer hidden1(200, type=lif, threshold=280, leak=2) {\n        lateral_inhibition: true\n    }\n    layer hidden2(100, type=lif, threshold=250, leak=1);\n    layer output(20, type=lif, threshold=200, leak=1);\n\n    connect input -> hidden1 with stdp(ltp=4, ltd=2);\n    connect hidden1 -> hidden2 with stdp(ltp=6, ltd=3);\n    connect hidden2 -> output with stdp(ltp=8, ltd=4);\n\n    config {\n        weight_storage: bram\n        wta_mode: enable\n        homeostasis: enable\n        target: ice40_hx8k\n    }\n}"
}
```

### 11. Recurrent SNN for Temporal Sequences
```json
{
  "instruction": "Create a recurrent SNN for processing temporal sensor data",
  "output": "network TemporalNet {\n    layer input(128, encoding=rate, time_window=50);\n    layer reservoir(500, type=lif, threshold=250, leak=3) {\n        recurrent: true\n        connectivity: 0.1\n        exc_inh_ratio: 0.8\n    }\n    layer output(10, type=lif, threshold=180, leak=1);\n\n    connect input -> reservoir with stdp(ltp=3, ltd=1);\n    connect reservoir -> output with stdp(ltp=7, ltd=3) {\n        delay: uniform(1, 5)\n    }\n\n    decode output with spike_count(window=20);\n\n    config {\n        weight_storage: bram\n        power_gating: enable\n        target: zynq7020\n        clock: 50MHz\n    }\n}"
}
```

### 12. DVS Camera Event Processor
```json
{
  "instruction": "Create an SNN that processes events from a DVS camera for gesture recognition",
  "output": "network GestureNet {\n    layer input(346 * 260, encoding=event);\n    layer conv1(64, type=lif, threshold=300, leak=2, kernel=5, stride=2);\n    layer conv2(128, type=lif, threshold=250, leak=2, kernel=3);\n    layer pool(128, type=max_pool, window=2);\n    layer fc(256, type=lif, threshold=200, leak=1);\n    layer output(11, type=lif, threshold=150, leak=1);\n\n    connect input -> conv1 with stdp(ltp=3, ltd=1);\n    connect conv1 -> conv2 with stdp(ltp=5, ltd=2);\n    connect conv2 -> pool;\n    connect pool -> fc with stdp(ltp=7, ltd=3);\n    connect fc -> output with stdp(ltp=10, ltd=4);\n\n    decode output with spike_count(window=30);\n\n    config {\n        weight_storage: bram\n        wta_mode: enable\n        target: zynq7020\n    }\n}"
}
```

---

## Error Handling Patterns

### 13. Validation Pipeline
```json
{
  "instruction": "Create a data validation pipeline with multiple error types",
  "output": "@error\nstruct ValidationError {\n    field: string\n    message: string\n    code: int = 400\n}\n\n@error\nstruct DatabaseError {\n    operation: string\n    detail: string\n    code: int = 500\n}\n\ntype CreateUserError = ValidationError | DatabaseError\n\n@api(POST, \"/users\")\nfn create_user(req: CreateUserRequest) -> Result<User, CreateUserError> {\n    return req\n        |> validate_email\n        |> validate_password_strength\n        |> check_unique_email\n        |> save\n        |> send_welcome_email\n        |! log_error\n}"
}
```

### 14. Retry with Fallback
```json
{
  "instruction": "Create a function that retries on database error and falls back to cache",
  "output": "@retry(max=3, backoff=exponential, on=DatabaseError)\nfn get_user(id: uid) -> Result<User, ServiceError> {\n    return db.users.find(id)\n        |recover DatabaseError => cache.get(\"user:\" + id)\n        |? not_found(\"User not found\")\n}"
}
```

---

## Mixed Patterns

### 15. State Machine
```json
{
  "instruction": "Create an order status state machine with compile-time transition validation",
  "output": "@state_machine\nenum OrderStatus {\n    pending   -> [paid, cancelled]\n    paid      -> [processing, refunded]\n    processing -> [shipped]\n    shipped   -> [delivered, returned]\n    delivered -> [returned]\n    cancelled -> []\n    refunded  -> []\n    returned  -> [refunded]\n}\n\n@api(POST, \"/orders/{id}/ship\")\n@auth(role: admin)\nfn ship_order(id: uid) -> Order {\n    return db.orders.find(id)\n        |> assert(status == processing)\n        |> transition(status: shipped)\n        |> save\n        |> emit(\"order.shipped\")\n}"
}
```

### 16. Real-Time WebSocket
```json
{
  "instruction": "Create a real-time chat room with WebSocket",
  "output": "@db(table=\"messages\")\nstruct Message {\n    id: uid\n    room_id: uid\n    user_id: uid @ref(User)\n    content: string @validate(min=1, max=2000)\n    created_at: timestamp = now()\n}\n\n@ws(\"/chat/{room_id}\")\n@auth\nfn chat_room(room_id: uid) {\n    on_message(content: string) {\n        { room_id, content, user_id: auth.user.id }\n            |> validate\n            |> save\n            |> broadcast(room_id)\n    }\n    on_connect {\n        db.messages\n            |> where(room_id == room_id)\n            |> order_by(created_at, desc)\n            |> limit(50)\n            |> send\n    }\n}"
}
```

---

## Statistics

| Category | Count | Coverage |
|----------|-------|----------|
| App Dialect (Web) | 8 pairs | CRUD, Auth, Upload, Webhook, Search |
| Hardware Dialect (SNN) | 4 pairs | MLP, Recurrent, Conv, DVS |
| Error Handling | 2 pairs | Validation, Retry/Fallback |
| Mixed Patterns | 2 pairs | State Machine, WebSocket |
| **Total** | **16 seed pairs** | |

> **Next step**: Expand to 100+ pairs with variations, then to 1000+ for production fine-tuning.

---

*Seed dataset v0.1 — 2026-02-10*
*Format compatible with OpenAI fine-tuning API and HuggingFace datasets*
