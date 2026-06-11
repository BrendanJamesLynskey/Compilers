# Compilers — Part 06: Intermediate Representations

**Part 06 — The Languages a Compiler Talks to Itself In**

> Between the AST and machine code sits the IR — the data structure the optimiser actually works on. We tour the spectrum from tree-like HIR to machine-near LIR, the linear and graph forms (three-address code, bytecode, the control-flow graph), and the idea that powers every modern compiler: Static Single Assignment, dominance and the φ-function.

`AST` -> `HIR` -> `MIR / TAC` -> `SSA` -> `LIR`

Levels - CFG - Dominance - SSA - LLVM

---

## Table of Contents

1. [Topics](#topics)
2. [Why an IR at All?](#why-an-ir-at-all)
3. [The N×M Argument, Again](#the-nm-argument-again)
4. [The Spectrum: HIR · MIR · LIR](#the-spectrum-hir--mir--lir)
5. [Linear IR: Three-Address Code](#linear-ir-three-address-code)
6. [Encoding TAC: Quads, Triples, Indirect Triples](#encoding-tac-quads-triples-indirect-triples)
7. [Stack-Based Bytecode](#stack-based-bytecode)
8. [Register vs Stack Bytecode](#register-vs-stack-bytecode)
9. [Graph IR: the Control-Flow Graph](#graph-ir-the-control-flow-graph)
10. [Finding Basic Blocks — Leaders](#finding-basic-blocks--leaders)
11. [Dominance & the Dominator Tree](#dominance--the-dominator-tree)
12. [The Dominance Frontier](#the-dominance-frontier)
13. [Static Single Assignment (SSA)](#static-single-assignment-ssa)
14. [SSA Before → After: the φ at a Join](#ssa-before--after-the-φ-at-a-join)
15. [Minimal-SSA Construction (Cytron et al.)](#minimal-ssa-construction-cytron-et-al)
16. [SSA Destruction (Out-of-SSA)](#ssa-destruction-out-of-ssa)
17. [LLVM IR — Structure](#llvm-ir--structure)
18. [LLVM IR — a Real .ll Snippet](#llvm-ir--a-real-ll-snippet)
19. [Other IRs: CPS & Sea-of-Nodes](#other-irs-cps--sea-of-nodes)
20. [MLIR & GCC's GIMPLE / RTL](#mlir--gccs-gimple--rtl)
21. [Two Shapes of the Same Expression](#two-shapes-of-the-same-expression)
22. [Design Tradeoffs](#design-tradeoffs)
23. [Summary & Further Reading](#summary--further-reading)

---

## Topics

An IR is the compiler's **working representation** — produced from the AST, optimised, then lowered to a target. This deck covers what makes a good IR and the major families in use today.

### Why & What
- Why an IR at all; desirable properties
- The N×M reuse argument, revisited
- The level spectrum: HIR / MIR / LIR

### Linear & Stack IRs
- Three-address code; quads / triples
- Stack bytecode: JVM, CPython, Wasm
- Register bytecode: Lua, Dalvik

### Graph IRs & SSA
- The control-flow graph & basic blocks
- Dominators, idom, dominance frontiers
- SSA, φ-functions, construction & destruction

### Real IRs & Tradeoffs
- LLVM IR in depth; the three forms
- CPS, sea-of-nodes, MLIR, GIMPLE/RTL
- Design tradeoffs: SSA, typing, levels

---

## Why an IR at All?

You *could* walk the AST and emit machine code directly — some toy compilers do. But the AST is shaped by the **source language**, and machine code by the **target**. An **intermediate representation** is a deliberately neutral middle form: a data structure designed for *analysis and transformation*, not for humans and not for a CPU.

**Four desirable properties:**

- **Easy to produce** from the AST (a straightforward lowering)
- **Easy to analyse & transform** — the optimiser's home turf
- **Easy to lower** to many targets
- **Language- & machine-independent** in the middle

These properties pull against each other. A high-level IR is easy to produce but hard to optimise for machines; a low-level IR is easy to lower but has lost source structure. Real compilers resolve this by using *several* IRs at different levels — lowering progressively.

Above all, an IR is the representation the *middle end* rewrites. Every optimisation in Part 07 — constant folding, CSE, dead-code elimination, loop-invariant code motion — is a transformation on the IR. Its design dictates which analyses are cheap and which are awkward.

---

## The N×M Argument, Again

The decisive reason for a *stable, shared* IR is reuse. With **N** source languages and **M** targets, route everything through one IR and you need **N + M** halves, not **N × M** whole compilers. The IR is the *narrow waist* — and crucially, the optimiser written once serves all N×M combinations.

*(Diagram: N front ends on the left converging into a single shared IR box, which fans out to M back ends on the right.)*

Beyond reuse, a shared IR is a **contract**: front-end and back-end teams can evolve independently as long as both honour the IR's semantics. This is why LLVM IR, JVM bytecode and Wasm are ecosystems, not single programs.

---

## The Spectrum: HIR · MIR · LIR

IRs vary in **level** — how close they sit to the source vs the machine. Modern compilers use *several*, lowering step by step: each level discards source abstraction and exposes more machine detail, enabling different optimisations.

- **High-level IR** — tree-like, close to the AST; desugared source. Where source-level checks (borrow-checking, type inference) live.
- **Mid-level IR** — CFG + three-address; explicit control flow. Where most machine-independent optimisation happens.
- **Low-level IR** — close to machine operations; addressing modes, near-ISA. Where register allocation and scheduling run.

| Compiler | High-level IR | Mid-level IR | Low-level IR |
|---|---|---|---|
| Rust (rustc) | HIR (desugared AST) | MIR (CFG, borrow-check) | LLVM IR → codegen |
| Swift | AST | SIL (Swift IL, SSA) | LLVM IR |
| V8 (JS) | AST / bytecode (Ignition) | Turbofan sea-of-nodes | machine code |

---

## Linear IR: Three-Address Code

**Three-address code** (TAC) is the canonical mid-level linear IR. Each instruction has at most one operator and (up to) three operands — a result and two sources: `t = a op b`. Complex expressions are broken into a sequence using compiler-generated **temporaries** (`t1`, `t2`…), which makes every sub-computation explicit and addressable.

```
source:   x = (a + b) * (a + b) - c;

three-address code:
   t1 = a + b
   t2 = a + b
   t3 = t1 * t2
   t4 = t3 - c
   x  = t4

after common-subexpression elimination:
   t1 = a + b
   t3 = t1 * t1
   x  = t3 - c
```

- **Why "three-address"?** At most three names per instruction. Control flow is explicit too — `goto L`, `if x < y goto L`, `param`/`call`. It maps cleanly onto a CFG.
- **Temporaries** each hold one intermediate value. They are *virtual* — unbounded in number; the register allocator (Part 08) later maps them onto finite physical registers.
- **Why linear?** Tree-walking obscures sharing and ordering. Flattening to a list of simple instructions makes data dependencies and evaluation order plain — exactly what analyses need.

---

## Encoding TAC: Quads, Triples, Indirect Triples

How is each TAC instruction *stored*? Three classic encodings trade memory against ease of reordering. For `t1 = b * 2; t2 = a + t1`:

**Quadruples** — `(op, arg1, arg2, result)`; the result is a named field.

```
  op   a1  a2  res
0 *    b   2   t1
1 +    a   t1  t2
```
Explicit result names → instructions move freely. Most common in practice.

**Triples** — `(op, arg1, arg2)`; no result field, the result *is* the row index, referenced as `(i)`.

```
  op   a1   a2
(0) *  b    2
(1) +  a    (0)
```
Compact — no temporaries stored. But moving a row renumbers every reference.

**Indirect triples** — a list of *pointers* into a triple table. Reorder by shuffling the pointer list; the triples stay put. Best of both: compact *and* reorderable — the classic optimising-compiler choice.

The lesson generalises: a result that is a stable *name* (quad) is easier to move than one that is a *position* (triple). SSA, later, takes naming to its logical extreme — every value gets a unique name forever.

---

## Stack-Based Bytecode

Many IRs target a **virtual machine** rather than analysis. Stack bytecode has no named operands at all: instructions push and pop an **operand stack**. `2 + 3` becomes *push 2, push 3, add*. It is trivially easy to generate from an AST (a post-order walk) and extremely compact.

JVM bytecode for `a + b * 2` (ints in locals 1, 2):

```
iload_1        ; push a
iload_2        ; push b
iconst_2       ; push 2
imul           ; pop 2,b  push b*2
iadd           ; pop b*2,a  push a+(b*2)
```

CPython bytecode (`dis` of `a + b * 2`):

```
LOAD_FAST   a
LOAD_FAST   b
LOAD_CONST  2
BINARY_OP   * (5)
BINARY_OP   + (0)
```

*(Diagram: the operand stack growing and shrinking after each op — push a, push b, push 2, imul collapses b·2, iadd collapses a+b·2.)*

**Where used:** JVM, CPython (`.pyc`), WebAssembly, the CLR. Compact, portable, easy to verify — ideal as a distribution format.

---

## Register vs Stack Bytecode

Stack VMs are easy to generate but execute many tiny instructions. **Register bytecode** uses a flat array of virtual registers instead of a stack, so one instruction can name its operands directly — fewer instructions, fewer dispatches, faster interpretation. Lua 5 and Android's Dalvik pioneered this for production VMs.

Stack (Wasm-style) for `x = a + b * 2`:

```
local.get a
local.get b
i32.const 2
i32.mul
i32.add
local.set x      ; 6 instructions
```

Register (Lua-style) — `R` are registers:

```
MUL  R3  Rb  K2   ; R3 = b * 2
ADD  Rx  Ra  R3   ; Rx = a + R3
                  ; 2 instructions
```

| Axis | Stack VM | Register VM |
|---|---|---|
| Operands | implicit (the stack) | explicit (register #) |
| Instr. count | more (push/pop) | fewer |
| Instr. size | small | larger (operand fields) |
| Codegen | trivial (post-order) | needs register assignment |
| Dispatch cost | higher (more ops) | lower |
| Examples | JVM, CPython, Wasm, CLR | Lua 5, Dalvik, BEAM-ish |

**The trade:** stack code is smaller and simpler to *produce*; register code is faster to *interpret* (fewer dispatches dominate). Distribution formats (Wasm, JVM) favour stack; embedded interpreters (Lua) favour registers.

---

## Graph IR: the Control-Flow Graph

Optimisation needs structure linear code hides. The **control-flow graph** (CFG) is a directed graph whose nodes are **basic blocks** — maximal straight-line instruction sequences — and whose edges are possible transfers of control. Control enters a block only at its top and leaves only at its bottom.

*(Diagram: a loop CFG — entry B0 branches to body B1 and exit B3; B1 → latch B2; B2 has a back edge to B1.)*

- **Basic block** — a maximal run of instructions with one entry (the first) and one exit (the last). No branch lands in the middle; no branch leaves before the end.
- **Edges** — an edge B→C means control *can* pass from B to C. A conditional branch gives two successors; a fall-through or jump gives one.
- **Back edge & loops** — an edge to a block that dominates its source is a *back edge* — the signature of a loop. Loops are found from the CFG, not the source syntax.

---

## Finding Basic Blocks — Leaders

Given a linear instruction list, the CFG is built by identifying **leaders** — the first instruction of each block. A block runs from a leader up to (but not including) the next leader.

**A leader is…**
- The *first* instruction of the function
- Any *target* of a jump or branch (a label)
- Any instruction *immediately following* a jump or branch (conditional or unconditional)

Each leader begins a basic block containing it and all following instructions up to the next leader or the end. Add a CFG edge for each fall-through and each branch target.

```
    i = 0          ; L1: leader (first)
L2: t = i < n      ; leader (branch target)
    if !t goto L4  ;
    s = s + a[i]   ; L3: leader (after branch)
    i = i + 1
    goto L2        ;
L4: return s       ; leader (branch target)

leaders: {1, 2, 4, 7}
blocks:
  B0 = [i=0]                       → B1
  B1 = [t=i<n, if !t goto L4]      → B2, B3
  B2 = [s=s+a[i], i=i+1, goto L2]  → B1
  B3 = [return s]
```

This three-rule algorithm is linear in the instruction count and is the foundation under every later analysis — liveness, reaching definitions, dominance and SSA all operate on the CFG it produces.

---

## Dominance & the Dominator Tree

A node **d dominates** node **n** (written `d dom n`) if *every* path from the entry to n passes through d. Dominance is reflexive (n dom n) and transitive. It is the backbone of loop detection and SSA construction.

*(Diagram: a diamond CFG A → {B, C} → D → E, and beside it the dominator tree where A is the root and B, C, D are all direct children of A — because the join D is reached via both B and C, so neither dominates it.)*

- **Immediate dominator `idom(n)`** — the *unique closest* strict dominator of n, the last dominator before n on every path. Every node except entry has exactly one idom, so the idom relation forms a **tree** rooted at entry.
- **Why D's idom is A** — D is reached via A→B→D *and* A→C→D. Neither B nor C is on *every* path, so neither dominates D. Only A does — so `idom(D)=A`.
- **Computing it** — iterative dataflow (Cooper–Harvey–Kennedy, 2001) is simple and fast in practice; Lengauer–Tarjan (1979) is near-linear. Both yield the dominator tree.

---

## The Dominance Frontier

The **dominance frontier** of node n, `DF(n)`, is the set of nodes that n *almost* dominates: nodes where n's dominance "runs out". Formally, `DF(n)` is the set of nodes y such that n dominates a *predecessor* of y but does *not* strictly dominate y itself. It is precisely where merging happens — and exactly where SSA must insert φ-functions.

- **Intuition** — start at n and walk forward. As long as you stay in n's dominated region, no merge is needed. The frontier is the *first* set of join points reachable from n where control could also have arrived *without* going through n — so a value defined in n may or may not be live.
- **Why it matters** — if a variable is assigned in block n, then any use that could see *either* n's value or another definition needs a φ to reconcile them. Those merge points are exactly `DF(n)`. This is the key insight of Cytron et al. (1991) for placing φ-functions optimally.

In the diamond above, block D is a join of the B and C paths. `DF(B) = {D}` and `DF(C) = {D}`: B dominates itself (a predecessor of D) but does not dominate D (you can reach D via C). So a variable assigned in B *and* in C forces a φ at D.

---

## Static Single Assignment (SSA)

**SSA form** imposes one rule: every variable is assigned *exactly once*, statically. Reassignments become fresh, subscripted names — `x` → `x₁`, `x₂`, …. The payoff is that every name has a *single, unambiguous definition*, so def–use chains are explicit and free: to find a value's definition you just read its name.

```
original:            SSA:
  x = 1                x₁ = 1
  x = x + 1            x₂ = x₁ + 1
  y = x * 2            y₁ = x₂ * 2
```

**Why it powers optimisation:**
- Each use names exactly one def — no aliasing of names
- Constant propagation, GVN, dead-code elimination become near-trivial
- Sparse analyses: walk def→use directly, skip irrelevant blocks

**The join problem.** What name does a variable have *after* two branches that each assigned it? Straight-line renaming can't answer — control could arrive from either side. SSA's answer is the **φ-function**.

**φ-function.** `x₃ = φ(x₁, x₂)` at a join block *selects* the value corresponding to the predecessor through which control arrived. It is a notional, parallel, edge-indexed copy — not a real instruction the CPU runs.

**History.** Introduced by Cytron, Ferrante, Rosen, Wegman & Zadeck at IBM (1991). Now the standard mid-level form: LLVM, GCC, V8, HotSpot, Swift SIL.

---

## SSA Before → After: the φ at a Join

A diamond: `x` is assigned on both arms of an `if`, then used after the merge. Renaming alone produces two names; the join block needs a **φ** to pick the right one per incoming edge.

```
Before SSA:                After SSA:

  entry: if c                entry: if c
   /        \                 /         \
x = 1      x = 2           x₁ = 1      x₂ = 2
   \        /                 \         /
    merge:                     merge:
    y = x + 1                  x₃ = φ(x₁, x₂)
                               y₁ = x₃ + 1
```

The φ reads `x₁` if control came via the left edge, `x₂` via the right — a value parameterised by the incoming CFG edge. The merge block here is the *dominance frontier* of both arms.

---

## Minimal-SSA Construction (Cytron et al.)

The classic algorithm builds **minimal SSA** — the fewest φ-functions that satisfy the single-assignment property — in two phases, driven by the dominator tree and dominance frontiers.

**Phase 1 — Place φ-functions:**
- Compute the dominator tree and `DF(n)` for every block
- For each variable v, find the blocks that *define* v
- Insert a φ for v at every block in the **iterated dominance frontier** of those defining blocks
- Iterate: a placed φ is itself a definition, so it can trigger more φs

**Phase 2 — Rename:**
- Walk the dominator tree top-down
- Keep a version stack per variable; each definition pushes a fresh subscript
- Each use takes the current top-of-stack version
- Fill in φ operands from the version live on each predecessor edge

**Why dominance frontiers?** A φ for v is needed exactly where two definitions of v converge — the join points just outside a definition's dominated region. That set *is* the dominance frontier. Iterating it accounts for φs creating new definitions.

**"Minimal" vs "pruned":** minimal SSA may place φs for variables that are dead at the join. *Pruned SSA* adds a liveness check so a φ is inserted only if v is live there — fewer φs, more analysis. Braun et al. (2013) construct SSA *directly* during IR generation, without a separate dominance-frontier pass.

---

## SSA Destruction (Out-of-SSA)

Hardware has no φ. Before code generation the compiler must **leave SSA**, replacing each φ with ordinary copies on the incoming edges. Done naively this is buggy — two classic hazards lurk.

**The lost-copy problem.** Naively inserting a copy `x = x₁` at the end of a predecessor can be clobbered if that block has other successors or the value of `x₁` is overwritten before the copy along a critical edge. The copy's value is "lost". *Fix:* split **critical edges** — an edge from a block with multiple successors to one with multiple predecessors — by inserting a new empty block to host the copy safely.

**The swap problem.** Parallel φs like `a₂=φ(a₁,b₁)`, `b₂=φ(b₁,a₁)` swap two values. Sequentialised carelessly — `a=b; b=a` — the first copy destroys the value the second needs. *Fix:* all φs in a block execute *simultaneously*; lower them as a **parallel copy**, then sequentialise with a temporary to break cycles (`t=a; a=b; b=t`). Sreedhar et al. (1999) and Boissinot et al. (2009) give correct, efficient methods.

Both hazards share a root cause: a φ is a *parallel, edge-placed* operation, and turning it into sequential, block-placed copies must preserve those two properties. Get either wrong and the program miscompiles silently.

---

## LLVM IR — Structure

LLVM IR (Lattner, 2003) is the most widely used production IR. It is **typed**, in **SSA form**, and organised as a strict containment hierarchy. Virtual registers (`%name`) are SSA values — assigned once, infinite in number.

**Hierarchy:** Module → Function → Basic Block → Instruction. Every block ends in exactly one *terminator* (`br`, `ret`, `switch`…).

**Typed & SSA:** every value carries a type (`i32`, `i8*`, `double`, `{i32,i8}`). `%x` are virtual registers (SSA); `@g` are globals. A verifier enforces both.

**Key instructions:**
- `getelementptr` — type-safe address arithmetic, never dereferences
- `phi` — the SSA φ-function
- `br` — conditional / unconditional branch
- `call` + intrinsics (`@llvm.memcpy`…)

---

## LLVM IR — a Real .ll Snippet

Iterative factorial, lowered to LLVM IR. Note the SSA virtual registers, the explicit basic blocks, the loop `phi` nodes selecting per predecessor, and the typed conditional branch.

```llvm
define i32 @fact(i32 %n) {
entry:
  br label %loop

loop:
  %i   = phi i32 [ 1, %entry ], [ %i.next, %loop ]
  %acc = phi i32 [ 1, %entry ], [ %acc.next, %loop ]
  %acc.next = mul i32 %acc, %i
  %i.next   = add i32 %i, 1
  %cond = icmp sle i32 %i.next, %n
  br i1 %cond, label %loop, label %done

done:
  ret i32 %acc.next
}
```

**Read it:**
- `%i` / `%acc` are loop-carried — each `phi` picks the entry value first, the updated value on later iterations
- `icmp sle` → an `i1` (signed ≤)
- `br i1 %cond, A, B` — typed two-way terminator

**Three isomorphic forms:**
- **.ll** — human-readable text (shown here)
- **.bc** — compact binary bitcode
- **in-memory** — the C++ object graph the optimiser mutates

All three encode the *same* IR; `llvm-as` / `llvm-dis` convert losslessly between text and bitcode.

---

## Other IRs: CPS & Sea-of-Nodes

**Continuation-Passing Style (CPS).** A functional IR where every call passes an explicit *continuation* — "what to do next" as a function. Control flow becomes function application; there is no implicit return.

```
; direct:   z = f(x) + 1
; CPS:
f(x, k) where
  k(r) = { z = r + 1; ... }
```

**SSA ≡ CPS:** Appel and Kelsey showed they are essentially the same idea — a φ corresponds to a continuation's parameter, dominance to lexical scope. Used by SML/NJ, MLton, and the Scheme/ML world.

**Sea-of-Nodes.** Cliff Click's IR (1995) where *control and data are one graph*. Nodes are operations; edges are data and control dependencies. Instructions are not pinned to blocks — only ordering constraints exist, so the scheduler places them late, freely floating to better positions.
- Implicit, maximal instruction freedom → strong global code motion
- SSA is "baked in" — data edges *are* def–use
- Used by Java HotSpot C2 and V8 TurboFan

Cost: harder to read, debug and serialise than a block-structured IR — V8 has since moved parts toward a more conventional CFG IR (Maglev/Turboshaft).

---

## MLIR & GCC's GIMPLE / RTL

**MLIR (2019, Lattner et al.)** — "Multi-Level IR" — not one IR but a *framework* for defining IRs. Its core abstraction is the **dialect**: a namespaced set of operations and types. One module can mix dialects (`linalg`, `affine`, `gpu`, `llvm`) and lower progressively between them.
- Built for ML/heterogeneous compilers (TensorFlow, IREE)
- Solves IR *proliferation*: shared infra for many domain IRs
- Region/op/block structure; SSA values throughout

Why it exists: every domain (tensors, hardware, polyhedral) was reinventing IR plumbing. MLIR makes the levels first-class and composable.

**GCC — GIMPLE.** GCC's mid-level IR: a *three-address*, tree-based form in SSA. The front ends lower to GENERIC, then to GIMPLE, on which most machine-independent optimisation runs.

**GCC — RTL.** *Register Transfer Language* — GCC's low-level IR, close to the machine: virtual then physical registers, addressing modes, machine patterns. Instruction selection, scheduling and register allocation happen here.

GENERIC → GIMPLE (SSA, target-neutral) → RTL (target-near) is the same progressive-lowering pattern as rustc's HIR/MIR/LLVM and Swift's AST/SIL/LLVM.

---

## Two Shapes of the Same Expression

A final side-by-side for `x = a + b * 2`: the same computation as **stack bytecode** (implicit operands, more instructions) and as **three-address code** (explicit temporaries, named results).

```
Stack bytecode               Three-address code
  push a    ; [a]              t1 = b * 2
  push b    ; [a, b]           t2 = a + t1
  push 2    ; [a, b, 2]        x  = t2
  mul       ; [a, b*2]
  add       ; [a+b*2]
  store x   ; []
  (6 ops · implicit)          (3 ops · named → easy CSE/SSA)
```

Stack form is built for *generation & transport*; three-address form is built for *analysis & optimisation*. SSA refines the latter further by making every name unique.

---

## Design Tradeoffs

There is no universally best IR. Three axes dominate the design space, and real compilers pick a different point on each — often using several IRs to get the best of each.

**SSA vs non-SSA:**
- SSA: explicit def–use, sparse fast analyses, simpler optimisers
- Cost: φ placement & the in/out-of-SSA dance
- Net win for most optimisers — near-universal today

**Typed vs untyped:**
- Typed (LLVM): catches lowering bugs, enables type-based alias analysis & verification
- Untyped (some bytecode): simpler, but pushes checks elsewhere
- LLVM is moving to *opaque pointers* — less type, less friction

**How many levels:**
- One IR: simple, but compromises high- vs low-level needs
- Several: HIR/MIR/LIR — each level fits its job; more lowering code
- MLIR turns "many levels" into a managed framework

**Rules of thumb.** Match the IR to the work: tree/HIR for source-level checks; SSA + CFG (MIR) for machine-independent optimisation; low-level IR for register allocation and scheduling. Make each transformation's job *local* and the right facts *cheap to read* — that is the whole art of IR design.

---

## Summary & Further Reading

### Key Takeaways

- An IR is a neutral middle form: easy to produce, analyse, lower — the N+M narrow waist
- IRs span a level spectrum: HIR (tree) → MIR (CFG + 3-address) → LIR (machine-near)
- Three-address code: `t = a op b`; quads / triples / indirect triples encode it
- Stack bytecode (JVM, CPython, Wasm) is compact & easy to emit; register bytecode (Lua, Dalvik) is faster to interpret
- The CFG = basic blocks (found via leaders) + edges; back edges mark loops
- *d dom n* = every entry→n path goes through d; idom → dominator tree; DF(n) = where φs go
- SSA: one assignment per name; φ at dominance frontiers (Cytron et al.); out-of-SSA must handle the lost-copy & swap problems
- LLVM IR: typed SSA, Module→Func→BB→Instr, three isomorphic forms (.ll / .bc / in-memory)
- CPS ≡ SSA; sea-of-nodes (HotSpot C2, TurboFan) fuses control & data; MLIR generalises "many levels"; GCC uses GIMPLE then RTL

### Further Reading

- Cytron, Ferrante, Rosen, Wegman & Zadeck — *Efficiently Computing SSA & the Control Dependence Graph* (TOPLAS, 1991)
- Cooper & Torczon — *Engineering a Compiler*, ch. 5 & 9
- Aho, Lam, Sethi & Ullman — *Compilers: Principles, Techniques & Tools* (the Dragon Book), ch. 6 & 8–9
- Cliff Click — *A Simple Graph-Based Intermediate Representation* (sea-of-nodes)
- Appel — *SSA is Functional Programming* (CPS ≡ SSA)
- Lattner & Adve — the LLVM paper (CGO 2004); the LLVM Language Reference
- Braun et al. — *Simple and Efficient Construction of SSA Form* (2013); the MLIR documentation

### Mental Model to Keep

The AST becomes a series of ever-lower representations — tree, then linear three-address, then a CFG in SSA, then machine-near code — each form making explicit exactly what the next phase needs and the previous form hid.

`Part 06 done` -> `Part 07: Optimization`
