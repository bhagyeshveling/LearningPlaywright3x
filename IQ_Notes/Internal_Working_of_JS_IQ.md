# Internal Working of a JavaScript Program

> **Example:** `console.log("Hello World!");` saved in `01_HelloWorld.js`

---

## TL;DR

| Stage | What happens | Component | Output |
|-------|-------------|-----------|--------|
| **Parsing** | Source code is tokenized and turned into an AST | Parser (V8, SpiderMonkey) | Abstract Syntax Tree |
| **Execution** | AST → Bytecode → Interpreted line-by-line | Ignition (V8) | Program output |
| **Optimization** | Hot code is compiled to native machine code | TurboFan (V8) | Optimized binary |
| **Runtime** | Memory managed, async tasks coordinated | Heap + Call Stack + Event Loop | Responsive execution |

**Key takeaway:** JavaScript is **not just interpreted** — modern engines parse → compile to bytecode → interpret → JIT-compile hot paths, all at runtime. The source file is just the starting line of a multi-stage pipeline.

---

## 1. Full Comparison Table — JavaScript Engine Components

| Component | Role | What it processes | With our example |
|-----------|------|-------------------|------------------|
| **Engine** (V8, SpiderMonkey, JavaScriptCore) | The entire runtime — reads, parses, compiles, executes JS | Source code → all intermediate forms | `node 01_HelloWorld.js` launches V8 |
| **Parser** | Converts source text into structured AST | Raw characters & tokens | `console` → Identifier node, `"Hello World!"` → StringLiteral |
| **Interpreter** (Ignition in V8) | Walks the AST, emits bytecode, executes it immediately | AST → Bytecode | `LdaConstant [0]` → loads string |
| **JIT Compiler** (TurboFan in V8) | Compiles "hot" (frequently run) bytecode to native machine code | Bytecode → Optimized assembly | Not triggered for one-shot scripts |
| **Call Stack** | Tracks function calls — LIFO (Last In, First Out) | Function execution contexts | `console.log()` pushed, then popped |
| **Heap** | Where objects, closures, and strings are allocated | Object references, not primitives | The `"Hello World!"` string object lives here |
| **Garbage Collector** | Automatically frees memory no longer reachable | Heap objects with zero references | After `console.log` returns, the string is collectible |
| **Event Loop** | Coordinates sync code, callbacks, promises, timers | Tasks & microtask queues | `console.log` is sync — runs before any pending async tasks |

---

## 2. How Our Example Breaks Down

### Source Code Layer

```javascript
console.log("Hello World!");
```

This line triggers every internal component of the JavaScript engine — let's trace it.

### Stage 1: Parsing — Source → Tokens → AST

