# Compiler vs Interpreter

> **Example:** `console.log("Hello World!");` — same source, two execution strategies

---

## TL;DR

| Layer | Compiler (e.g., GCC, Rustc) | Interpreter (e.g., Node.js/V8, Python) |
|-------|---------------------------|----------------------------------------|
| **Strategy** | Translates **all** source to machine code **before** running | Translates source **one line at a time** while running |
| **Output** | A standalone executable (`.exe`, `.out`) | No permanent output — executes directly |
| **Speed** | Faster execution (pre-compiled binary) | Slower execution (translation happens live) |
| **Analogy** | Translating an entire book before reading it | Interpreting a speech sentence-by-sentence as a person speaks |

**Key takeaway:** A compiler is **offline translation** — it converts the whole program to machine code upfront, then runs it. An interpreter is **live translation** — it reads, translates, and executes each line in one step.

---

## 1. Full Comparison Table

| Aspect | Compiler | Interpreter |
|--------|----------|-------------|
| **Definition** | Translates the entire source program into machine code **before execution** | Translates and executes source code **line by line** at runtime |
| **Phases** | All phases (lexing, parsing, code gen, optimization) happen **ahead of time** | Lexing & parsing happen live; code gen often skipped in favor of direct execution |
| **Output** | Produces a standalone binary file (`.exe`, `.elf`, `.out`) | No persistent output — result is the program's execution itself |
| **Execution speed** | **Fast** — binary runs directly on the CPU | **Slower** — translation overhead every time a line runs |
| **Startup time** | **Slower** — must compile fully before anything runs | **Faster** — execution begins immediately after parsing the first line |
| **Error detection** | Finds **all** syntax/type errors at compile time (before any execution) | Finds errors **one at a time** — stops at the first error on the line it's executing |
| **Portability** | Binary is **platform-specific** — must recompile for x86 vs ARM vs RISC-V | Source is **portable** — run anywhere with the right interpreter installed |
| **Distribution** | Ship the binary — user doesn't need any toolchain | User **must have the interpreter** installed on their machine |
| **Optimization** | Can spend time optimizing the whole program (loop unrolling, inlining, etc.) | Limited optimization per line; newer interpreters use JIT to bridge the gap |
| **Examples** | GCC, Clang/LLVM, Rustc, TypeScript (tsc), Go compiler | Node.js (V8), Python (CPython), Ruby (MRI), Bash |
| **Languages** | C, C++, Rust, Go, Zig — typically compiled | JavaScript, Python, Ruby, Bash, PHP — typically interpreted |

---

## 2. How Our Example Breaks Down

### The Common Starting Point

```javascript
console.log("Hello World!");
```

This same line of code takes a **radically different journey** depending on whether it's fed to a compiler or an interpreter.

### Compiler Path (imagine this were C)

```
┌────────────────────────────────────────────────────────────┐
│            SOURCE CODE (hello.c)                            │
│            console.log("Hello World!");                     │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   1. PREPROCESSING                                         │
│   Resolves #includes, #defines, expands macros             │
│   Output: "translation unit" — pure C ready to compile     │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   2. COMPILATION (Frontend)                                │
│   Lexing:    `console` → identifier token                  │
│              `.` → dot operator token                      │
│              `log` → identifier token                      │
│              `(` → open paren                              │
│              `"Hello World!"` → string literal token       │
│              `)` → close paren                             │
│              `;` → semicolon                               │
│                                                             │
│   Parsing:   Builds AST (Abstract Syntax Tree)             │
│   ┌───── CallExpression ─────────────────────────┐         │
│   │  ┌── MemberAccess ───────┐  ┌─ Arg ────────┐ │         │
│   │  │ Identifier("console") │  │ StringLiteral│ │         │
│   │  │ Identifier("log")     │  │ "Hello World!"│ │         │
│   │  └───────────────────────┘  └──────────────┘ │         │
│   └──────────────────────────────────────────────┘         │
│                                                             │
│   Semantic Analysis: Checks types, scopes, resolves         │
│   `console` → find `log` is a callable → OK                │
│                                                             │
│   Output: Annotated AST + Symbol Tables                     │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   3. COMPILATION (Middle-end / Optimization)                │
│   - Constant folding                                        │
│   - Dead code elimination                                   │
│   - Inline expansion                                        │
│   Output: Optimized Intermediate Representation (IR)        │
│   e.g., LLVM IR, GIMPLE                                     │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   4. COMPILATION (Backend / Code Generation)                │
│   Lowers IR → Assembly for target CPU                      │
│                                                             │
│   x86-64 assembly (simplified):                             │
│   .section .rodata                                         │
│   .LC0:                                                    │
│     .string "Hello World!"                                 │
│   .text                                                    │
│   movl   $.LC0, %edi        # load string address          │
│   call   puts               # call the print function      │
│   movl   $0, %eax           # return 0                     │
│   ret                                                      │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   5. ASSEMBLY                                              │
│   Assembler converts assembly → relocatable machine code   │
│   Output: hello.o (object file)                            │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   6. LINKING                                               │
│   Links hello.o with libraries (libc, etc.)                │
│   Resolves `puts` → links to libc implementation           │
│   Output: a.out (final executable binary)                  │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   7. EXECUTION                                             │
│   $ ./a.out                                                │
│   Hello World!                                             │
│   CPU runs the binary directly — no source needed anymore  │
└────────────────────────────────────────────────────────────┘
```

