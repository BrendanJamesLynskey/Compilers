# Compilers — Part 08: Code Generation

**Part 08 — The Back End: Lowering IR to Real Machine Code**

> The back end turns optimised, machine-independent IR into a concrete instruction stream for a real ISA. Three classic, interacting subproblems dominate — choosing instructions (selection), mapping unbounded virtual values onto finite registers (allocation), and ordering instructions to keep the pipeline busy (scheduling) — all bound by the target's calling convention. We work each one, then walk a function from IR to x86-64 and AArch64.

`Optimised IR` -> `Select` -> `Allocate` -> `Schedule` -> `Emit`

Selection - Allocation - Scheduling - ABI - Emission

---

## Table of Contents

1. [Topics](#topics)
2. [The Back End's Job](#the-back-ends-job)
3. [Target Architecture Primer](#target-architecture-primer)
4. [x86-64 · AArch64 · RISC-V](#x86-64--aarch64--risc-v)
5. [Instruction Selection — the Problem](#instruction-selection--the-problem)
6. [Selection Approaches — a Spectrum](#selection-approaches--a-spectrum)
7. [Tree Tiling — Covering an Expression](#tree-tiling--covering-an-expression)
8. [LLVM: SelectionDAG & GlobalISel](#llvm-selectiondag--globalisel)
9. [Register Allocation — the Problem](#register-allocation--the-problem)
10. [Allocation as Graph Colouring](#allocation-as-graph-colouring)
11. [Chaitin–Briggs: Simplify, Spill, Select](#chaitinbriggs-simplify-spill-select)
12. [Coalescing & Live-Range Splitting](#coalescing--live-range-splitting)
13. [Linear-Scan Allocation](#linear-scan-allocation)
14. [SSA-Based Allocation & PBQP](#ssa-based-allocation--pbqp)
15. [Instruction Scheduling — Hazards](#instruction-scheduling--hazards)
16. [List Scheduling over the Dependence DAG](#list-scheduling-over-the-dependence-dag)
17. [Pressure Tension & Software Pipelining](#pressure-tension--software-pipelining)
18. [Calling Conventions & the ABI](#calling-conventions--the-abi)
19. [Stack Frames, Prologue & Epilogue](#stack-frames-prologue--epilogue)
20. [How the ABI Constrains the Allocator](#how-the-abi-constrains-the-allocator)
21. [Worked Example — Source & IR](#worked-example--source--ir)
22. [Worked Example — the Assembly](#worked-example--the-assembly)
23. [Emission — Text vs Object Code](#emission--text-vs-object-code)
24. [Relocations & the Linker Handoff](#relocations--the-linker-handoff)
25. [Summary & Further Reading](#summary--further-reading)

---

## Topics

The back end is where portability ends and the silicon begins. By the close you will have walked optimised IR all the way to **x86-64** and **AArch64** assembly, naming every instruction and register chosen.

### Targets & Selection
- The back end's job; three interacting subproblems
- RISC vs CISC; x86-64 / AArch64 / RISC-V primer
- Instruction selection: tiling, BURS, SelectionDAG, GlobalISel

### Register Allocation
- Live ranges, interference graphs
- Graph colouring: Chaitin–Briggs; spilling & coalescing
- Linear scan; SSA-based allocation; PBQP

### Scheduling & ABI
- Pipeline hazards, latencies, list scheduling
- Scheduling vs register pressure; software pipelining
- System V AMD64 ABI; stack frames; prologue/epilogue

### Putting It Together
- A worked function: IR → x86-64 & AArch64
- Emission: assembly text vs the LLVM MC layer
- Relocations & the assembler/linker handoff

---

## The Back End's Job

Optimisation (Part 07) left us with clean, machine-independent IR — unbounded virtual registers, abstract operations, no notion of the target. **Code generation** lowers this to a concrete instruction stream: real opcodes, real registers, a real stack frame, obeying a real ABI. Three subproblems dominate — and they *interact*, which is what makes the back end hard.

The pipeline is roughly: optimised IR → instruction selection → register allocation → scheduling → emission → assembly/object code.

**The phase-ordering dilemma.** Selection fixes which instructions exist — and so which registers they demand. Allocation may then insert *spill* code, creating new loads and stores that selection never saw. Scheduling reorders to hide latency — but spreading live ranges apart raises register pressure, fighting the allocator. No order is globally optimal; real compilers iterate and approximate.

---

## Target Architecture Primer

The ISA dictates everything downstream. A **register** is a small, fast, named store inside the CPU; an **addressing mode** is a rule for computing an operand's memory address. The deepest split is **RISC** vs **CISC**.

**RISC — Reduced**
- Fixed-width instructions, simple to decode
- *Load/store* architecture: ALU ops touch registers only
- Few, orthogonal addressing modes
- Many registers (32+); regular encoding
- ARM64, RISC-V, MIPS, POWER, SPARC

**CISC — Complex**
- Variable-length instructions (x86: 1–15 bytes)
- Memory operands on arithmetic: `add [rax], rbx`
- Rich addressing: `base + index*scale + disp`
- Fewer architectural registers (16 GPRs)
- x86-64 is CISC — but decodes internally to RISC-like µops

ISA choice reshapes the back end: CISC's memory operands make instruction selection matter more (one instruction can do a load + an add); RISC's load/store regularity makes scheduling and register pressure the main game.

---

## x86-64 · AArch64 · RISC-V

The three ISAs a modern back end is most likely to target.

| Property | x86-64 | AArch64 (ARM64) | RISC-V (RV64) |
|---|---|---|---|
| Class | CISC | RISC (load/store) | RISC (load/store) |
| GP registers | 16 (RAX…R15) | 31 + XZR/SP | 32 (x0=zero … x31) |
| Instr. width | 1–15 bytes (variable) | 4 bytes fixed | 4 bytes (+ 2-byte "C") |
| Mem operands on ALU | yes | no | no |
| Addressing | base+index*scale+disp | base+offset, pre/post-index | base+12-bit disp only |
| Flags register | RFLAGS (implicit) | NZCV (explicit, opt-in) | none — compare-and-branch |
| Conv. arg registers | RDI, RSI, RDX, RCX, R8, R9 | X0–X7 | a0–a7 (x10–x17) |

- **x86-64**: dense code, but variable-length decode and few registers stress the allocator. Two-operand form (`dst` is also a source) forces extra copies.
- **AArch64**: three-operand, 31 GPRs, clean encoding — the allocator breathes easy. `x31` reads as zero or means SP by context.
- **RISC-V**: minimal, extensible base (RV64I) plus optional extensions (M, A, F, D, C, V). `x0` is hard-wired zero — a free constant.

---

## Instruction Selection — the Problem

**Instruction selection** maps IR operations to target instructions. It is rarely one-to-one. The mapping is both **many-to-one** (a multiply-by-4 plus add becomes a single `lea`) and **one-to-many** (a 64-bit divide may expand to a sequence). The selector must find a *covering* of the IR by available instructions — ideally the cheapest one.

Many-to-one example: IR `t1 = i*4; t2 = b + t1; v = load t2` folds the whole address calculation *and* the load into one x86 instruction:

```x86asm
mov rax, [rbx + rcx*4]   ; one instr
```

One-to-many: a high-level op (128-bit add, `select`, saturating arithmetic) may have no single instruction and expand to several.

The goal: cover the IR DAG/tree completely, every node accounted for, choosing tiles (instructions) that minimise total cost (≈ latency × frequency, or code size). This is the *tiling* problem.

---

## Selection Approaches — a Spectrum

| Approach | Idea | Code quality | Notes |
|---|---|---|---|
| Macro / template expansion | Emit a fixed sequence per IR op, independently | poor | Trivial; misses cross-op opportunities. Needs heavy peephole |
| Tree pattern matching (tiling) | Cover the expression tree with tiles; maximal munch | good | Greedy; pick the largest matching tile at each node |
| BURS / DP tiling | Dynamic programming over the tree; bottom-up rewrite | optimal (per tree) | iburg, BEG, lburg — cost-minimal tiling, linear time |
| DAG-based selection | Match over the DAG, sharing common sub-expressions | better than trees | NP-hard in general; LLVM's SelectionDAG |

**Maximal munch** — top-down greedy: at each node pick the *largest* tile that matches, recurse on the leaves it left. Fast, good code; not guaranteed optimal because a locally-large tile can force costlier choices below.

**BURS — bottom-up rewrite** — two passes: a bottom-up labelling pass computes the minimum cost to produce each non-terminal at each node (DP), then a top-down pass reads off the optimal tiles. *iburg* (Fraser, Hanson, Proebsting) generates such selectors from a cost-annotated grammar.

---

## Tree Tiling — Covering an Expression

Take `a[i] = a[i] + 1` where `a` is a 4-byte array at base `rbx` and `i` is in `rcx`. The address tree `MEM(+ (base, * (i, 4)))` can be covered by tiles. On x86-64, a single addressing mode swallows the whole sub-tree.

*(Diagram: the expression tree `MEM → + → {rbx, * → {rcx, 4}}`, with a dashed tile T1 outlining the entire address sub-tree, matched to the `[rbx + rcx*4]` addressing mode.)*

A naive tiler would emit a `lea` to compute the address, a load, an add and a store:

```x86asm
mov  rax, [rbx + rcx*4]
add  rax, 1
mov  [rbx + rcx*4], rax
```

A good selector recognises the whole address sub-tree as one addressing-mode tile — and on x86 may fold the increment into a single memory-destination instruction:

```x86asm
add  qword [rbx + rcx*4], 1   ; one CISC instruction: load+add+store
```

---

## LLVM: SelectionDAG & GlobalISel

Real selectors operate on a **DAG**, not a tree, so common sub-expressions are shared. LLVM has shipped two selectors.

**SelectionDAG (classic)**
- Builds a DAG per basic block from LLVM IR
- Legalises types & operations the target can't handle natively
- Pattern-matches via target `.td` TableGen rules + custom C++
- Schedules the DAG into a linear `MachineInstr` list
- Block-local only — cannot select across BB boundaries

**GlobalISel (newer)**
- A pipeline of passes: *IRTranslator → Legalizer → RegBankSelect → InstructionSelect*
- Operates on Generic MIR (gMIR) — whole-function, not per-block
- Faster compiles (no separate DAG build); better debuggability
- Default for AArch64 at `-O0`; growing elsewhere

**Peephole as cleanup.** Whatever the selector, a final peephole pass scans short instruction windows and rewrites local patterns — killing redundant moves, folding `mov`+`add` into `lea`, replacing `mul`-by-constant with shifts, removing `xor reg,reg` redundancies.

---

## Register Allocation — the Problem

The IR has *unbounded* virtual registers; the machine has **k** physical ones (16 on x86-64, 31 on AArch64). **Register allocation** maps virtual registers onto physical registers, and when there are not enough, **spills** some values to memory (stack slots).

- **Live range / live interval.** A value is *live* from its definition to its last use. Its live range is the set of program points where it holds a needed value. Two values *interfere* if their live ranges overlap — they cannot share a register.
- **Spilling.** When pressure exceeds k, a value is kept in a stack slot and loaded/stored around each use. The allocator picks *which* values to spill to minimise that cost.
- **Liveness** (from Part 07) is computed by backward dataflow: a variable is live at a point if some path forward uses it before redefining it.
- **Caller- vs callee-saved.** The ABI splits registers: callee-saved survive a call (preferred for values live across calls); caller-saved must be spilled by the caller if needed after a call. The allocator is ABI-aware.

---

## Allocation as Graph Colouring

Build the **interference graph**: one node per value, an edge between any two that interfere. Assigning registers so that no two interfering values share one is exactly **graph k-colouring** — colour the nodes with k colours (registers) so no edge joins two same-coloured nodes. Chaitin (1981) made this the standard formulation.

*(Diagram: a 5-node interference graph {a,b,c,d,e} and its 3-colouring — a→RAX, b and c→RBX, d and e→RCX, where non-interfering pairs share a register.)*

- **k = number of registers.** If the graph is k-colourable, every value fits in a register.
- **It's NP-complete.** General graph k-colouring is NP-complete (Chaitin's reduction). Allocators use heuristics — simplify, spill, select — not exact optimisation.
- **Pre-coloured nodes.** Some values are pinned to specific registers (ABI args in RDI/RSI…, the `div` result in RAX/RDX). These appear as pre-coloured nodes the rest must avoid.

---

## Chaitin–Briggs: Simplify, Spill, Select

The classic colouring allocator works by **graph simplification**. The key lemma: a node with *fewer than k* neighbours (low-degree) can always be coloured *after* its neighbours, so remove it and recurse.

**The algorithm**
1. **Build** the interference graph
2. **Simplify**: repeatedly remove a node of degree < k, push it on a stack
3. **Spill**: if only high-degree nodes remain, mark one a *potential spill*, remove it, continue
4. **Select**: pop nodes off the stack, giving each a colour free among its (already-coloured) neighbours
5. If a potential spill finds no free colour → *actual spill*; insert spill code & restart

- **Chaitin (1981)**: a node that must be spilled is spilled immediately and the graph rebuilt.
- **Briggs *optimistic* colouring (1994)**: a high-degree node is *optimistically* pushed onto the stack anyway. At select time it might still find a free colour — because several neighbours often share colours. Only if no colour is free does it actually spill. This colours many graphs Chaitin would have spilled.
- **Spill cost**: choose the candidate minimising `cost / degree`, where cost ≈ Σ (use/def frequency, loop-depth weighted). Spill cheap, rarely-used, high-degree values.

---

## Coalescing & Live-Range Splitting

Two refinements make colouring allocators competitive: removing pointless copies (**coalescing**) and breaking up troublesome ranges (**splitting**).

**Coalescing — move elimination.** If `y = mov x` and `x`,`y` don't otherwise interfere, give them the *same* register and the move vanishes (merge their nodes). But aggressive coalescing raises degree and can make a colourable graph un-colourable. Hence *conservative* coalescing:
- **Briggs**: coalesce only if the merged node has < k neighbours of significant degree
- **George**: coalesce if every neighbour of `x` already interferes with `y` or is low-degree

**Live-range splitting** is the opposite: split one long live range into several shorter ones connected by copies. A value can then live in a register where pressure is low and be spilled only across the busy region — instead of spilling the whole range.

The balance: coalescing *merges* ranges (fewer moves, higher degree); splitting *divides* them (more moves, lower degree, easier colouring). The *iterated register coalescing* of George & Appel (1996) interleaves both.

---

## Linear-Scan Allocation

Graph colouring is powerful but slow. JITs and `-O0` need speed. **Linear scan** (Poletto & Sarkar, 1999) skips the graph entirely.

**The idea**
- Approximate each value's live range by one *live interval* — `[start, end]` over a linearised instruction numbering
- Sort intervals by start point
- Sweep left-to-right; keep an *active* set of intervals currently holding registers
- At each new interval, expire intervals that have ended (free their registers)
- If a register is free, assign it; else spill the interval with the furthest *end*

*(Diagram: five intervals v1–v5 drawn as horizontal bars over time; at one cut, four overlap, so with k=3 one must spill — pick the furthest-ending.)*

**Where used:** HotSpot's client (C1) compiler, early V8, many JITs and Wasm engines. Linear in practice; near-optimal for straight-line-heavy code. Extended forms (Wimmer–Mössenböck) handle holes and SSA.

---

## SSA-Based Allocation & PBQP

A beautiful result reshaped allocation theory: the interference graph of a program *in SSA form* is **chordal** — and chordal graphs are colourable in polynomial time.

**Why SSA helps**
- In strict SSA, every value has one definition; live ranges align with the dominator tree
- The interference graph is *chordal* (every cycle > 3 has a chord)
- Optimal colouring of a chordal graph is **polynomial** — a perfect elimination order exists
- The hard part decouples: colour the chordal graph, then handle spilling and φ-related copies separately

**The catch:** the chromatic number equals the maximum register pressure (max simultaneously-live values), but *spilling* to get pressure below k, and resolving φ-copies on SSA destruction, remain NP-hard. SSA tames colouring, not the whole problem.

**PBQP allocation** casts allocation as a *Partitioned Boolean Quadratic Problem*: each value gets a cost vector over its candidate registers/spill, and each interference edge a cost matrix; minimise total cost. It naturally models irregular ISAs — register classes, aliasing (AX/AL), pairing constraints — and is an alternative LLVM allocator (`-regalloc=pbqp`).

**LLVM's default: Greedy** — a global, priority-ordered allocator with live-range splitting and eviction, neither pure colouring nor pure linear scan, but fast and high-quality.

---

## Instruction Scheduling — Hazards

Modern CPUs are **pipelined**: an instruction takes several cycles, and the next can start before the previous finishes — unless a **hazard** stalls it. **Scheduling** reorders instructions to hide latency and keep functional units busy, without changing the result.

- **Data hazard**: an instruction needs a result not yet ready (e.g. `add` after a cache-missing `load`). RAW (read-after-write) is the common true dependence.
- **Structural hazard**: two instructions want the same functional unit (one divider, one load port) in the same cycle.
- **Control hazard**: a branch's direction isn't known until late; the pipeline may fetch wrong instructions. Mitigated by branch prediction.

**Latencies differ:** integer add ~1 cycle, L1-hit load ~4, multiply ~3, divide 20–40. The scheduler knows the target's latency table and issues independent work into the shadow of a slow instruction.

**In-order vs out-of-order:** an out-of-order core (most desktop x86/ARM) reorders dynamically, so scheduling matters less — but still helps. An in-order core (many embedded, some Atom/A55) depends critically on a good static schedule.

---

## List Scheduling over the Dependence DAG

Within a basic block, build the **data-dependence DAG**: nodes are instructions, edges are dependences (true RAW, plus anti/output and memory ordering), weighted by latency. **List scheduling** greedily picks, each cycle, a ready instruction (all predecessors done) of highest priority — usually longest path to the end (critical path).

*(Diagram: five-node DAG — two independent loads i1, i2 feed a multiply i3, which feeds an add i4 and a store i5; the critical path i1→i3→i4 = 4+3+1 = 8 cycles.)*

**Why reorder?** Issue `i1` and `i2` (the two independent loads) back-to-back so their 4-cycle latencies overlap; by the time `i3` needs both, they're ready. A naive in-order emit would stall twice.

**Priority = critical path.** Among ready instructions, prefer the one on the longest remaining path to the block's end. Ties broken by latency, then by register-pressure impact.

**Scope:** local (one block) is standard; *superblock* / *trace* scheduling crosses blocks along hot paths for more freedom.

---

## Pressure Tension & Software Pipelining

**Scheduling vs register pressure.** Moving an instruction's definition *earlier* (to overlap latency) lengthens its live range — more values live at once — raising register pressure and risking spills. The two phases pull opposite ways:
- Schedule too eagerly → spill code negates the latency win
- Schedule too timidly → pipeline stalls
- Modern back ends use *pressure-aware* schedulers, or schedule twice (before allocation to expose parallelism, after to clean up spill code)

**Software pipelining.** For loops, overlap iterations: start iteration *i+1*'s independent work while *i* finishes. The body becomes a steady-state *kernel* bracketed by a *prologue* and *epilogue*.
- *Modulo scheduling* finds an *initiation interval* (II) — the cycle gap between successive iteration starts
- II is bounded by resources (ResMII) and recurrences (RecMII)
- Crucial on in-order VLIW/DSP targets; Itanium relied on it

On big out-of-order cores the hardware already overlaps iterations dynamically, so software pipelining's payoff is largest on in-order and explicitly-parallel machines.

---

## Calling Conventions & the ABI

The **ABI** (Application Binary Interface) is the contract that lets separately-compiled functions interoperate: who passes arguments where, who preserves which registers, how the stack is laid out. The back end must obey it exactly.

**System V AMD64 ABI (Linux/macOS/BSD)**
- Integer/pointer args: `RDI, RSI, RDX, RCX, R8, R9` — then the stack
- FP/vector args: `XMM0–XMM7`
- Return value: `RAX` (and `RDX` for 128-bit / second word)
- Stack must be **16-byte aligned** at the point of a `call`
- The **red zone**: 128 bytes below `RSP` a leaf function may use without adjusting RSP

**Caller- vs callee-saved (SysV)**

| Caller-saved (volatile) | Callee-saved (preserved) |
|---|---|
| RAX RCX RDX | RBX RBP |
| RSI RDI | R12 R13 |
| R8 R9 R10 R11 | R14 R15 |

**Other ABIs differ:** Windows x64 uses RCX, RDX, R8, R9 + 32-byte *shadow space*, no red zone. AArch64 AAPCS64 passes in X0–X7, returns in X0; X19–X28 callee-saved. Same idea, different registers.

---

## Stack Frames, Prologue & Epilogue

Each call gets a **stack frame**: storage for spills, locals, callee-saved registers and outgoing arguments. The **prologue** sets it up; the **epilogue** tears it down. The stack grows *down* on x86-64.

*(Diagram, high→low addresses: caller's frame, stack args, return address (pushed by `call`), saved RBP (RBP points here), saved callee regs, local variables, spill slots, outgoing args (RSP), then the 128-byte red zone for leaves.)*

**Prologue**
```x86asm
push rbp           ; save caller's RBP
mov  rbp, rsp      ; set frame pointer
sub  rsp, 48       ; reserve locals+spills (keep 16-byte aligned)
push rbx           ; save callee-saved used
```

**Epilogue**
```x86asm
pop  rbx           ; restore callee-saved
mov  rsp, rbp      ; or: leave
pop  rbp
ret
```

**RBP vs RSP-relative.** A frame pointer (RBP) gives stable offsets & easy unwinding; `-fomit-frame-pointer` frees RBP as a GPR and addresses locals off RSP — one more register for the allocator.

---

## How the ABI Constrains the Allocator

The calling convention is not advisory — it injects hard constraints straight into register allocation.

**Pinned registers at calls**
- Argument values *must* land in RDI, RSI, RDX… before a `call` — pre-coloured nodes
- The return value *must* be in RAX after
- A `call` *clobbers* all caller-saved registers: model it as defining (killing) them

**Values live across calls** should live in a *callee-saved* register (it survives the call for free) — or be spilled. The allocator weighs the one-off cost of saving a callee-saved register in the prologue against repeated caller-saved spills.

**Returning structs**
- Small structs (≤ 16 bytes) returned in `RAX:RDX` (or XMM pair) per SysV field classification
- Large structs: caller allocates space, passes a hidden pointer in `RDI` (*sret*); the other args shift right

**Variadic & alignment.** Variadic calls set `AL` = number of vector args used. The 16-byte alignment at `call` forces the prologue's `sub rsp` to round up.

---

## Worked Example — Source & IR

```c
int sumsq(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i] * a[i];
    return s;
}
```

Optimised SSA-ish IR (loop body):

```llvm
loop:
  %i   = phi i32 [0, %entry], [%i.n, %loop]
  %s   = phi i32 [0, %entry], [%s.n, %loop]
  %p   = getelementptr i32, i32* %a, i32 %i
  %x   = load i32, i32* %p
  %sq  = mul i32 %x, %x
  %s.n = add i32 %s, %sq
  %i.n = add i32 %i, 1
  %c   = icmp slt i32 %i.n, %n
  br i1 %c, label %loop, label %exit
```

What the back end must decide: which instructions realise `a[i]`, the multiply, the add; which registers hold `a`, `n`, `i`, `s`, the loaded element; and where to load `a[i]` only once (CSE already shared it).

---

## Worked Example — the Assembly

Args arrive per the ABI: **x86-64** `a`→RDI, `n`→ESI, return in EAX; **AArch64** `a`→X0, `n`→W1, return in W0.

**x86-64 (System V)**
```x86asm
sumsq:
    xor   eax, eax        ; s = 0  (return reg)
    xor   ecx, ecx        ; i = 0
    test  esi, esi        ; n <= 0 ?
    jle   .done
.loop:
    mov   edx, [rdi+rcx*4] ; x = a[i]  (scaled load)
    imul  edx, edx         ; x*x
    add   eax, edx         ; s += x*x
    add   ecx, 1           ; i++
    cmp   ecx, esi
    jl    .loop
.done:
    ret                    ; result in EAX
```

**AArch64 (AAPCS64)**
```armasm
sumsq:
    mov   w8, wzr          // s = 0
    mov   w9, wzr          // i = 0
    cmp   w1, #0
    b.le  .done
.loop:
    ldr   w10, [x0, w9, sxtw #2] // x = a[i]
    madd  w8, w10, w10, w8       // s += x*x (integer FMA)
    add   w9, w9, #1             // i++
    cmp   w9, w1
    b.lt  .loop
.done:
    mov   w0, w8           // return s
    ret
```

`xor eax,eax` is the idiomatic zero on x86 (smaller, breaks dependences); `wzr` is the AArch64 zero register. `madd` fuses multiply-add into one instruction — selection exploited an ISA feature x86 lacks for integers. Both loops are free of spills, so neither needs a frame: pure red-zone/frameless code.

---

## Emission — Text vs Object Code

The final step turns the scheduled `MachineInstr` stream into bytes. Two paths: write **assembly text** and shell out to an external assembler, or emit an **object file directly**. LLVM's **MC layer** does both from one representation.

**Assembly text (`.s`)**
- Human-readable; what `-S` prints
- Portable across assemblers; easy to inspect/debug
- Slower — re-parses text, spawns an assembler process

**Direct object emission**
- Compiler writes ELF/Mach-O/COFF (`.o`) itself — the *integrated assembler*
- No fork/exec, no re-parse — faster builds
- The default for `clang -c`

**The LLVM MC layer.** `MCInst` is a target-neutral instruction record. The same MC stream drives `MCAsmStreamer` → textual `.s`, or `MCObjectStreamer` → `.o` via an `MCCodeEmitter` (encodes opcodes to bytes) and an `MCAsmBackend` (relaxation, fixups). One encoder, two outputs — assembler and compiler share machinery. The emitter resolves each instruction to its binary form (opcode, ModRM/SIB on x86, fixed fields on AArch64) and lays out sections (`.text`, `.data`, `.rodata`).

---

## Relocations & the Linker Handoff

The compiler emits one object file at a time and cannot know the final address of anything outside it — an external function, a global, a string. It leaves a **relocation**: a note telling the linker "patch this field once you know the address". This is the bridge to Part 09.

**What a relocation is**
- A record: *(offset in section, symbol, type)*
- Type encodes the patch arithmetic: absolute64, PC-relative32, GOT/PLT-relative…
- Emitted whenever an operand references a symbol whose address isn't yet fixed

```x86asm
call  printf           ; → R_X86_64_PLT32 reloc
lea   rax, [rip + msg] ; → R_X86_64_PC32
```

**The assembler/linker split**
- *Assembler*: instructions → bytes + symbol table + relocations → `.o`
- *Linker*: combines objects, assigns final addresses, *applies* relocations, resolves symbols, builds the executable/shared object

**Teaser → Part 09.** Linking, dynamic loading, the GOT/PLT for position-independent code, and *just-in-time* compilation — where code is generated, relocated and executed at run time — are the subject of the next deck.

---

## Summary & Further Reading

### Key Takeaways
- The back end lowers optimised IR to target code via three interacting subproblems — a phase-ordering puzzle.
- **Selection**: cover the IR DAG with instruction tiles — maximal munch (greedy), BURS/iburg (optimal per tree), LLVM SelectionDAG & GlobalISel; peephole cleans up.
- **Allocation**: map vregs → k registers as graph k-colouring (Chaitin–Briggs: simplify/spill/select, optimistic colouring); coalescing (Briggs/George) removes moves; linear scan is the fast JIT choice; SSA → chordal graphs colour in P; PBQP for irregular ISAs.
- **Scheduling**: list-schedule the dependence DAG by critical path to hide latency — in tension with register pressure; software pipelining overlaps loop iterations.
- The **SysV AMD64 ABI** pins args to RDI/RSI/RDX/RCX/R8/R9, returns in RAX, requires 16-byte stack alignment at `call`, offers a 128-byte red zone — and constrains the allocator.
- Emission via the MC layer → `.s` or `.o`; relocations defer addresses to the linker.

### Further Reading
- Aho, Lam, Sethi & Ullman — the Dragon Book, ch. 8 (codegen) & 10 (scheduling)
- Cooper & Torczon — *Engineering a Compiler*, ch. 11–13
- Appel — *Modern Compiler Implementation*, ch. 9–11 (selection, allocation, scheduling)
- Chaitin et al. (1981); Briggs, Cooper & Torczon — *Improvements to Graph Colouring Register Allocation* (1994)
- George & Appel — *Iterated Register Coalescing* (1996)
- Poletto & Sarkar — *Linear Scan Register Allocation* (1999)
- Fraser, Hanson & Proebsting — *iburg* (BURS code generation)
- The System V AMD64 ABI document; the AArch64 AAPCS64; the LLVM Code Generator & MC docs

---

`Part 08 ✓` → `Part 09: Runtimes, JITs & Modern Backends`