The **Parser** (specifically V8's preparser + full parser) breaks this into:

```
Tokens:
  console     →  IDENTIFIER
  .           →  PERIOD / ACCESSOR
  log         →  IDENTIFIER
  (           →  LEFT_PAREN
  "Hello World!"  →  STRING_LITERAL
  )           →  RIGHT_PAREN
  ;           →  SEMICOLON
```

These tokens become an **AST** (Abstract Syntax Tree):

```
Program
  └── ExpressionStatement
        └── CallExpression
              ├── callee: MemberExpression
              │     ├── object: Identifier ("console")
              │     └── property: Identifier ("log")
              └── arguments
                    └── StringLiteral ("Hello World!")
```

> The AST is a *tree*, not a list — it captures nesting and precedence. `console.log()` is a `CallExpression` whose callee is a `MemberExpression`.

### Stage 2: Bytecode Generation

V8's Ignition interpreter walks the AST and emits **bytecode**:

```asm
LdaConstant        r0, <"Hello World!">      // Load string constant at pool index 0
Star               r1                         // Store in register r1
LdaGlobal          r2, "console"              // Load global object `console`
Star               r3                         // Store in register r3
LdaNamedProperty   r4, r3, "log"              // Load `log` property of console
Star               r5                         // Store in register r5
Call               r6, r5, r1                 // Call the function with argument
Return                                        // Return undefined
```

This bytecode is **not human-readable at rest** — it's stored as 1-byte opcodes in memory.

### Stage 3: Interpretation

Ignition begins executing bytecode **immediately**, instruction by instruction:

1. `LdaConstant [0]` → loads `"Hello World!"` from the constant pool into an accumulator
2. `Star r1` → moves the string to register 1
3. `LdaGlobal [1]` → looks up `console` in the global object (window/globalThis)
4. `Star r2` → stores it
5. `LdaNamedProperty r2, [2]` → resolves `log` via prototype chain lookup on `console`
6. `Call r3, r1` → invokes the function

### Stage 4: Call Stack & Heap at Runtime

```
Call Stack (push/pop order):
┌─────────────────────────────────────┐
│  (top)                              │
│  console.log("Hello World!")  ← pop │  ← executing now
│  (top-level) module             ← pop│
│  (empty)                            │
└─────────────────────────────────────┘

Heap (objects allocated):
┌─────────────────────────────────────┐
│  "Hello World!" (String object)     │  ← allocated when LdaConstant runs
│  console (Object)                   │  ← already in heap (global object)
│  log (Function object)              │  ← already in heap (on console prototype)
└─────────────────────────────────────┘

After execution:
┌─────────────────────────────────────┐
│  "Hello World!" (unreachable)  ← GC │  ← garbage collected when nothing references it
│  console, log                  kept  │  ← still referenced by global scope
└─────────────────────────────────────┘
```

---

## 3. Step-by-Step Walkthrough

| Step | What happens | Component involved | Format |
|------|-------------|-------------------|--------|
| **①** | You type `node 01_HelloWorld.js` | Shell | Command |
| **②** | Node.js reads the file into memory | File system | UTF-8 text |
| **③** | V8 scans characters → produces tokens | Scanner / Lexer | Token stream |
| **④** | V8 builds the AST from tokens | Parser | Tree structure |
| **⑤** | Ignition emits bytecode from AST | Bytecode generator | Opcode list |
| **⑥** | Ignition interprets bytecode (loads `console`, resolves `log`, calls it) | Interpreter | Live execution |
| **⑦** | `"Hello World!"` string allocated in heap | Heap allocator | Memory |
| **⑧** | `console.log` function is called — prints to stdout | Runtime + C++ bindings | I/O |
| **⑨** | Function returns, call frame popped | Call Stack | — |
| **⑩** | Script ends, Node.js exits | Process | — |

> **Step ⑪ (if this were a hot loop):** TurboFan would compile the bytecode to x86 machine code, replacing interpretation with native execution. V8's Ignition and TurboFan run **side by side** — cold code stays interpreted, hot code gets JIT-compiled.

---

## 4. Pipeline Diagram

```
┌───────────────────────────────────────────────────────────┐
│           01_HelloWorld.js                                │
│     console.log("Hello World!");                          │
│                   │                                       │
│           RAW SOURCE CODE (UTF-8 text)                    │
└───────────────────┬───────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│          1. SCANNER / LEXER                               │
│     Reads characters one at a time                         │
│     "c" "o" "n" "s" "o" "l" "e" → token IDENTIFIER       │
│     "."           → token DOT                             │
│     "log"         → token IDENTIFIER                      │
│     "("           → token LPAREN                          │
│     "\"Hello World!\"" → token STRING_LITERAL             │
│     ")"           → token RPAREN                          │
│     ";"           → token SEMICOLON                       │
│                   │                                       │
│           OUTPUT: Flat token stream                        │
└───────────────────┬───────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│          2. PARSER                                        │
│     Consumes tokens → builds AST                          │
│                                                           │
│     Program                                               │
│       ExpressionStatement                                │
│         CallExpression                                    │
│           MemberExpression                                │
│             obj: Identifier("console")                    │
│             prop: Identifier("log")                       │
│           args: StringLiteral("Hello World!")             │
│                                                           │
│     Also: scope analysis, variable resolution             │
│           │                                               │
│     OUTPUT: AST (Abstract Syntax Tree)                    │
└───────────────────┬───────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│          3. INTERPRETER (V8 Ignition)                     │
│                                                           │
│     Walks AST → emits bytecode:                           │
│     ┌──────────────────────────────────────────────────┐  │
│     │  [0] LdaConstant [0]       # "Hello World!"      │  │
│     │  [2] Star r1                                      │  │
│     │  [4] LdaGlobal [1]         # "console"           │  │
│     │  [7] Star r2                                      │  │
│     │  [9] LdaNamedProperty r2, [2]  # "log"           │  │
│     │ [13] Star r3                                      │  │
│     │ [15] Call r3, r1                                   │  │
│     │ [19] Return                                       │  │
│     └──────────────────────────────────────────────────┘  │
│                                                           │
│     Interprets bytecode immediately:                      │
│     ┌─ Step through each opcode ──────────────────────┐   │
│     │ → loads "Hello World!" into register            │   │
│     │ → finds `console` in global scope               │   │
│     │ → resolves `log` on console's prototype          │   │
│     │ → calls the function                           │   │
│     │ → "Hello World!" printed                        │   │
│     └────────────────────────────────────────────────┘   │
│                                                           │
│     INTERPRETED EXECUTION (cold path)                     │
└───────────────────┬───────────────────────────────────────┘
                    │
                    │  (if code runs many times — "hot")
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│          4. JIT COMPILER (V8 TurboFan)                    │
│                                                           │
│     Profiler identifies hot bytecode                     │
│     → Compiles to optimized native machine code          │
│                                                           │
│     x86-64 assembly (simplified):                        │
│       movq rdi, [r14 + 0x18]    # load console          │
│       movq rsi, [rdi + 0x20]    # load log method       │
│       leaq rdx, [rip + str]     # load string addr      │
│       call *rsi                 # call log               │
│                                                           │
│     Machine bytes:                                       │
│     48 8b 7e 18 48 8b 77 20 48 8d 15 xx xx xx xx       │
│     ff d6                                                 │
│                                                           │
│     NATIVE EXECUTION (hot path — no interpretation)      │
└───────────────────────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│          5. RUNTIME SUPPORT                               │
│                                                           │
│     ┌──────────────┐   ┌──────────────────────────────┐  │
│     │  CALL STACK  │   │          HEAP                │  │
│     │  (LIFO)      │   │  (objects, closures, etc.)   │  │
│     │              │   │                              │  │
│     │  console.log │   │  "Hello World!" string       │  │
│     │  (top-level) │   │  console object              │  │
│     │              │   │  log function                │  │
│     └──────────────┘   └──────────────────────────────┘  │
│                                                           │
│     ┌──────────────────────────────────────────────────┐  │
│     │        EVENT LOOP (not used here — sync code)    │  │
│     │  This code is synchronous — no waiting needed.   │  │
│     │  Event loop only activates for:                  │  │
│     │  setTimeout, fetch, fs.readFile, DOM events, etc.│  │
│     └──────────────────────────────────────────────────┘  │
│                                                           │
│     ┌──────────────────────────────────────────────────┐  │
│     │        GARBAGE COLLECTOR                         │  │
│     │  After console.log returns:                      │  │
│     │  "Hello World!" string has no references → free  │  │
│     └──────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

---

## 5. Breakdown Table — What Each Internal Component "Sees"

| Component | `console` | `.` | `log` | `(` | `"Hello World!"` | `)` | `;` |
|-----------|-----------|-----|-------|-----|------------------|-----|-----|
| **Lexer** | `IDENTIFIER` token | `DOT` token | `IDENTIFIER` token | `LPAREN` token | `STRING_LITERAL` token | `RPAREN` token | `SEMICOLON` token |
| **Parser** | `MemberExpression.object` node | Structure — not a node itself | `MemberExpression.property` node | `CallExpression` structure boundary | `CallExpression.arguments[0]` node | Structure boundary | Statement end indicator |
| **Bytecode** | `LdaGlobal [1]` (index in constant pool) | — (resolved via NamedProperty) | `LdaNamedProperty r2, [2]` | — (implicit in `Call` instruction) | `LdaConstant [0]` (constant pool index) | — (implicit in Call's operand count) | — (Return handles it) |
| **Ignition Interpreter** | Global lookup in `globalThis` | Property access via prototype chain | `console.log` → built-in C++ function | Sets up call frame | Heap-allocated JS string | Tears down call frame | Advances bytecode pointer |
| **TurboFan (JIT)** | Fixed memory offset from `globalThis` base | Compiled to pointer arithmetic | Address of `log` function in V8's runtime | `call` instruction in assembly | Address in `.rodata` section of generated code | — | — |
| **Heap** | Reference to global object in heap | — | Reference to function object in heap | — | Actual string object allocated in heap | — | — |
| **Call Stack** | — | — | Stack frame pushed | Arguments prepared in registers | — | Stack frame popped | — |
| **Garbage Collector** | Alive (still reachable via global) | — | Alive (still reachable) | — | ⚠️ **Dead after return** — eligible for GC | — | — |

---

## 6. Real-World Comparison

| Component | Analogy |
|-----------|---------|
| **Source Code** | A blueprint written in English |
| **Parser** | An architect who reads the blueprint and draws up structural plans (AST) |
| **Bytecode** | Step-by-step construction instructions in a universal format |
| **Interpreter** | A worker who follows instructions one step at a time |
| **JIT Compiler** | An expert who watches the worker, notices they hammer a nail 1000 times, and builds a nail gun instead |
| **Call Stack** | A stacked tray of plates — you put a new plate on top (call a function) and remove it when done (return) |
| **Heap** | A warehouse where all materials (objects) are stored and retrieved |
| **Garbage Collector** | A cleanup crew that throws away materials nobody is using anymore |
| **Event Loop** | A receptionist who takes messages (callbacks/promises) and delivers them when the worker is free |

---

## Key Concepts

- **JavaScript is never "just interpreted"** — modern engines like V8 have a multi-stage pipeline: parse → bytecode → interpret → optionally JIT-compile
- **The source file is only the beginning** — by the time `"Hello World!"` prints, the code has passed through 4+ internal representations (text → tokens → AST → bytecode → machine code)
- **The Event Loop isn't involved in sync code** — `console.log()` runs and returns before the event loop even checks its first task
- **JIT compilation is opportunistic** — one-shot scripts never see TurboFan. Hot loops trigger it. The engine balances compile cost against execution speed
- **Garbage Collection is invisible but essential** — short scripts like this barely notice it, but long-running apps depend on the GC reclaiming memory from short-lived objects like our `"Hello World!"` string