### Interpreter Path (Node.js / V8 for JavaScript)

```
┌────────────────────────────────────────────────────────────┐
│            SOURCE CODE (01_HelloWorld.js)                   │
│            console.log("Hello World!");                     │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   1. LEXING (on-the-fly)                                   │
│   Reads characters one at a time, produces tokens          │
│   `console` → identifier                                   │
│   `.` → dot                                                │
│   `log` → identifier                                       │
│   `(` → open paren                                         │
│   `"Hello World!"` → string literal                        │
│   `)` → close paren                                        │
│   `;` → semicolon                                          │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   2. PARSING (on-the-fly)                                  │
│   Builds AST immediately                                   │
│   CallExpression(MemberAccess(console, log), ["Hello..."]) │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   3. BYTECODE GENERATION (V8 Ignition)                     │
│   AST → platform-independent bytecode                      │
│                                                             │
│   LdaConstant [0]     # "Hello World!"                     │
│   Star r1              # store it                          │
│   LdaGlobal [1]       # "console"                          │
│   Star r2              # store it                          │
│   LdaNamedProperty r2, [2]  # .log                         │
│   Star r3              # store it                          │
│   Call r3, r1         # call the function                  │
│   Return              # return undefined                   │
└───────────────────┬────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────┐
│   4. INTERPRETATION                                        │
│   V8 Ignition interprets each bytecode instruction:        │
│                                                             │
│   [0] → load "Hello World!" into a register                │
│   [1] → find global object "console"                       │
│   [2] → look up property "log" on that object              │
│   [3] → call it with the string                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐  │
│   │  "Hello World!" printed to console                  │  │
│   └─────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘

> **No separate binary is produced.** The interpreter reads, translates, and
> executes in a single live flow. Stop it mid-way and there's no artifact.
```

---

## 3. Step-by-Step Walkthrough

| Step | What Happens | Compiler | Interpreter |
|------|-------------|----------|-------------|
| **①** | You write the source | `console.log("Hello World!");` | `console.log("Hello World!");` |
| **②** | You invoke the tool | `gcc hello.c -o hello` | `node 01_HelloWorld.js` |
| **③** | Lexing | Scans the whole file into tokens | Scans first line into tokens |
| **④** | Parsing | Builds the full AST | Builds the AST for this line only |
| **⑤** | Analysis/Checks | Checks all types & semantics across all functions | Minimal — checks types at call time |
| **⑥** | Translation | Lowers AST → IR → Assembly → Machine code | Lowers AST → Bytecode only |
| **⑦** | Output | Writes `hello` (binary executable) | No file output — stays in memory |
| **⑧** | Execution | `./hello` → CPU runs the binary directly | V8 interprets bytecode → "Hello World!" printed |
| **⑨** | After execution | Binary persists on disk; can run again instantly | No artifact remains; must re-run `node` to execute again |
| **⑩** | Errors | All errors reported at once before anything runs | First error stops execution immediately |

---

## 4. Pipeline Diagram

```
                    SAME SOURCE CODE
                    console.log("Hello World!");
                            │
                            │
            ┌───────────────┴───────────────┐
            │                               │
            ▼                               ▼
┌───────────────────────┐   ┌───────────────────────────────┐
│   COMPILER PATH       │   │   INTERPRETER PATH            │
│   (e.g., GCC)         │   │   (Node.js / V8)              │
├───────────────────────┤   ├───────────────────────────────┤
│                       │   │                               │
│   Preprocessing       │   │   Offline          Live       │
│   ↓                   │   │   ────────         ────       │
│   Lexing (ALL)        │   │                               │
│   ↓                   │   │   Lexing (line 1)             │
│   Parsing (ALL)       │   │   ↓                           │
│   ↓                   │   │   Parsing (line 1)            │
│   Analysis (ALL)      │   │   ↓                           │
│   ↓                   │   │   Bytecode Gen (line 1)       │
│   Optimization (ALL)  │   │   ↓                           │
│   ↓                   │   │   ◄──── EXECUTE line 1 ────►  │
│   Code Gen (ALL)      │   │       "Hello World!"          │
│   ↓                   │   │                               │
│   Assembly + Link     │   │   (next line... if any)       │
│   ↓                   │   │                               │
│   ┌──────────┐        │   │                               │
│   │  a.out   │        │   │                               │
│   │ (binary) │        │   │                               │
│   └──────────┘        │   │                               │
│       │               │   │                               │
│       ▼               │   │                               │
│   $ ./a.out           │   │                               │
│   "Hello World!"      │   │                               │
│                       │   │                               │
│   ┌─────────────────┐ │   │   ┌─────────────────────────┐ │
│   │  BINARY PERSISTS│ │   │   │  NO PERSISTENT ARTIFACT │ │
│   │  Re-runs without│ │   │   │  Must re-interpret each │ │
│   │  recompiling    │ │   │   │  time you run it        │ │
│   └─────────────────┘ │   │   └─────────────────────────┘ │
└───────────────────────┘   └───────────────────────────────┘
```

