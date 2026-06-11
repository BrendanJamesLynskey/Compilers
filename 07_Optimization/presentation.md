# Compilers — Part 07: Optimization

**Part 07 — Making Code Faster, Smaller, Leaner — Without Changing What It Means**

> The optimiser is where the compiler earns its keep. We build the data-flow analysis framework from the ground up — lattices, transfer functions, the iterative worklist — survey the four classic analyses and the transformations they enable, dig into loop and SSA-based optimisations, and end with the phase-ordering problem and the real GCC/LLVM pass pipelines. The one inviolable rule throughout: optimisations must preserve program semantics.

`Analyse` -> `Transform` -> `Verify` -> `Repeat`

Scope - Data-Flow - Transforms - Loops - SSA - Pipeline

---

## Table of Contents

1. [Topics](#topics)
2. [Goals & the Inviolable Rule](#goals--the-inviolable-rule)
3. [Scope: How Wide Does the Optimiser Look?](#scope-how-wide-does-the-optimiser-look)
4. [The Data-Flow Analysis Framework](#the-data-flow-analysis-framework)
5. [The Lattice — ⊤, ⊥, Meet & Join](#the-lattice--top-bottom-meet--join)
6. [The Iterative Worklist Algorithm](#the-iterative-worklist-algorithm)
7. [Data-Flow on the CFG — Worked Example](#data-flow-on-the-cfg--worked-example)
8. [Two Axes: Direction × Confidence](#two-axes-direction--confidence)
9. [Reaching Definitions & Live Variables](#reaching-definitions--live-variables)
10. [Available & Very-Busy Expressions](#available--very-busy-expressions)
11. [The Four Classic Analyses — at a Glance](#the-four-classic-analyses--at-a-glance)
12. [Constant Folding, Propagation & Copy Propagation](#constant-folding-propagation--copy-propagation)
13. [CSE, Dead-Code Elimination & Algebraic Simplification](#cse-dead-code-elimination--algebraic-simplification)
14. [Strength Reduction & Function Inlining](#strength-reduction--function-inlining)
15. [Loop Optimisations — LICM](#loop-optimisations--licm)
16. [Induction Variables, Unrolling, Fusion & Interchange](#induction-variables-unrolling-fusion--interchange)
17. [Auto-Vectorisation & SIMD](#auto-vectorisation--simd)
18. [Why SSA Makes Optimisation Cheaper](#why-ssa-makes-optimisation-cheaper)
19. [It All Compounds — A Before / After](#it-all-compounds--a-before--after)
20. [Alias / Points-To Analysis — the Great Limiter](#alias--points-to-analysis--the-great-limiter)
21. [Supporting Structures: Call Graph & Dominators](#supporting-structures-call-graph--dominators)
22. [The Phase-Ordering Problem](#the-phase-ordering-problem)
23. [The Real Pipeline: Pass Managers & -O Levels](#the-real-pipeline-pass-managers---o-levels)
24. [Undefined Behaviour as an Optimisation Licence](#undefined-behaviour-as-an-optimisation-licence)
25. [Summary & Further Reading](#summary--further-reading)

---

## Topics

Optimisation is a misnomer — it is really *code improvement*. This deck builds the theory (data-flow analysis) and then applies it: the classic transformations, loop and SSA-based passes, and how real compilers sequence them.

### Foundations
- Goals & the as-if (semantics-preserving) rule
- Scope: peephole → local → global → IPO → LTO → PGO
- The data-flow framework: lattices, transfer functions
- The iterative worklist algorithm & why it terminates

### The Classic Analyses
- Reaching definitions, live variables
- Available & very-busy expressions
- Direction × meet: the four-corner table

### Transformations
- Constant folding/propagation, copy prop, CSE, DCE
- Strength reduction, inlining, algebraic simplification
- Loop: LICM, IV reduction, unrolling, fusion, vectorisation
- SSA-based: SCCP, GVN, aggressive DCE

### In Practice
- Alias analysis — why it limits everything
- Phase ordering; the LLVM/GCC pass pipeline
- `-O0..-O3 / -Os / -Oz`; pass managers
- Undefined behaviour as an optimisation licence

---

## Goals & the Inviolable Rule

An optimiser rewrites code to improve some **objective** — but it may only do so if the rewrite is **semantics-preserving**. This is the **"as-if" rule**: the program must behave *as if* every step of the abstract machine ran, even if the compiler actually did something cheaper.

- **Run faster** — fewer / cheaper instructions, fewer memory accesses, better cache & branch behaviour. The classic target.
- **Smaller code** — shrink the binary (`-Os`, `-Oz`); matters for embedded, instruction cache, mobile download size.
- **Less energy** — fewer cycles and memory ops mean less power; critical on battery & in the datacentre.

**Never "optimal".** Finding genuinely optimal code is undecidable (reduces to the halting problem) or NP-hard in the decidable cases. "Optimisation" is a historical misnomer — we make code *better*, heuristically, never provably best.

**Observable behaviour only.** The as-if rule constrains only *observable* effects: I/O, volatile accesses, program termination, and (in C++) the order of certain side-effects. Intermediate values, evaluation order and dead stores are fair game to rewrite or delete.

A "faster" program that computes a different answer is not an optimisation — it is a **miscompilation**. Correctness first, always.

---

## Scope: How Wide Does the Optimiser Look?

Optimisations are classified by the **scope** they reason over. Wider scope sees more opportunities but costs more analysis — and crosses harder boundaries.

| Scope | Reasons over | Examples | Cost |
|---|---|---|---|
| Peephole | A small sliding window of adjacent instructions | `mul x,2` → `shl x,1`; remove redundant `mov` | Tiny |
| Local | One basic block (straight-line code) | Local CSE, local constant folding, value numbering | Low |
| Global (intraprocedural) | One whole function — the entire CFG | Data-flow analyses, LICM, global DCE, GVN | Moderate |
| Interprocedural (IPA/IPO) | Across function boundaries & the call graph | Inlining, constant arg propagation, devirtualisation | High |
| Link-Time (LTO) | Across translation units, at link time | Cross-TU inlining & DCE; ThinLTO scales it | High |
| Whole-Program | The entire program at once (closed world) | Aggressive devirtualisation, global layout | Very high |
| Profile-Guided (PGO) | Any scope, informed by runtime profiles | Hot/cold splitting, better inlining & layout, branch hints | 2-pass build |

**LTO & ThinLTO.** Classic LTO serialises IR into the object files and re-optimises the merged whole at link time. ThinLTO (LLVM) keeps it parallel & incremental by exchanging only summaries plus on-demand imports.

**PGO — measure, don't guess.** Build instrumented → run a representative workload → feed the `.profdata` back. The compiler now *knows* which branches are hot, sizing inlining and block layout to reality rather than static heuristics.

---

## The Data-Flow Analysis Framework

Almost every global optimisation rests on **data-flow analysis** (DFA): a general method for computing, at every program point, a conservative summary of what *must* or *may* hold there. The framework has four parts.

The four ingredients:

- **A lattice** of facts (the domain), with a partial order & a *meet* operator
- **A direction** — forward or backward through the CFG
- **Transfer functions** per block: how a block changes the fact (gen / kill)
- **A boundary condition** at entry (forward) or exit (backward)

The general equation:

```
in[B]  = MEET over predecessors P of out[P]   (forward)
out[B] = transfer_B( in[B] )
       = gen[B] ∪ ( in[B] − kill[B] )

# backward analyses swap in/out and use successors:
out[B] = MEET over successors S of in[S]
in[B]  = transfer_B( out[B] )
```

The **meet** operator combines facts arriving along different paths. Its choice (∪ vs ∩) is exactly what makes an analysis **may** or **must**.

---

## The Lattice — ⊤, ⊥, Meet & Join

The domain of facts forms a **lattice**: a partially-ordered set where every pair of elements has a greatest lower bound (**meet**, ∧) and a least upper bound (**join**, ∨). **⊤** (top) is "no information yet / everything possible"; **⊥** (bottom) is "over-defined / conflicting". Iteration moves facts monotonically *down*.

*(Diagram: the constant-propagation lattice — ⊤ on top meaning "unknown/undefined", a middle row of known constants `x=0, x=1, x=2, …`, and ⊥ at the bottom meaning "not a constant / varies". The meet of two different constants is ⊥.)*

**Why a lattice?**
- The meet of facts from two paths is well-defined & associative
- The partial order gives a notion of "more / less precise"
- **Finite height** + monotone transfers ⇒ iteration *must* terminate

**Meet vs join.** For **may** analyses the meet is set **union** ("could happen on *some* path"). For **must** analyses it is set **intersection** ("holds on *every* path"). The lattice is drawn so meet always moves toward less information.

---

## The Iterative Worklist Algorithm

To solve the equations we iterate to a **fixed point**. Initialise every block to ⊤, then repeatedly recompute blocks whose inputs changed until nothing changes. The **worklist** keeps us from rescanning stable blocks.

```
worklist ← all blocks
out[B]   ← ⊤   for all B   (∅ for a union analysis)
out[entry] ← boundary condition

while worklist not empty:
    B ← pop(worklist)
    in[B]  ← MEET over preds P of out[P]
    new    ← gen[B] ∪ (in[B] − kill[B])
    if new ≠ out[B]:
        out[B] ← new
        worklist ← worklist ∪ successors(B)
```

Backward analyses run on the reverse CFG: swap preds/succs and in/out, seed from the exit.

**Why it terminates:**
- **Monotonicity** — transfer functions never *add* precision; facts only move down the lattice
- **Finite height** — any chain ⊤ > … > ⊥ has bounded length
- So each block's value can change only finitely often ⇒ the loop halts

**MFP vs MOP.** The algorithm computes the **Maximal Fixed Point** (MFP). The ideal answer is the **Meet-Over-all-Paths** (MOP) solution. For *distributive* frameworks MFP = MOP; otherwise MFP is a safe, conservative under-approximation (MFP ≤ MOP). Enumerating all paths is undecidable — hence we solve equations instead.

**Order matters for speed.** Visiting in **reverse postorder** (forward) or postorder (backward) typically converges in a handful of passes — bounded by loop-nesting depth + 2.

---

## Data-Flow on the CFG — Worked Example

**Reaching definitions** (forward, may): which assignments could reach a point unkilled? At a join block, `in` is the *union* of both predecessors' `out`.

*(Diagram: a diamond CFG. B1 `d1: x = 1` branches to B2 `d2: x = 2` (then) and B3 `d3: x = 3` (else), which both flow into the join B4 `use x`. At B4, `in = {d2, d3}` — `d1` is killed on both arms.)*

Gen/kill for B2:

```
gen[B2]  = { d2 }
kill[B2] = { d1, d3 }   # other defs of x
out[B2]  = gen ∪ (in − kill)
         = { d2 }
```

At the join B4: `in[B4] = out[B2] ∪ out[B3] = {d2, d3}`. The original `d1` was killed on both arms, so it does *not* reach the use. This drives constant propagation: since two different values reach, `x` is not a known constant at B4.

---

## Two Axes: Direction × Confidence

Every data-flow analysis sits in one of four boxes, fixed by two independent choices: which **direction** information flows, and whether facts are **may** (union) or **must** (intersection).

|  | may (∪) | must (∩) |
|---|---|---|
| **forward** | Reaching Definitions | Available Expressions |
| **backward** | Live Variables | Very-Busy Expressions |

**Direction.** *Forward*: a block's fact depends on its *predecessors* (what has happened so far). *Backward*: it depends on its *successors* (what will be needed later) — solved on the reverse CFG.

**May (union).** "True on *some* path." Safe for optimistic deletion guards — a variable is *live* if used on any later path, so we must keep it.

**Must (intersection).** "True on *every* path." Needed to justify a rewrite — an expression is *available* only if already computed on all incoming paths, so we may reuse it.

---

## Reaching Definitions & Live Variables

**Reaching Definitions — forward / may.** A definition `d` *reaches* a point if there is a path from `d` to it along which `d` is not overwritten. Powers use-def chains, constant propagation, loop-invariance tests.

```
gen[B]  = defs in B not later redefined in B
kill[B] = all other defs of those vars
in[B]   = ∪ out[pred]          # MAY = union
out[B]  = gen[B] ∪ (in[B] − kill[B])
init: out = ∅
```

**Live Variables — backward / may.** A variable is *live* at a point if its current value *may* be used along some path before being redefined. The backbone of dead-code elimination and register allocation (interference).

```
use[B] = vars read in B before any def in B
def[B] = vars assigned in B
out[B] = ∪ in[succ]            # MAY = union
in[B]  = use[B] ∪ (out[B] − def[B])
init: in = ∅ ;  solved on reverse CFG
```

**The symmetry.** Reaching defs flows *forward* over `gen/kill`; liveness flows *backward* over `use/def`. Both are **may** analyses (union meet) — they answer "could this matter on *some* path?", the safe question when deciding what to *keep*.

---

## Available & Very-Busy Expressions

**Available Expressions — forward / must.** An expression `a+b` is *available* at a point if computed on *every* path to it with neither operand changed since. Justifies global CSE: reuse the earlier value.

```
gen[B]  = exprs computed in B, operands not later killed in B
kill[B] = exprs whose operands B redefines
in[B]   = ∩ out[pred]          # MUST = intersection
out[B]  = gen[B] ∪ (in[B] − kill[B])
init: out = U (universal set); in[entry]=∅
```

**Very-Busy (Anticipated) Expressions — backward / must.** An expression is *very busy* at a point if it *will* be evaluated on *every* path from there before any operand changes. Enables code hoisting (compute once, earlier) to shrink code size.

```
use[B] = exprs evaluated in B before operand def
def[B] = exprs killed by defs in B
out[B] = ∩ in[succ]            # MUST = intersection
in[B]  = use[B] ∪ (out[B] − def[B])
init: in = U ; solved on reverse CFG
```

**The must initialiser gotcha.** For **must** (intersection) analyses, interior blocks must be initialised to the *universal* set `U` (full optimism), not ∅ — otherwise the first intersection collapses everything to empty and the fixed point is wrong. May/union analyses initialise to ∅.

---

## The Four Classic Analyses — at a Glance

Memorise the four-corner pattern and you can derive any of them from first principles.

| Analysis | Domain (facts) | Direction | Meet | Confidence | Init (interior) | Enables |
|---|---|---|---|---|---|---|
| Reaching Definitions | Set of definitions | Forward | ∪ union | may | ∅ | use-def chains, const prop |
| Live Variables | Set of variables | Backward | ∪ union | may | ∅ | DCE, register allocation |
| Available Expressions | Set of expressions | Forward | ∩ intersect | must | U (universe) | global CSE |
| Very-Busy Expressions | Set of expressions | Backward | ∩ intersect | must | U (universe) | code hoisting (size) |

**The pattern:**
- *may* ↔ *union* ↔ init ∅ — "keep if it might matter"
- *must* ↔ *intersection* ↔ init U — "rewrite only if guaranteed"
- Forward seeds the entry; backward seeds the exit

**Beyond the four.** The same framework yields copy propagation, constant propagation (a non-distributive lattice), dominators (forward/must), nullness, taint, def-use, and points-to. One engine, endless analyses.

---

## Constant Folding, Propagation & Copy Propagation

The bread-and-butter scalar transforms. Each is cheap, enabling, and synergistic.

**Constant folding** — evaluate constant expressions at compile time:

```c
int s = 60 * 60 * 24;   // folds to:
int s = 86400;
```

Mind overflow & FP rounding — folding must match the target's runtime semantics exactly.

**Constant propagation** — substitute a variable's known constant value at its uses:

```c
x = 4;
y = x + 2;   // → y = 4 + 2 → y = 6 (then folds)
```

Driven by reaching-definitions / SSA. A single value must reach the use.

**Copy propagation** — after `x = y`, replace later uses of `x` with `y`:

```c
x = y;
z = x + 1;   // → z = y + 1 ; x now possibly dead → DCE
```

**They feed each other.** Propagation exposes constants → folding collapses them → some variables become dead → DCE removes them → new copies/expressions become trivial. This cascade is exactly why a pass pipeline revisits the same simple passes many times.

---

## CSE, Dead-Code Elimination & Algebraic Simplification

**Common-subexpression elimination** — if `a+b` is *available*, compute once and reuse:

```c
t1 = a + b;
x = a + b;   // → x = t1
y = a + b;   // → y = t1
```

Local CSE via value numbering; global CSE via available-expressions or GVN.

**Dead-code elimination** — remove computations whose results are never used (variable not *live*), and unreachable blocks:

```c
x = expensive();  // x never used → deleted entirely
if (0) { ... }    // unreachable → removed
```

Must respect side-effects: a call that does I/O is not dead even if its result is.

**Algebraic simplification** — apply identities & canonicalise:

```c
x + 0 → x     x * 1 → x      x * 0 → 0
x - x → 0     x & x → x      !!b → b
(a+b)+c → a+(b+c)  reassoc
```

FP is tricky: `x*0.0 ≠ 0.0` for NaN/−0.0 unless `-ffast-math`.

**Sound deletion needs liveness + purity.** DCE is only legal when (a) the value is provably dead and (b) the computation is side-effect-free. This is why alias and effect analyses gate so much: if the compiler cannot prove a store or call is pure and unobserved, it cannot remove it.

---

## Strength Reduction & Function Inlining

**Strength reduction** — replace an expensive operation with a cheaper equivalent:

```c
y = x * 8;     → y = x << 3;
y = x % 16;    → y = x & 15;   // unsigned
y = x / 2;     → y = x >> 1;   // unsigned
```

The headline use is in loops: turn a multiply by the loop index into an add per iteration (induction-variable strength reduction).

**Function inlining** — replace a call with the callee's body. The single most *enabling* optimisation — it exposes the callee's internals to the caller's optimiser (const prop, CSE, DCE across the boundary):

```c
int sq(int n){ return n*n; }
y = sq(a+1);
// inlines to:
y = (a+1)*(a+1);   // now foldable/CSE-able
```

**Cost / benefit — code bloat.** Inlining duplicates the body at each call site. Over-inlining bloats the binary, blows the instruction cache, and can *slow* code. Heuristics weigh callee size, call-site hotness (PGO), and recursion. `__attribute__((always_inline))` forces it; `noinline` forbids it.

---

## Loop Optimisations — LICM

Loops are where programs spend their time, so they get the most attention. **Loop-Invariant Code Motion** hoists computations that produce the same value every iteration out to the **pre-header** — computed once instead of N times.

*(Diagram: before — a loop body containing `t = a * b` (invariant) and `x[i] = t + i`, with `a*b` recomputed n times. After — `t = a * b` lives in a pre-header block before the loop; the loop body only does `x[i] = t + i`.)*

**When is it legal?**
- The expression's operands are invariant (defined outside, or themselves invariant)
- The instruction is safe to speculate, or its block dominates all loop exits (so it would always have run)
- No side-effects / no trap the loop might have avoided

**The pre-header.** LICM needs a single dedicated block on the loop's entry edge to hoist into. Compilers insert one if absent — a recurring theme: *normalise the CFG (loop-simplify form) first, then transform*.

---

## Induction Variables, Unrolling, Fusion & Interchange

**Induction variables & strength reduction.** A basic IV steps by a constant (`i += 1`). A derived IV is a linear function of it (`j = 4*i`). IV strength reduction turns the multiply into an add:

```c
for(i=0;i<n;i++) a[4*i]=0;
// → keep j, add 4 each iter:
for(j=0; j<4*n; j+=4) *(a+j)=0;
```

**Loop unrolling.** Replicate the body K times, stepping by K. Amortises loop overhead (test/branch/increment), exposes ILP, and feeds vectorisation — at the cost of code size. A *remainder* loop handles leftover iterations.

**Fusion & fission.**
- *Fusion*: merge two adjacent loops over the same range — one loop overhead, better locality & reuse
- *Fission* (distribution): split one loop into several — isolates a vectorisable part, or reduces register pressure

**Loop interchange.** Swap nested loops so the innermost stride is unit, walking arrays in **row-major** order for cache locality:

```c
for j: for i: A[i][j]  // bad: column stride
// interchange →
for i: for j: A[i][j]  // good: unit stride
```

Legal only if the dependence direction permits the swap.

---

## Auto-Vectorisation & SIMD

Modern CPUs offer **SIMD** units (SSE/AVX/AVX-512 on x86, NEON/SVE on ARM, V on RISC-V) that apply one operation to a vector of lanes. The vectoriser rewrites a scalar loop to process several elements per iteration.

```c
// scalar: 1 add per iteration
for (i = 0; i < n; i++)
    c[i] = a[i] + b[i];

// vectorised (width 4): 4 adds per iteration
for (i = 0; i + 4 <= n; i += 4)
    c[i:i+4] = a[i:i+4] + b[i:i+4];  // one VADD
// + a scalar remainder/peel loop for the tail
```

**SLP vs loop vectorisation.** Loop vectorisation widens across iterations; **SLP** (Superword-Level Parallelism) packs independent scalar statements within one block into a vector.

**What blocks it:**
- Loop-carried dependences (`a[i]=a[i-1]`)
- Possible aliasing of pointers (needs runtime checks or `restrict`)
- Unknown trip count, complex control flow, calls
- Non-unit / gather-scatter access patterns

**Helping the compiler.** `restrict` pointers, `#pragma omp simd`, aligned data, simple counted loops. Or write *intrinsics* when the auto-vectoriser gives up.

---

## Why SSA Makes Optimisation Cheaper

In **SSA form** (Part 06) each variable is assigned exactly once, so a use points *directly* at its single definition — def-use chains are explicit and free. Many analyses that needed full data-flow become a sparse graph walk.

**SCCP** — Sparse Conditional Constant Propagation (Wegman & Zadeck). Propagates constants *and* reachability together over the SSA + CFG lattices — strictly more powerful than running constant propagation and unreachable-code removal separately, because each unlocks the other.

**GVN** — Global Value Numbering assigns the same number to expressions that provably compute the same value — even across blocks & through φ-nodes. Subsumes CSE and catches redundancies syntactic CSE misses (e.g. `a+b` vs `b+a`, or values flowing through copies).

**Aggressive DCE** — optimistic: assume *all* code is dead, then mark live only what (transitively) feeds an observable effect (return, store, I/O). Anything unmarked — including whole control-dependence regions — is deleted. SSA makes the mark-and-sweep over def-use edges trivial.

**The sparse advantage.** Classic DFA stores a fact *per variable per program point* — dense and expensive. SSA attaches information to *values*, each with exactly one definition, so facts propagate along def-use edges only where they actually flow. Same answers, far less work — the reason SSA dominates modern middle-ends (LLVM, GCC's GIMPLE-SSA, V8, HotSpot).

---

## It All Compounds — A Before / After

No single optimisation is dramatic; the **cascade** is. Watch a snippet shrink as const-prop, folding, CSE, strength-reduction and DCE fire in sequence.

```c
// before
int w = 4;
int h = w * 2;
int area = w * h;
int t = w * h;
int unused = expensive();
return area + t;

// after const-prop · fold · CSE · DCE
// w=4, h=8, area=32, t=32 ; 'unused' deleted (dead)
return 64;
```

Each pass is simple; together they fold the whole function to `return 64;`. The art is in **ordering** and **repeating** them.

---

## Alias / Points-To Analysis — the Great Limiter

Two pointers **alias** if they may refer to the same storage. Almost every memory optimisation hinges on "can this store affect that load?" — and a conservative "maybe" *kills* the optimisation. Alias analysis is the silent ceiling on what a compiler can do.

```c
*p = 1;
x = *q;        // is x == 1 ? only if p,q alias
*p = 2;        // can we drop the *p=1 store?
               // only if nothing reads through an alias between
```

If `p` and `q` *may* alias, the compiler must keep the load, keep the store, and forgo CSE/DCE/reordering across them.

**Flavours.**
- Type-based (TBAA): different types can't alias (C/C++ strict aliasing)
- Andersen (inclusion, precise, ~O(n³)) vs Steensgaard (unification, near-linear, coarser)
- Flow-/context-sensitive variants trade precision for cost

**Why it limits everything.** Pointer aliasing is undecidable in general, so analysis is always conservative. `restrict` / `__restrict` and strict-aliasing rules exist precisely to *give the compiler back* the no-alias guarantees it cannot prove.

---

## Supporting Structures: Call Graph & Dominators

**The call graph.** Nodes = functions, edges = "may call". The substrate of all interprocedural work: inlining order, propagating facts across calls, devirtualisation.
- Indirect/virtual calls & function pointers make edges *uncertain* — needs points-to or class-hierarchy analysis
- Visited *bottom-up* over SCCs so a callee is optimised & summarised before its callers
- Recursion = a cycle (SCC) in the graph

**Dominators (recap from Part 06).** Block `d` *dominates* `n` if every path from entry to `n` goes through `d`. The dominator tree underpins much of the optimiser:
- SSA φ-placement (dominance frontiers)
- LICM & code hoisting (a hoist target must dominate all uses)
- GVN / redundancy elimination scoping
- Natural-loop detection via back edges to a dominating header

Computed in near-linear time (Lengauer–Tarjan; or the simpler Cooper–Harvey–Kennedy iterative algorithm).

---

## The Phase-Ordering Problem

Optimisations **interact**: one pass creates opportunities for another — or destroys them. There is no universally best order, and the search space is enormous. This is the **phase-ordering problem**, and it has no clean solution.

**Why it's hard:**
- Inlining enables const-prop; const-prop enables DCE; DCE may make more inlining profitable — cyclic dependencies
- A pass can also *obscure* structure a later pass needed (e.g. aggressive lowering hides loop shape from the vectoriser)
- Optimal ordering is intractable to search per-program

**How real compilers cope:**
- A carefully hand-tuned *fixed pipeline*, refined over decades
- *Repetition*: cheap clean-up passes (instcombine, simplifycfg, DCE) run many times between the heavy passes
- Run-to-fixed-point loops within a region
- Research: ML / autotuning to search orderings (e.g. MLGO, autophase) — niche in production

The pragmatic answer: a *good enough, robust* order that works across most programs, with cheap normalising passes interleaved to keep each heavy pass fed with clean IR.

---

## The Real Pipeline: Pass Managers & -O Levels

LLVM and GCC organise optimisation as a sequence of **passes** driven by a **pass manager**. The `-O` level selects which passes run and how aggressively.

`parse` → `mem2reg (SSA)` → `inline` → `SCCP·GVN·CSE` → `LICM·unroll·vec` → `DCE·simplifycfg` → `codegen`

| Level | Intent |
|---|---|
| `-O0` | None; fast compile, debuggable, predictable |
| `-O1` | Cheap wins, little code growth |
| `-O2` | Full opt, no risky size blow-up — the default |
| `-O3` | + aggressive inlining, unroll, vectorise (more size) |
| `-Os` | Optimise for size (O2 minus size-growing passes) |
| `-Oz` | Minimise size at almost any speed cost |
| `-Ofast` | O3 + unsafe maths (`-ffast-math`) — breaks strict IEEE |

**Pass kinds & managers.** Passes are scoped: module, call-graph (SCC), function, loop. LLVM's *New Pass Manager* nests them and threads analysis results (with invalidation) so a transform can request, e.g., dominators or alias info on demand.

**Inspecting it:**

```
clang -O2 -mllvm -print-pipeline-passes
opt -passes='default<O2>' -print-after-all
gcc -O2 -fdump-tree-all -fdump-rtl-all
```

---

## Undefined Behaviour as an Optimisation Licence

C and C++ declare some constructs **undefined behaviour** (UB): the standard imposes *no* requirements. The optimiser is then free to **assume UB never happens** — and to rewrite code on that assumption. A powerful, famously controversial, source of speed.

```c
// signed overflow is UB → compiler assumes it never happens
// → this loop is bounded; i+1 can't wrap:
for (int i = 0; i <= n; i++) { ... }
// → may use a wider IV, vectorise, hoist checks

int v = *p;       // deref → p assumed non-null
if (p) { ... }    // → branch can be deleted!
```

Strict aliasing, no-null-deref, no-signed-overflow, no-OOB and dereferenceable assumptions all feed real optimisations.

**The controversy:**
- UB-based rewrites can *delete* security checks (the famous "null-check removed after deref" CVEs)
- Behaviour can change silently between `-O` levels or compiler versions
- "The compiler is allowed to do *anything*" surprises programmers who expected a sane wrap/trap

**Mitigations.** Sanitizers (`-fsanitize=undefined,address`) catch UB at runtime; flags like `-fwrapv`, `-fno-strict-aliasing`, `-fno-delete-null-pointer-checks` tame specific cases. Safer languages (Rust) eliminate most UB by construction.

---

## Summary & Further Reading

### Key Takeaways

- Optimisation = *improvement*, never optimal; it must be semantics-preserving (the as-if rule)
- Scope spans peephole → local → global → IPO → LTO → whole-program, with PGO orthogonal
- Data-flow = lattice + direction + transfer (gen/kill) + meet; solved by the monotone iterative worklist (terminates: monotonicity + finite height); MFP ≤ MOP
- Four classics: reaching-defs (fwd/may), live-vars (bwd/may), available-exprs (fwd/must), very-busy (bwd/must)
- Transforms cascade: const-fold/prop, copy-prop, CSE, DCE, strength reduction, inlining
- Loops: LICM, IV strength reduction, unroll, fusion/fission, interchange, vectorise
- SSA makes SCCP, GVN & aggressive DCE sparse & cheap; alias analysis is the ceiling on memory opts
- Phase ordering is unsolved — hand-tuned, repeated pipelines; `-O0..-O3/-Os/-Oz`

### Further Reading

- Aho, Lam, Sethi & Ullman — *Compilers: Principles, Techniques & Tools* (Dragon Book), ch. 9
- Cooper & Torczon — *Engineering a Compiler*, ch. 8–10
- Muchnick — *Advanced Compiler Design & Implementation*
- Kildall (1973) — the lattice/data-flow framework; Kam & Ullman on monotone frameworks
- Wegman & Zadeck — *Constant Propagation with Conditional Branches* (SCCP)
- Allen & Cocke — catalogue of program transformations
- The LLVM *Passes* reference & *Writing an LLVM Pass*; `llvm.org`

### Mental Model to Keep

Build the analysis (a sound, conservative fact at every point via a monotone data-flow framework), use it to justify a transformation (legal only when the analysis guarantees safety), then repeat — because every transformation reshapes the IR and reopens opportunities the previous pass could not see.

`Part 07 done` -> `Part 08: Code Generation`
