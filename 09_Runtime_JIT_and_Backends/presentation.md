# Compilers — Part 09: Runtimes, JITs & Modern Backends

**Part 09 — Where Code Actually Runs**

> The compiler does not stop at an executable. Between source and silicon lies a whole execution spectrum — interpreters, just-in-time compilers, tiered pipelines that re-compile hot code, speculation guarded by deoptimisation, and the runtime services (garbage collection, exceptions, dynamic linking) the compiled code leans on. We close with the shared infrastructure shaping the field: LLVM, Cranelift, GraalVM, MLIR and WebAssembly — the "new UNCOL".

`Interpret` -> `Baseline JIT` -> `Optimising JIT` -> `Deopt / OSR`

Spectrum - Interpreters - JIT & Tiering - Speculation - GC - Backends - Wasm

---

## Table of Contents

1. [Topics](#topics)
2. [The Execution Spectrum](#the-execution-spectrum)
3. [Tradeoffs at a Glance](#tradeoffs-at-a-glance)
4. [Interpreters: Tree-Walking vs Bytecode](#interpreters-tree-walking-vs-bytecode)
5. [Dispatch Dominates Interpreter Cost](#dispatch-dominates-interpreter-cost)
6. [Direct Threading & the Computed Goto](#direct-threading--the-computed-goto)
7. [Just-in-Time Compilation](#just-in-time-compilation)
8. [Tiered Execution](#tiered-execution)
9. [Warmup & the Performance Curve](#warmup--the-performance-curve)
10. [Speculation: Type Feedback & Inline Caches](#speculation-type-feedback--inline-caches)
11. [Hidden Classes / Maps / Shapes](#hidden-classes--maps--shapes)
12. [Speculation, Deoptimisation & OSR](#speculation-deoptimisation--osr)
13. [Runtime Services: Memory Management](#runtime-services-memory-management)
14. [Tracing GC Algorithms](#tracing-gc-algorithms)
15. [The Generational Heap](#the-generational-heap)
16. [Incremental & Concurrent GC](#incremental--concurrent-gc)
17. [Exception Handling & Unwinding](#exception-handling--unwinding)
18. [Dynamic Linking & Loading](#dynamic-linking--loading)
19. [Modern Backends: LLVM & GCC](#modern-backends-llvm--gcc)
20. [Cranelift, GraalVM & MLIR](#cranelift-graalvm--mlir)
21. [WebAssembly: A Portable Target](#webassembly-a-portable-target)
22. [Wasm Engines & the "New UNCOL"](#wasm-engines--the-new-uncol)
23. [Where Compilers Are Going](#where-compilers-are-going)
24. [Summary & Further Reading](#summary--further-reading)

---

## Topics

Decks 03–08 took source text to machine code. This part asks the next question: **how does that code actually run**, and what does the runtime do for it? We trace the execution spectrum, then the engines and services that make modern dynamic languages fast.

### The Execution Spectrum
- AOT vs interpretation vs JIT; hybrids
- Startup vs peak, portability, footprint
- Tree-walking vs bytecode interpreters
- Dispatch: switch, threaded, computed-goto

### JIT Compilation
- Baseline / template vs optimising JITs
- Method-based vs trace-based
- Tiered execution; the warmup curve
- V8, HotSpot, .NET pipelines

### Speculation & Feedback
- Type feedback; inline caches
- Hidden classes / maps / shapes
- Deoptimisation & on-stack replacement

### Runtime Services
- Memory management & garbage collection
- Exception handling & stack unwinding
- Dynamic linking: PLT / GOT, PIC, relocations

### Modern Backends & Wasm
- LLVM & GCC; Cranelift; GraalVM / Truffle; MLIR
- WebAssembly, WASI, the "new UNCOL"
- ML-guided optimisation; HW/SW co-design

---

## The Execution Spectrum

There is no single way to "run" a program. The same source can be **compiled ahead-of-time (AOT)** to native code, **interpreted** directly, or **compiled just-in-time (JIT)** while it runs. Most real systems are *hybrids*.

*(Diagram: a spectrum bar from "all work before run" on the left to "all work during run" on the right, with AOT, JIT and Interpret boxes placed along it.)*

| | AOT native | JIT compile | Interpret |
| --- | --- | --- | --- |
| Examples | C, C++, Rust, Go, Swift | JVM, V8, .NET, PyPy | CPython, Ruby, shells |
| + | fast start, peak perf | profile-driven, portable | instant start, tiny, portable |
| − | no runtime feedback; one target/binary | warmup, memory, ships a compiler | slowest steady state; dispatch per op |

**The core trade-off.** Work moved *earlier* (AOT) gives fast startup and predictable peak but cannot see actual inputs. Work moved *later* (JIT/interp) can specialise to observed behaviour, at the cost of warmup time and runtime footprint.

**Nobody is pure.** CPython compiles to bytecode then interprets. The JVM interprets *then* JITs. V8 has no interpreter-free mode. Even C uses a runtime (libc, the dynamic loader). The lines are practical, not absolute.

---

## Tradeoffs at a Glance

Each strategy optimises a different point on the **startup–peak–footprint–portability** surface. A serverless function and a long-running server want opposite ends of it.

| Property | AOT native | Bytecode interpreter | Tiered JIT | AOT + PGO |
| --- | --- | --- | --- | --- |
| Startup latency | instant | instant | warmup needed | instant |
| Peak throughput | high | low | highest | high |
| Memory footprint | small | small | large (code + profiles + GC) | small |
| Uses runtime feedback | no | no | yes (online) | offline only |
| Portability of artefact | one ISA/ABI | bytecode is portable | bytecode is portable | one ISA/ABI |
| Ships a compiler? | no | no | yes, at runtime | no |
| Sweet spot | CLIs, systems, FaaS | scripting, glue | servers, browsers | predictable hot paths |

**Startup vs peak is the tension.** A JIT pays a *warmup tax* before reaching peak. For short-lived processes (a Lambda invocation, a CLI run) that tax may never amortise — which is exactly why GraalVM native-image and CRaC-style snapshots exist.

**PGO: a middle path.** Profile-guided optimisation runs the program on representative input, records branch/edge profiles, then re-compiles AOT using them — offline feedback without a runtime compiler.

---

## Interpreters: Tree-Walking vs Bytecode

The simplest interpreter **walks the AST** directly. It is easy to write but slow: every node is a virtual dispatch, pointers chase all over the heap, and there is no locality. Real engines first lower the AST to a **linear bytecode** and interpret that.

**Tree-walking** — recursively `eval(node)` over the AST. Trivial to implement (teaching, config DSLs), but poor locality and one polymorphic call per node (e.g. early Ruby MRI <1.9).

**Bytecode** — compile AST → flat array of opcodes; a *dispatch loop* fetches, decodes, executes. Compact, cache-friendly, easy to JIT later (CPython, JVM, Lua, V8 Ignition, Wasm).

```c
// Stack bytecode interpreter — the classic
// fetch/decode/execute switch dispatch loop
for (;;) {
  Op op = *ip++;              // fetch
  switch (op) {               // decode + dispatch
    case OP_CONST: push(*ip++);          break;
    case OP_ADD:   { int b=pop(),a=pop();
                     push(a+b); }        break;
    case OP_JMP:   ip += (int8_t)*ip;    break;
    case OP_RET:   return pop();
  }
}
```

**Why the switch is slow.** Each iteration ends in *one* indirect branch (the switch jump). The branch predictor sees a single site whose target is effectively random — near-constant mispredicts.

---

## Dispatch Dominates Interpreter Cost

In a bytecode interpreter the *useful* work of an opcode (an add, a load) is tiny; the **dispatch** — fetch, decode, branch to the handler — dominates. Faster interpreters are mostly about cheaper dispatch and better branch prediction.

*(Diagram: switch-based dispatch funnels every handler back through one branch site the predictor can't learn; direct-threaded dispatch gives each handler its own branch site, so the predictor learns opcode-pair correlations.)*

| Technique | Mechanism | Note |
| --- | --- | --- |
| Switch dispatch | One `switch`; compiler emits a jump table | Simplest; one mispredicting branch site |
| Indirect threading | Bytecode → table of handler addresses; jump via table | Used where computed-goto is unavailable |
| Direct threading | Each handler ends in `goto *next_handler` (GCC labels-as-values) | Distinct branch per opcode → better prediction |
| Token / subroutine threading | Bytecode is a list of `call`s to handlers | Forth heritage; trades CALL/RET overhead |
| Superinstructions | Fuse common opcode *sequences* into one handler | Amortises dispatch over several ops |

---

## Direct Threading & the Computed Goto

The standard speedup: a GNU C **labels-as-values** table maps each opcode to its handler's address, and every handler ends by *jumping directly* to the next op's handler — no central switch. CPython adopted this; it gave a 15–20% interpreter win.

```c
// Direct-threaded dispatch (GCC/Clang extension)
static void *table[] = {
  [OP_CONST]=&&do_const, [OP_ADD]=&&do_add,
  [OP_JMP]=&&do_jmp,     [OP_RET]=&&do_ret,
};
#define NEXT() goto *table[*ip++]

  NEXT();                       // start dispatch
do_const: push(*ip++);  NEXT();
do_add:  { int b=pop(),a=pop();
           push(a+b); }         NEXT();
do_jmp:   ip += (int8_t)*ip;    NEXT();
do_ret:   return pop();
```

**Why it predicts better.** Each `NEXT()` is its *own* indirect branch. The hardware predictor keys on the site, so it learns correlations like "after a `LOAD` usually comes an `ADD`" — impossible with one shared switch site.

**Superinstructions.** Profile the bytecode, find hot opcode *pairs* or *triples* (e.g. `LOAD_FAST; LOAD_FAST; BINARY_ADD`), and synthesise a single fused handler. One dispatch now does three ops' work. Forth called these "compiled words".

**Modern twists.** Inline caching in bytecode (CPython 3.11+ "specialising adaptive interpreter" rewrites opcodes in place); register bytecode (Lua, Dalvik) → fewer push/pop → fewer dispatches; stack caching (keep top-of-stack in a register).

**The ceiling.** Even a perfect interpreter still *dispatches* once per op. To go faster you must remove dispatch entirely — that means compiling to native code. Enter the JIT.

---

## Just-in-Time Compilation

A JIT compiles bytecode (or a trace) to **native machine code at runtime**, then jumps into it. Because it runs *during* execution, it can specialise to the program's actual behaviour — the inputs, the types, the hot paths it has observed. The idea dates to McCarthy's LISP (1960) and Hansen's adaptive FORTRAN.

**Baseline / template JIT** — one-pass: each bytecode → a fixed template of native instructions. Almost no optimisation; compiles *fast*; goal is to kill dispatch overhead and reach native quickly (V8 Sparkplug, JSC baseline).

**Optimising JIT** — full SSA IR, inlining, GVN, LICM, register allocation (decks 06–08); uses *profile + type feedback* to specialise and speculate. Slow to compile, fast code — reserved for hot methods (V8 TurboFan, HotSpot C2, JSC FTL).

**Where does the code go?** Into an executable *code cache* — pages mapped `RWX` (or W^X with flips). The runtime patches call sites to point at compiled entries, manages the cache, and can throw code away when it deoptimises.

**Method-based vs trace-based:**
- *Method JIT* — the unit of compilation is a function/method (HotSpot, V8, .NET RyuJIT).
- *Trace JIT* — record a hot linear path through loops/calls, compile *that* trace, guard side-exits (TraceMonkey, PyPy, LuaJIT).

Traces handle tight loops superbly but struggle with branchy, megamorphic code (trace explosion). Most mainstream VMs settled on method JITs with inlining; PyPy and LuaJIT remain the great trace-based survivors.

---

## Tiered Execution

No single compiler wins on both startup and peak, so modern VMs run a **tiered pipeline**: start interpreting for instant launch, then promote hot code through progressively more optimising compilers as profiles accumulate. Counters (invocation + back-edge) trigger each promotion.

*(Diagram: Interpreter → Baseline JIT → Mid-tier JIT → Optimising JIT, with a dashed "deoptimise" arrow looping all the way back to the interpreter when speculation fails.)*

- **V8:** Ignition (interpreter) → Sparkplug (baseline) → Maglev (mid-tier) → TurboFan (optimising).
- **HotSpot:** interpreter → C1 → C2.
- **.NET:** Quick JIT / Tier-0 → optimising RyuJIT / Tier-1, plus ReadyToRun AOT.

---

## Warmup & the Performance Curve

Tiering trades a brief slow start for a far higher ceiling. Plotting throughput over time shows the classic **warmup curve**: each tier steps performance up as hot code is recompiled — with a temporary dip whenever a deopt throws optimised code away.

*(Diagram: a step curve rising interpret → baseline → mid-tier → optimising peak, then a downward "deopt dip" before re-optimising back up.)*

**Why warmup hurts.** Microbenchmarks and short jobs may finish before reaching peak, so they measure mostly warmup. Benchmark harnesses (JMH, JetStream) explicitly discard warmup iterations.

**Fighting warmup.** AOT-compile a baseline (.NET ReadyToRun, OpenJDK AOT/Leyden), snapshot a warmed heap (CRaC), or skip the JIT entirely (GraalVM native-image) — trading peak ceiling for fast, predictable start.

---

## Speculation: Type Feedback & Inline Caches

In a dynamic language `a.x` or `a + b` could mean anything — the lookup is potentially a full hash search every time. The **inline cache** (Deutsch & Schiffman, Smalltalk-80; refined in Self) caches the result of the last lookup at the call site and reuses it while the type stays the same.

*(Diagram: a site degrading from monomorphic (1 shape, guard + direct field load) → polymorphic (2–4 shapes, small linear check) → megamorphic (many shapes, full runtime lookup, no inlining).)*

An IC *degrades* as it sees more types:
- **Monomorphic** (1 type) — fastest; guard + direct field load, fully inlinable.
- **Polymorphic** (2–4 types) — a small list of (shape → offset); linear check, still fast.
- **Megamorphic** (many types) — give up; fall back to a full runtime lookup, no inlining, a deopt source.

The optimising JIT reads these states as *type feedback*: a monomorphic site can be inlined to a single guarded field load; a megamorphic one cannot.

---

## Hidden Classes / Maps / Shapes

Inline caches need a cheap, stable answer to "what type is this object?" — but dynamic objects have no declared class. The trick from **Self**: synthesise one. Objects with the same set of properties in the same order share a **hidden class** (V8 calls it a *Map*; JSC a *Structure*; SpiderMonkey a *Shape*).

```javascript
function Point(x, y) {
  this.x = x;   // transition: {} -> M1 (has x)
  this.y = y;   // transition: M1 -> M2 (has x,y)
}
let a = new Point(1, 2);   // hidden class M2
let b = new Point(3, 4);   // SAME hidden class M2  ✓

// M2 records: x @ offset 0, y @ offset 1
// IC at "p.x" specialises to: load slot 0 if map==M2

b.z = 9;   // b transitions M2 -> M3; a stays M2
           // now the .x site is polymorphic — slower
```

**What a hidden class buys.** O(1) type identity (one pointer compare); property → fixed slot offset, like a C struct; field access becomes a guarded *load*, not a hash probe.

**Transition trees.** Hidden classes form a tree of transitions keyed by "added property X". Constructing objects the same way walks the same path — reusing maps and keeping caches monomorphic.

**Why initialisation order matters.** Adding properties in different orders, or after construction, forks the transition tree into *different* hidden classes — turning monomorphic sites polymorphic. Hence: initialise all fields, in the same order, in the constructor.

**Heritage.** Maps + inline caches are the *same two ideas* from Self (Chambers, Ungar, Hölzle, 1989–91) that made dynamically-typed code competitive with statically-typed code — later carried into HotSpot and V8.

---

## Speculation, Deoptimisation & OSR

The optimising JIT *bets*: "`x` is always a small integer", "this call is always `Array.map`". It compiles fast code that assumes the bet, protected by a cheap **guard**. If a guard ever fails, it must **deoptimise** — bail out of the optimised code mid-execution, without losing the running computation.

**Deoptimisation (bailout).** A guard fails (unexpected type, overflow, map change). The VM *reconstructs* the interpreter's stack frame(s) from the optimised frame using a **deopt map**, resumes in the interpreter/baseline at the exact bytecode, and records the bad assumption so it isn't re-made.

**On-stack replacement (OSR).** The dual of deopt. A long-running loop is *already* on the stack in the interpreter when it goes hot. OSR swaps that live frame for an optimised one — entering compiled code *in the middle* of the loop. ("OSR exit" = the deopt direction.)

```javascript
function sum(arr) {
  let s = 0;
  for (let i = 0; i < arr.length; i++)
    s += arr[i];   // guard: arr is a packed
                   // Smi (small-int) array
  return s;
}
// Hot, monomorphic -> TurboFan inlines a
// tight int loop, no bounds re-check, no boxing.
//
// Later: sum([1, 2, "x"])  -> guard FAILS
//   -> DEOPT: rebuild interpreter frame,
//      resume at the "+=" bytecode, mark
//      the array element type as polymorphic.
```

**Why it's safe.** Every speculation is guarded and every optimised point has a recorded mapping back to bytecode state (locals, stack, PC). Speculation is therefore *always* recoverable — observable semantics never change.

---

## Runtime Services: Memory Management

Compiled code rarely manages memory alone. The runtime provides allocation and reclamation — either **manual** (`malloc`/`free`, RAII, ownership) or **automatic** (a garbage collector). The compiler must cooperate: it emits the metadata the collector needs.

**Manual** — `malloc`/`free`, `new`/`delete`, RAII/smart pointers (C++), ownership & borrows (Rust, checked at compile time). No pauses; bugs are leaks, use-after-free, double-free.

**Reference counting** — each object holds a count; free at zero. Prompt, incremental, simple (CPython, Swift ARC). Count traffic + atomics cost; **cannot collect cycles** — needs a backup cycle collector or weak refs.

**Tracing GC** — periodically trace from *roots* (stack, globals, registers); anything unreachable is garbage, so cycles are handled naturally. Mark-sweep, mark-compact, copying, generational. Needs *safepoints* and *stack maps* from the compiler.

**The compiler's job.** For a precise tracing GC, the compiler must tell the collector, at each safepoint, *which registers and stack slots hold live pointers* (a stack map), and emit **write barriers** on pointer stores. Without this the GC can't move or even find objects safely.

**Reference cycles.** Two objects referencing each other never reach count zero. CPython runs a separate generational cycle detector; Swift relies on the programmer marking `weak`/`unowned` references.

---

## Tracing GC Algorithms

All tracing collectors find the live set the same way; they differ in **how they reclaim and compact** — trading throughput, pause time, space overhead and fragmentation.

| Algorithm | How it works | Pros | Cons |
| --- | --- | --- | --- |
| Mark–sweep | Mark live, sweep dead onto free lists | No copying; simple | Fragmentation; sweep cost |
| Mark–compact | Mark, then slide live objects together | No fragmentation; bump allocation | Extra compaction pass; moves objects |
| Copying / semispace | Copy live from "from"-space to "to"-space (Cheney) | Fast bump alloc; compacts for free | Halves usable heap |
| Generational | Separate young/old; collect young often | Cheap, high yield on young gen | Needs write barriers + remembered set |

**The weak generational hypothesis.** *"Most objects die young."* Empirically the vast majority of allocations become garbage almost immediately. So collect the young generation frequently and cheaply (a small copying collection), and promote the rare survivors to an old generation collected rarely.

**Write barriers & remembered sets.** To collect the young gen alone, the GC must know about old→young pointers. A **write barrier** — a few instructions the compiler emits on every pointer store — records such cross-generation (or cross-region) references into a *remembered set* / card table.

---

## The Generational Heap

A typical generational layout: a small **young generation** (an Eden plus two survivor spaces for copying), and a large **old generation** for tenured survivors. New objects bump-allocate into Eden; minor GCs copy survivors and age them; long-lived objects are promoted.

*(Diagram: Eden + Survivor S0/S1 inside a "Young Generation" box, a "promote" arrow into the "Old/Tenured" box, and a remembered-set / card-table strip recording old→young pointers written by the barrier.)*

A minor GC traces the roots plus the remembered set, copies survivors from Eden and one survivor space into the other survivor space, and promotes objects that have survived enough collections.

---

## Incremental & Concurrent GC

A naive collector is **stop-the-world**: it pauses every thread while it traces. For interactive and server workloads those pauses are unacceptable, so modern collectors do most of their work *concurrently with* the running program — the headline goal is bounded, sub-millisecond pauses.

**Spreading the work.** *Incremental* — trace in small slices interleaved with mutation. *Concurrent* — collector threads run alongside mutator threads. *Parallel* — many GC threads cooperate on one phase. The *tri-colour invariant* (white/grey/black) keeps a concurrent trace correct; write barriers preserve it.

**The hard part.** While the collector traces, the program keeps mutating pointers. A barrier must catch any store that would hide a live object behind already-scanned (black) memory — otherwise the GC frees something still in use.

| Collector | Style | Target |
| --- | --- | --- |
| Java G1 | Generational, region-based, mostly-concurrent | Balanced, bounded pauses |
| Java ZGC / Shenandoah | Concurrent, load/colour-pointer barriers | Sub-ms pauses, huge heaps |
| Go GC | Concurrent mark-sweep, non-generational | Low latency, simple |
| V8 Orinoco | Generational, parallel + concurrent | Short main-thread pauses |
| .NET | Generational (0/1/2), bg concurrent | Server & workstation modes |

**The trade triangle.** Throughput, latency (pause time) and footprint form a trilemma — you tune two at the third's expense. ZGC buys tiny pauses with barrier overhead and memory; a simple STW copying GC maximises throughput but pauses.

---

## Exception Handling & Unwinding

Exceptions need the runtime to abandon the current computation and transfer control to a handler further up the stack — **unwinding** intervening frames and running their cleanup. How that is implemented determines whether the *non*-throwing path costs anything.

**Table-driven ("zero-cost") EH.** The normal path executes with *no* extra instructions; the compiler emits side **unwind tables** (`.eh_frame` / DWARF CFI, LSDA). On `throw`, a personality routine walks the tables to find a matching handler and runs destructors. Throwing is *expensive*; not throwing is free (C++, Rust, Swift).

**setjmp / longjmp.** Each `try` saves machine state with `setjmp`; `throw` = `longjmp` back. There is a cost on the *normal* path (the save) — not zero-cost. Portable; older/constrained implementations.

**Stack unwinding.** Walking frames from the throw point up to the handler, running each frame's cleanup (C++ destructors, Rust `Drop`, `finally`). The unwinder reads the same CFI tables a debugger uses to reconstruct frames.

**Stack maps & safepoints (ties to GC).** The very same precise-frame metadata serves the collector. At a **safepoint**, a **stack map** tells the GC which slots are live pointers — so it can find and move objects. Threads are brought to safepoints to be paused consistently.

**Wasm aside.** Core Wasm long lacked exceptions; the *exception-handling proposal* (`try`/`catch`/`throw` with tags) now standardises them for compiled C++/Swift targets.

---

## Dynamic Linking & Loading

Deck 08 emitted an object file with **relocations**. At load time the dynamic linker (`ld.so`) finishes the job: it maps shared libraries, resolves cross-module symbols, and patches addresses — ideally *lazily*, only on first use.

```x86asm
; A call to an external function via the PLT
    call    printf@PLT
; printf@PLT (the stub):
printf_plt:
    jmp     *printf@GOT(%rip)   ; 1st call: GOT
                                ; still points back
                                ; into the resolver
    push    $reloc_index        ; which symbol
    jmp     _dl_runtime_resolve ; ld.so resolves,
                                ; rewrites the GOT,
                                ; then tail-calls printf
; 2nd call onward: jmp *GOT goes straight to printf
```

**PLT + GOT = lazy binding.** The **PLT** (Procedure Linkage Table) holds call stubs; the **GOT** (Global Offset Table) holds resolved addresses. First call routes through `ld.so` which fills the GOT slot; every later call is a single indirect jump.

**Position-independent code (PIC).** A shared library can load at *any* address, so code addresses globals *relative* to itself (RIP-relative on x86-64) and goes through the GOT. Required for ASLR and `.so`/`.dylib` sharing across processes.

**Relocations.** Entries telling the loader "patch the address here once you know where the symbol landed". Resolved at link time (static), load time (e.g. `R_X86_64_RELATIVE`), or first use (PLT/GOT, lazy).

**Static vs dynamic.** Static linking bakes everything into one binary — fast start, larger, no shared updates. Dynamic linking shares libraries across processes and allows security patching, at the cost of load-time resolution. `RELRO` + eager binding hardens the GOT.

---

## Modern Backends: LLVM & GCC

Writing a production backend (instruction selection, scheduling, register allocation, every ISA) is enormous. The field's answer: **reusable backend infrastructure**. A new language emits a shared IR and inherits decades of optimisation and dozens of targets — the N×M argument from deck 06, realised.

**LLVM (Lattner, 2003).** SSA-based **LLVM IR** as a common currency. Frontends: Clang, Rust, Swift, Julia, Zig. Backends: x86-64, AArch64, RISC-V, Wasm, GPUs. Library-structured — embeddable as a JIT (ORC) too.

**GCC (1987, Stallman).** GENERIC → GIMPLE (SSA) → RTL pipeline. Many frontends (C/C++/Fortran/Ada/Go/D/Rust-gccrs); broadest target coverage of any compiler; historically monolithic, now exposes `libgccjit`.

**Why it reshaped the field.** Before LLVM, every language paid the backend tax alone. Sharing the optimiser and code generator let small teams ship serious languages (Rust, Swift, Julia, Zig) and let researchers prototype passes against a real, multi-target backend.

**Trade-offs of a big backend.** Superb generated code but *slow* to compile; heavy and ill-suited as a low-latency JIT (WebKit FTL eventually moved off LLVM to in-house B3). That demand for *fast* backends leads to the next slide.

---

## Cranelift, GraalVM & MLIR

LLVM optimises hard but compiles slowly. A wave of newer infrastructure targets different points: **fast** codegen, **language-agnostic** JITs, and **extensible, multi-level** IRs.

**Cranelift.** Fast, secure code generator (Bytecode Alliance). Own SSA IR (CLIF) designed for *compile speed*. Wasmtime's baseline + rustc's debug-build backend (`rustc_codegen_cranelift`). The right tool where LLVM is too slow.

**GraalVM / Truffle.** Write an AST interpreter on the **Truffle** framework; Graal *partially evaluates* it → an optimising JIT for free. This is the **first Futamura projection** in production. Also **native-image**: closed-world AOT to a small, fast-start binary.

**MLIR.** Multi-Level IR: *dialects* at many abstraction levels, with progressive lowering (e.g. `linalg` → `affine` → `llvm`). Born in TensorFlow; now the substrate for ML and hardware compilers. Lets domains share infrastructure instead of reinventing it.

**Futamura projections (1971).** Specialising an interpreter to a fixed program yields a *compiled* program (1st projection). Specialising the specialiser to the interpreter yields a *compiler* (2nd). GraalVM/Truffle makes the 1st projection a practical, language-agnostic JIT — one optimiser, many guest languages.

**The through-line.** All four — LLVM, Cranelift, Graal, MLIR — are bets that **compiler infrastructure should be shared**. The frontier is no longer one compiler per language but reusable, composable layers that many languages and accelerators plug into.

---

## WebAssembly: A Portable Target

**WebAssembly** (Wasm, 2017) is a compact, portable bytecode for a **stack machine** — designed as a *compile target*, not a language to write by hand. It is fast to validate, fast to compile, sandboxed by construction, and deterministic. C/C++/Rust/Go/Swift all emit it.

```
;; A Wasm function (text format) — stack machine
(func $sum (param $n i32) (result i32)
  (local $acc i32) (local $i i32)
  (loop $L
    (local.set $acc
      (i32.add (local.get $acc) (local.get $i)))
    (local.set $i
      (i32.add (local.get $i) (i32.const 1)))
    (br_if $L                       ;; structured
      (i32.lt_s (local.get $i)      ;; control flow:
                (local.get $n))))   ;; no raw jumps
  (local.get $acc))
```

**Structured control flow.** No arbitrary `goto` — only nested `block` / `loop` / `if` with labelled breaks (`br`, `br_if`). This guarantees a reducible CFG, makes single-pass validation trivial, and lets baseline engines compile in one fast linear sweep.

**Design goals.** Portable (one binary on any compliant engine); safe (linear memory sandbox, validated types, no raw host pointers); fast (near-native, streaming compile while downloading); compact and deterministic.

**Beyond the browser.** WASI — a capability-based syscall interface so Wasm runs server-side, in edge/CDN functions, plugins, databases. The Component Model — typed, language-neutral interfaces between modules. Ongoing proposals: threads, SIMD, GC, tail-calls, EH.

---

## Wasm Engines & the "New UNCOL"

A Wasm module is itself executed by an engine that re-uses everything in this deck: an **interpreter**, a **baseline JIT**, an **optimising JIT** — or an AOT compile. Many source languages converge on Wasm, and many engines run it: Wasm is increasingly the portable waypoint McCarthy & Strachey called **UNCOL** in 1958.

*(Diagram: many source languages (C/C++, Rust, Go/Swift, AssemblyScript) fan in to a single `.wasm` module, which fans out to many engines: V8 Liftoff/TurboFan, Wasmtime + Cranelift, Wasmer/WAMR/wasm3.)*

M languages → 1 portable target → N engines — the UNCOL dream, finally working.

---

## Where Compilers Are Going

The runtime story keeps moving. A few directions visible from here — the common thread is **co-design**: compilers, runtimes, and hardware increasingly evolve together rather than over a fixed interface.

**ML-guided optimisation.** Learned heuristics for inlining, unrolling, register allocation; MLGO (LLVM) replaces hand-tuned cost models with trained policies; superoptimisation & learned phase ordering.

**Shared multi-level IR.** MLIR dialects unifying ML, HPC, hardware and general compilers; progressive lowering instead of one monolithic IR; verified / formally-checked passes (CompCert lineage).

**HW / SW co-design.** Compilers for GPUs, TPUs, NPUs, FPGAs and dataflow accelerators; domain-specific architectures need domain-specific compilers; Wasm + the Component Model as a universal deployment substrate.

**The constant.** From A-0 and FORTRAN to TurboFan and MLIR, the job has not changed: *bridge the gap between how humans want to express computation and how machines actually execute it* — and keep moving work to whichever moment (compile, link, load, run) makes the program fastest, smallest or safest.

---

## Summary & Further Reading

### Key Takeaways
- Execution is a spectrum: AOT ↔ JIT ↔ interpret, trading startup vs peak vs footprint — real systems are hybrids.
- Interpreter cost is *dispatch*; threaded code & superinstructions attack it.
- Tiered JITs interpret first, then recompile hot code through baseline → optimising tiers.
- Speculation = type feedback + inline caches + hidden classes, made safe by **deopt** and **OSR**.
- GC variants (ref-counting, mark-sweep/compact, copying, generational, concurrent) need compiler-emitted stack maps & write barriers.
- EH, unwinding, PLT/GOT & PIC are runtime services the compiler feeds.
- Shared infra (LLVM, Cranelift, Graal, MLIR) & Wasm are the modern backend story.

### Further Reading
- Jones, Hosking & Moss — *The Garbage Collection Handbook*
- Aycock — *A Brief History of Just-in-Time* (ACM CSUR, 2003)
- Hölzle & Ungar — Self: type feedback & deoptimisation papers
- Smith & Nair — *Virtual Machines*
- The V8 blog (Ignition, Sparkplug, Maglev, TurboFan, Orinoco)
- LLVM, Cranelift, MLIR & the WebAssembly spec docs

### Exercises
- Convert a switch interpreter to computed-goto; measure the win.
- Add an inline cache to a toy property-lookup interpreter.
- Write a Cheney semispace copying collector.
- Compile a C function to Wasm and read the `.wat`.

---

`Part 09 ✓` -> `Part 10: Building a Mini-Compiler`

**Next → Part 10: Building a Mini-Compiler** — tying the whole series together: lexer, parser, types, IR, optimisation and codegen in one small, working compiler.