---

## 5. Breakdown Table — What Each Layer "Sees"

| Layer | `console` | `.` | `log` | `(` | `"Hello World!"` | `)` | `;` |
|-------|-----------|-----|-------|-----|------------------|-----|-----|
| **Source (both)** | Identifier | Member access op | Method name | Open paren | String literal | Close paren | Statement end |
| **Compiler Lexer** | Token `IDENTIFIER` | Token `DOT` | Token `IDENTIFIER` | Token `LPAREN` | Token `STRING_LITERAL` | Token `RPAREN` | Token `SEMICOLON` |
| **Compiler AST** | `MemberAccess.obj` | — (structural) | `MemberAccess.prop` | — (CallStructure) | `CallExpression.args[0]` | — | — (implicit) |
| **Compiler IR** | `%1 = alloca ptr` (alloc for obj) | `%2 = getelementptr %1` (offset calc) | `%3 = load ptr, %2` (load func ptr) | `call void %3(%4)` (call instr) | `%4 = alloca [13 x i8]` (string alloc) | — | — |
| **Compiler Binary** | Memory addr in .data section | Offset encoded in instruction | Call target address (.text) | `call` opcode | Address in .rodata section | — | `ret` opcode |
| **Interpreter Lexer** | Token `IDENTIFIER` | Token `DOT` | Token `IDENTIFIER` | Token `LPAREN` | Token `STRING_LITERAL` | Token `RPAREN` | Token `SEMICOLON` |
| **Interpreter AST** | `MemberAccess.obj` | — (structural) | `MemberAccess.prop` | — (CallStructure) | `CallExpression.args[0]` | — | — |
| **Interpreter Bytecode** | `LdaGlobal [1]` | — (resolved in property access) | `LdaNamedProperty r2, [2]` | — (implicit in `Call`) | `LdaConstant [0]` | — | — (bytecode is control-flow based) |
| **Interpreter Runtime** | Looks up `console` in global scope at call time | Property access resolved by prototype chain | Finds `log` on `console.prototype` | Pushes call frame | String object in V8 heap | Pops call frame | Moves to next bytecode |

---

## 6. Real-World Comparison

| Dimension | Compiler | Interpreter |
|-----------|----------|-------------|
| **Analogy** | A translator who reads the **whole book** first, translates it, and hands you a finished copy to read anytime | A translator at the UN who listens to a speech and interprets each sentence live — you can follow along but there's no written record |
| **Time to first word** | Slow — must wait for full translation before reading page 1 | Fast — starts interpreting the moment the speaker opens their mouth |
| **Ongoing speed** | Fast — the reader just opens the finished copy | Slow — the interpreter must keep up with every sentence in real time |
| **Error handling** | The translator says "this sentence is grammatically broken" before translating anything | The interpreter stops mid-speech the first time they hit a sentence they can't parse |
| **Portability** | The finished book is in one language — only speakers of that language can read it | The live interpreter adapts to the listener — same speech, different audience? Different language output |
| **Re-running** | Open the book again — instant | Must call the interpreter back and have the speaker repeat everything |
| **Optimization** | The translator can find better phrasing across chapters (global optimization) | The live interpreter must translate each sentence as it comes (local optimization only) |

---

## 7. Hybrid World: JIT Compilation (Best of Both)

Modern runtimes like V8 (JavaScript), JVM (Java), and CPython (Python) blur the line:

```
Compiler                            Interpreter
    │                                    │
    │     ┌─────────────────────────┐    │
    │     │    JIT Compilation      │    │
    │     │                         │    │
    │     │  Source → Bytecode      │    │
    │     │          │              │    │
    │     │  ┌───────┴───────┐      │    │
    │     │  │               │      │    │
    │     │  ▼               ▼      │    │
    │     │ Interpreter   Compiler  │    │
    │     │ (cold code)   (hot code)│    │
    │     │               │        │    │
    │     │               ▼        │    │
    │     │         Machine Code   │    │
    │     └─────────────────────────┘    │
    │                                    │
```

- Start as an **interpreter** (fast startup, slow execution)
- Profile which code runs most often ("hot paths")
- Switch to **compiler** for those paths (slow setup, fast execution)

This is exactly what V8 does — it interprets bytecode via Ignition, and when `console.log` gets called millions of times in a loop, it compiles that path via TurboFan to native machine code.

---

## Key Concepts

- **Compilers** trade **startup time for execution speed** — slow to start, fast to run
- **Interpreters** trade **execution speed for startup time and portability** — instant to start, slower to run
- **Pure interpreters** don't produce a binary artifact — source code is the executable
- **Pure compilers** produce a standalone binary — source code is not needed at runtime
- **JIT compilers** hybridize — interpret at startup, compile hot paths to native code on the fly
- **JavaScript is technically interpreted** (no separate compile step), but modern V8 uses JIT internally, making it a **compiled-at-runtime** language
