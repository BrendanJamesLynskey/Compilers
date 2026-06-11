# Compilers — Part 02: The Compiler Pipeline

**Part 02 — From Source Text to Running Code: the Architecture**

> A map of the whole journey a program takes — the analysis/synthesis split, the seven classic phases, the intermediate representations that flow between them, and the wider toolchain of driver, assembler, linker and loader that turns text into a process.

`Lex` -> `Parse` -> `Sema` -> `IR` -> `Opt` -> `Codegen`

Analysis - Synthesis - Narrow Waist - Toolchain

---

## Table of Contents

1. [Topics](#topics)
2. [What Is a Compiler?](#what-is-a-compiler)
3. [Compiler vs Interpreter vs ...](#compiler-vs-interpreter-vs-)
4. [Two Halves: Analysis & Synthesis](#two-halves-analysis--synthesis)
5. [The Classic Phases, in Order](#the-classic-phases-in-order)
6. [Front-End Phases](#front-end-phases)
7. [Middle / Back-End Phases](#middle--back-end-phases)
8. [The Pipeline — Representations Flowing Through](#the-pipeline--representations-flowing-through)
9. [Worked Trace — int x = a + b * 2;](#worked-trace--int-x--a--b--2)
10. [Worked Trace — the Code at Each Stage](#worked-trace--the-code-at-each-stage)
11. [Why the Split? The N×M Narrow Waist](#why-the-split-the-nm-narrow-waist)
12. [Who Owns What](#who-owns-what)
13. [Single-Pass vs Multi-Pass](#single-pass-vs-multi-pass)
14. [The Wider Toolchain](#the-wider-toolchain)
15. [Linking & Loading](#linking--loading)
16. [Bootstrapping & Self-Hosting](#bootstrapping--self-hosting)
17. [Cross-Compilation & Target Triples](#cross-compilation--target-triples)
18. [The Phase-Ordering Problem](#the-phase-ordering-problem)
19. [Error-Handling Philosophy](#error-handling-philosophy)
20. [The Representations, End to End](#the-representations-end-to-end)
21. [How the Series Maps onto the Pipeline](#how-the-series-maps-onto-the-pipeline)
22. [Summary & Further Reading](#summary--further-reading)

---

## Topics

This deck **frames the whole series**. Every later part zooms into one box on the pipeline; here we lay out the map — the phases, the representations between them, and the tools around the compiler proper.

### What a Compiler Is

- The translation problem & the classic definition
- Compiler vs interpreter vs transpiler
- AOT, JIT and hybrid bytecode + VM

### The Two Halves

- Analysis (front end) vs synthesis (back end)
- The middle end in between
- The seven classic phases, in order

### Representations & Trace

- Tokens -> AST -> IR -> optimised IR -> asm
- A worked trace of `int x = a + b * 2;`
- The N×M "narrow waist" argument

### Toolchain & Beyond

- Driver, preprocessor, assembler, linker, loader
- Single- vs multi-pass; bootstrapping; cross-compilation
- Phase ordering & error-handling philosophy

---

## What Is a Compiler?

A **compiler** is a program that reads a program in one language — the **source** — and translates it into an equivalent program in another language — the **target** — reporting any errors it finds along the way. *Equivalent* means same observable behaviour, not same text.

```
Source  --->  Compiler  --->  Target
(C, Rust)   (analyse +      (x86-64 asm)
             synthesise)
                |
                v
           error reports
```

### Two Obligations

- **Correctness** — preserve the meaning of the source
- **Efficiency** — make the target fast & small

### Why "Compile"?

Grace Hopper's **A-0** (1952) *compiled* — assembled — subroutines from a library. The name stuck even as the job grew into full translation.

The target need not be machine code: a compiler may emit assembly, bytecode, or another high-level language. The defining act is **meaning-preserving translation across a language gap**.

---

## Compiler vs Interpreter vs ...

"Compiler" is one point in a family of **language processors**. The axes that distinguish them: *when* translation happens, and *what* the target is.

| Kind | What it does | When | Example |
|------|--------------|------|---------|
| **Compiler (AOT)** | Source -> machine code, ahead of execution | Build time | GCC, Clang, rustc, Go |
| **Interpreter** | Executes source / AST directly, no separate target | Run time | CPython (tree-walk-ish), Ruby MRI |
| **Transpiler** | Source -> another *high-level* language (source-to-source) | Build time | TypeScript->JS, Babel, f2c |
| **Bytecode + VM** | Source -> bytecode, then a VM interprets it | Build + run | javac -> JVM, CPython -> CPython VM |
| **JIT** | Compiles hot bytecode to machine code *during* execution | Run time | HotSpot C2, V8 TurboFan, PyPy |
| **Hybrid** | AOT to bytecode + JIT/interpret tiers at run time | Both | .NET CLR, modern JVMs, JS engines |

### Compile vs Interpret — a Spectrum

The line is blurry. CPython *compiles* to `.pyc` bytecode then interprets it; a JIT *interprets* first then compiles hot paths. The question is really "how much work happens before the first instruction runs?"

### Trade-offs

- **AOT**: fast steady-state, no warm-up, but no run-time knowledge
- **JIT**: adapts to profiles & the host CPU, but pays warm-up & memory
- **Interpret**: instant start, portable, but slow per-op

---

## Two Halves: Analysis & Synthesis

The Dragon Book splits a compiler into an **analysis** half (the *front end*) that breaks the source apart and a **synthesis** half (the *back end*) that builds the target up. A **middle end** sits between, working on a language- and target-neutral IR.

```
Analysis · Front End  ->  Middle End  ->  Synthesis · Back End
lex/parse/sema            IR optimise      instr-sel/alloc/sched
"understand source"       "improve prog"   "emit good machine code"
language-dependent        language- &      target-dependent
                          target-neutral
```

### Front End Outputs

A checked AST or IR plus a populated symbol table. It has decided the program is *well-formed*.

### Middle End

Rewrites the IR into a faster, smaller, equivalent IR. Knows neither syntax nor registers.

### Back End Outputs

Target assembly or object code, tuned to one ISA's registers, addressing modes and pipeline.

---

## The Classic Phases, in Order

Conventionally seven phases. The first three are the front end (analysis); IR generation bridges; the last three are the middle/back end (synthesis). Each consumes the previous phase's output and produces a richer representation.

`Lexical` -> `Syntax` -> `Semantic` -> `IR Gen` -> `MI Optimise` -> `Code Gen` -> `Target Opt`

| Phase | In -> Out |
|-------|-----------|
| 1. Lexical analysis | chars -> token stream |
| 2. Syntax analysis | tokens -> parse tree / AST |
| 3. Semantic analysis | AST -> annotated (typed) AST |
| 4. IR generation | AST -> intermediate code |
| 5. Machine-independent optimisation | IR -> better IR |
| 6. Code generation | IR -> target asm |
| 7. Target-specific optimisation | asm -> tuned asm |

### Cross-Cutting

Two services thread through *every* phase: the **symbol table** (names -> attributes) and the **error handler** (diagnostics & recovery).

---

## Front-End Phases

### 1 · Lexical Analysis

- **In:** raw characters. **Out:** a stream of *tokens* (lexeme + class).
- Strips whitespace/comments; groups `int`, `x`, `=` into kinds. Driven by regular expressions -> a DFA scanner. *(Part 03.)*

### 2 · Syntax Analysis

- **In:** token stream. **Out:** a parse tree / AST.
- Checks the tokens form a valid sentence of the grammar (CFG). LL/LR parsing builds the tree that captures structure. *(Part 04.)*

### 3 · Semantic Analysis

- **In:** AST. **Out:** annotated AST + symbol table.
- Resolves names, checks types, scopes, declarations. Adds the meaning the grammar alone can't enforce. *(Part 05.)*

### Why the Order?

Each phase needs its predecessor's *guarantee*: you can't parse without tokens, can't type-check without structure, can't generate code without a meaning. The split also localises errors — a lexer error is a stray character, a parser error a malformed phrase, a semantic error a type mismatch.

---

## Middle / Back-End Phases

### 4 · IR Generation

- **In:** typed AST. **Out:** intermediate code (e.g. *three-address code*, SSA).
- Lowers tree-shaped meaning into a linear, machine-independent form — explicit temporaries, one operation per instruction. *(Part 06.)*

### 5 · Machine-Independent Optimisation

- **In:** IR. **Out:** better IR.
- Constant folding, common-subexpression elimination, dead-code elimination, loop-invariant code motion — all on the neutral IR. *(Part 07.)*

### 6 · Code Generation

- **In:** optimised IR. **Out:** target asm / object code.
- Instruction selection (tree-tiling/BURS), register allocation (graph-colouring or linear-scan), instruction scheduling. *(Part 08.)*

### 7 · Target-Specific Optimisation

- **In:** asm. **Out:** tuned asm.
- Peephole rewrites, addressing-mode tricks, branch alignment, micro-arch scheduling — things that only make sense once the ISA is fixed.

Phases 4–5 belong to the **middle end**; 6–7 to the **back end**. The boundary is exactly where target details enter the picture.

---

## The Pipeline — Representations Flowing Through

What actually *moves* between phases is a sequence of representations. The **symbol table** and **error handler** are cross-cutting — every phase reads and writes them.

```
Symbol Table  (names -> type · scope · location · storage)
  |        |        |        |          |         |        |
Lexer -> Parser -> Semantic -> IR Gen -> Optimiser -> Codegen -> asm
chars   tokens    AST         ann.AST    IR          opt.IR     out
  |        |        |        |          |         |        |
Error Handler  (detect · report · recover — across all phases)

[--- front end ---][--- middle end ---][--- back end ---]
```

---

## Worked Trace — `int x = a + b * 2;`

Let's push one statement through the whole machine and watch its representation change at each stage. Assume `a`, `b` are `int` locals already in scope.

```
① Tokens  ->  ② AST  ->  ③ 3-Address Code  ->  ④ x86-64
```

Precedence (`*` binds tighter than `+`) is captured by the *shape* of the AST, not the token order.

---

## Worked Trace — the Code at Each Stage

**① Token stream** — lexemes tagged with a class:

```text
KW(int) ID(x) OP(=) ID(a) OP(+) ID(b) OP(*) NUM(2) SEMI
```

**② / ③ AST (semantic-checked)** — precedence in the tree shape; types resolved:

```text
Decl(int, x)
└─ Assign
   └─ Add :int
      ├─ a :int
      └─ Mul :int
         ├─ b :int
         └─ 2 :int
```

**④ Three-address code** — linear, one op each:

```text
t1 = b * 2
t2 = a + t1
x  = t2
```

**⑤ x86-64** — after instruction selection, allocation & `*2 -> +` strength reduction:

```x86asm
mov   eax, DWORD PTR [rbp-8]   ; load b
add   eax, eax                 ; b * 2
add   eax, DWORD PTR [rbp-4]   ; + a
mov   DWORD PTR [rbp-12], eax  ; store x
```

### Notice

Three temporaries in IR collapsed to *one* register (`eax`); the multiply by 2 became an add. That's the back end and optimiser earning their keep.

---

## Why the Split? The N×M Narrow Waist

The front/middle/back split exists for **reuse**. With *N* source languages and *M* targets, a monolithic design needs **N×M** compilers. Route everything through one shared **IR** and you need only **N + M** halves.

```
C/C++  ─┐                              ┌─ x86-64
Rust   ─┤                              ├─ ARM64
Swift  ─┼──>   [ LLVM IR + optimiser ] ─┼─> RISC-V
Fortran─┤      (shared, written once)  ├─ WASM
Julia  ─┘                              └─ PowerPC

  N front ends                            M back ends
```

This is the architecture **LLVM** (Lattner, 2003) made mainstream: a stable IR as a *contract* so any front end pairs with any back end. The middle-end optimiser, written *once*, benefits all N×M combinations.

---

## Who Owns What

The split is also a **contract about knowledge**: each part knows exactly enough and no more, which is what lets the pieces be swapped.

### Front End Knows

- The *source language*: its grammar, type system, scoping rules
- Nothing about registers or the target ISA
- Produces a verified IR + symbol table

### Middle End Knows

- Only the *IR* and its semantics
- Dataflow, control flow, algebraic identities
- Neither syntax nor hardware — fully reusable

### Back End Knows

- The *target*: registers, addressing modes, calling convention, pipeline
- Nothing about which source language produced the IR
- Produces assembly / object code

### The Pay-off

Add a new language -> write one front end, get every target free. Add a new chip -> write one back end, get every language free. Improve the optimiser -> everyone benefits. This is why GCC and LLVM are *ecosystems*, not single programs.

---

## Single-Pass vs Multi-Pass

A **pass** is one complete traversal of the program. Phases are *logical*; passes are *physical*. One pass may run several phases; one phase may span several passes.

### Single-Pass

- Reads the source once, emitting code as it goes
- Early compilers were single-pass out of **necessity** — memory was tiny; you couldn't hold the whole program
- Forces language design: Pascal/C require *declare before use* so a name's type is known when first seen
- Fast, but limited optimisation & no forward references

### Multi-Pass

- Builds an in-memory IR (AST/SSA) and revisits it repeatedly
- Each optimisation is typically its own pass over the IR
- Enables whole-function and whole-program analysis
- The norm today — memory is cheap; LLVM runs *dozens* of passes in a configurable pipeline

C's header/forward-declaration culture and the `;`-before-use rules are fossils of the single-pass era. Modern languages (Rust, Go) declare freely because their compilers hold the whole module in memory.

---

## The Wider Toolchain

"The compiler" in everyday use is really a **driver** orchestrating several programs. `gcc hello.c` silently runs a preprocessor, the compiler proper, an assembler and a linker.

```
[ Driver: gcc/clang orchestrates these ............ ]
Preprocessor -> Compiler -> Assembler -> Linker     -> Loader -> Running
.c -> .i        .i -> .s    .s -> .o     .o+libs->exe   exe->proc  in memory
(cpp)           (cc1)       (as)         (ld)           (OS,exec)
```

### Object Files

The assembler emits *relocatable* object files (ELF on Linux, Mach-O on macOS, COFF/PE on Windows): machine code plus a symbol table and relocation entries the linker resolves.

### See It Yourself

```bash
gcc -E hello.c   # stop after preprocess
gcc -S hello.c   # stop after compile → .s
gcc -c hello.c   # stop after assemble → .o
gcc    hello.c   # all the way to a.out
```

---

## Linking & Loading

The **linker** stitches object files and libraries into one image, resolving every cross-reference. The **loader** (part of the OS) maps that image into a process and starts it.

### Static Linking

- Library code is *copied into* the executable at link time (`.a` archives)
- Self-contained, no runtime deps, larger binary
- A library fix means relinking everything

### Dynamic Linking

- Only a *reference* is stored; the shared object (`.so`/`.dll`/`.dylib`) is loaded at run time
- Smaller binaries, shared in memory across processes, patchable
- The dynamic loader (`ld.so`) resolves symbols, often lazily via the PLT/GOT

### The Loader's Job

Read the ELF program headers; `mmap` the text/data segments; set up the stack, heap, and `argv`/`env`; perform relocations and dynamic-symbol binding; transfer control to `_start` -> `main`. Only now is your program a *running process* — the destination this whole series is heading toward.

---

## Bootstrapping & Self-Hosting

A **self-hosting** compiler is written in the very language it compiles (GCC in C, rustc in Rust, Go's compiler in Go). The chicken-and-egg of building the first one is **bootstrapping**.

The classic **T-diagram** records three languages of a compiler: *Source* (left arm), *Target* (right arm), *Implementation* (stem). To self-host, compile your compiler with itself.

### The Bootstrap Path

- Write the compiler in an *existing* language (or a subset)
- Use it to compile a version written in the *new* language
- That binary now recompiles itself — self-hosting achieved
- Each release is built by the previous release (stage-0 -> stage-1 -> stage-2)

### Trusting Trust

Ken Thompson's 1984 Turing lecture, *Reflections on Trusting Trust*: a compiler binary could inject a backdoor into programs *and* into future copies of itself — invisibly, even with clean source. You cannot fully trust code you didn't build from a trusted compiler.

---

## Cross-Compilation & Target Triples

A **cross-compiler** runs on one machine but emits code for another — essential for embedded, mobile and bringing up new hardware. Three machines matter, named by convention.

| Role | Meaning |
|------|---------|
| **Build** | Where the compiler is *compiled* |
| **Host** | Where the compiler *runs* |
| **Target** | What the compiler *emits code for* |

A native compiler has build = host = target. A cross-compiler has host ≠ target. A *Canadian cross* has all three different.

### The Target Triple

```text
x86_64 - unknown - linux  - gnu
  │         │         │       │
 arch     vendor     OS      ABI/libc
```

Other examples: `aarch64-apple-darwin`, `riscv64gc-unknown-linux-gnu`, `wasm32-unknown-unknown`, `thumbv7em-none-eabihf` (bare-metal Cortex-M).

```bash
rustc --target aarch64-unknown-linux-gnu
clang --target=riscv64-linux-gnu  main.c
```

The triple selects the back end, the ABI, and which sysroot/libc to link against.

---

## The Phase-Ordering Problem

Optimisations are not independent: running one can *enable* or *disable* another. There is no universally optimal order — this is the **phase-ordering problem**, a teaser for Part 07.

### Optimisations Interact

- **Inlining** exposes constants -> enables **constant folding**
- Folding can make a branch dead -> enables **dead-code elimination**
- DCE shrinks a loop -> may enable *further* inlining — a cycle
- But aggressive inlining can *bloat* code and hurt the instruction cache

### How Real Compilers Cope

- Fixed, hand-tuned pass pipelines (LLVM's `-O2` schedule)
- Running key passes *multiple times* (e.g. instcombine repeatedly)
- Fixed-point iteration until no pass changes anything
- Research: ML-guided & autotuned pass orders

The takeaway for this overview: the pipeline is *not* a clean conveyor belt in the middle end — it loops, repeats, and is heuristically scheduled. Part 07 dives in.

---

## Error-Handling Philosophy

A compiler that stops at the first error is a poor tool. Good compilers **detect many errors per run**, locate them precisely, and recover to keep parsing — errors are categorised by the phase that catches them.

### Lexical Errors

Illegal characters, unterminated strings, malformed numbers. Caught by the scanner. `@` in C, `"abc` with no close quote.

### Syntactic Errors

Token sequence violates the grammar — a missing `;`, unbalanced `)`. Caught by the parser, which drives most recovery.

### Semantic Errors

Grammatically valid but meaningless: undeclared name, type mismatch, wrong arg count. Caught in semantic analysis.

### Recovery Strategies

- **Panic-mode** — skip tokens to a synchronising token (`;`, `}`)
- **Phrase-level** — locally patch (insert a missing `;`)
- **Error productions** — grammar rules for common mistakes

### Good Diagnostics Matter

Modern compilers (Clang, Rust) set the bar: precise spans with carets, the *actual* vs expected type, suggested fixes, and avoiding a cascade of bogus follow-on errors. A diagnostic is a UX surface.

---

## The Representations, End to End

Stepping back, a compiler is a sequence of **data structures**, each closer to the machine than the last. Knowing which representation you're in tells you what's easy and what's hard.

| Representation | Form | Good for | Phase |
|----------------|------|----------|-------|
| Character stream | bytes / text | — | input |
| Token stream | flat list of (class, lexeme) | regular structure | after lexing |
| Parse tree / AST | tree | nesting, precedence, scope | after parsing |
| Annotated AST | tree + types/symbols | type rules, name resolution | after sema |
| IR (3-addr / SSA) | linear / CFG of basic blocks | dataflow, optimisation | middle end |
| Asm / object code | instructions + relocations | scheduling, allocation | back end |
| Executable / image | linked segments | loading & running | linker/loader |

Each later part of this series lives in one or two rows of this table. Keep it as your map.

---

## How the Series Maps onto the Pipeline

Now you have the map, here's the route. Each remaining deck is one box, plus the runtime that all of this ultimately feeds.

### Front End

- **03** Lexical Analysis — the scanner & DFAs
- **04** Syntax Analysis — grammars & parsing
- **05** Semantic Analysis & Types

### Middle End

- **06** Intermediate Representations
- **07** Optimisation (incl. phase ordering)

### Back End & Runtime

- **08** Code Generation
- **09** Runtime, JIT & Backends

### Capstone

- **10** Building a Mini Compiler — assemble all the pieces end to end

### Already Behind Us

- **01** History of Compilers
- **02** This deck — the pipeline & toolchain

---

## Summary & Further Reading

### Key Takeaways

- A compiler is meaning-preserving translation across a language gap
- Compiler/interpreter/transpiler/JIT differ by *when* & *what* they translate
- Analysis (front) + middle + synthesis (back) — the IR is the boundary
- Seven phases: lex -> parse -> sema -> IR -> optimise -> codegen -> target-opt
- Symbol table & error handler cut across every phase
- The N×M narrow waist (LLVM) is why the split exists: N+M, not N×M
- The toolchain extends beyond the compiler: driver, assembler, linker, loader
- Bootstrapping, cross-compilation, phase ordering & diagnostics frame later decks

### Further Reading

- Aho, Lam, Sethi & Ullman — *Compilers: Principles, Techniques & Tools* (the Dragon Book), ch. 1
- Cooper & Torczon — *Engineering a Compiler*, ch. 1
- Appel — *Modern Compiler Implementation*
- Ken Thompson — *Reflections on Trusting Trust* (CACM, 1984)
- The LLVM project docs — the IR & pass pipeline
- Levine — *Linkers and Loaders*

### Mental Model to Keep

Source text becomes a series of ever-lower representations, each enabling work the previous couldn't — until the loader makes the last one a running process.

`Part 02 done` -> `Part 03: Lexical Analysis`
