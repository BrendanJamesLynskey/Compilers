# Compilers — Part 05: Semantic Analysis & Type Systems

**Part 05 — Meaning, Names & Types: What the Grammar Can't Catch**

> After parsing proves a program is well-*formed*, semantic analysis proves it is well-*meaning* — resolving every name to a declaration, enforcing scope, and assigning each expression a type. We build symbol tables, write typing rules as judgements, and reconstruct types from scratch with Hindley–Milner and unification.

`Resolve names` -> `Build scopes` -> `Type-check` -> `Infer` -> `Annotated AST`

Scope - Attributes - `Γ ⊢ e : τ` - Inference

---

## Table of Contents

1. [Topics](#topics)
2. [What Semantic Analysis Does](#what-semantic-analysis-does)
3. [Static vs Dynamic Semantics](#static-vs-dynamic-semantics)
4. [What a Parser Can't Catch](#what-a-parser-cant-catch)
5. [Symbol Tables — Structure & Entries](#symbol-tables--structure--entries)
6. [Scope Tree & the Symbol-Table Stack](#scope-tree--the-symbol-table-stack)
7. [Name Resolution & Binding](#name-resolution--binding)
8. [Attribute Grammars](#attribute-grammars)
9. [Syntax-Directed Translation — an Example](#syntax-directed-translation--an-example)
10. [Annotating the AST — the Typed Tree](#annotating-the-ast--the-typed-tree)
11. [Type Systems — the Axes](#type-systems--the-axes)
12. [Soundness](#soundness)
13. [Typing Judgements — Γ ⊢ e : τ](#typing-judgements--γ--e--τ)
14. [A Worked Derivation](#a-worked-derivation)
15. [Type Inference — Hindley–Milner](#type-inference--hindleymilner)
16. [Unification & the Occurs-Check](#unification--the-occurs-check)
17. [Unification — a Worked Example](#unification--a-worked-example)
18. [Let-Polymorphism & Generalisation](#let-polymorphism--generalisation)
19. [Subtyping & Variance](#subtyping--variance)
20. [Generics, Type Classes & Overloading](#generics-type-classes--overloading)
21. [Coercions, Conversions & Aliases](#coercions-conversions--aliases)
22. [Other Semantic Checks](#other-semantic-checks)
23. [Frontiers — Borrows, Dependent & Refinement Types](#frontiers--borrows-dependent--refinement-types)
24. [Diagnostics & Error Recovery](#diagnostics--error-recovery)
25. [Summary & Further Reading](#summary--further-reading)

---

## Topics

The parser handed us a tree that is *syntactically* legal. This deck makes it *meaningful*: every identifier bound, every operation type-checked, every static rule the grammar couldn't express now enforced.

### Names & Scope
- Static vs dynamic semantics; what the parser can't catch
- Symbol tables: structure, entries, name resolution
- Lexical scope, nesting, shadowing, namespaces

### Attributes & Rules
- Attribute grammars: synthesized vs inherited
- S- and L-attributed; syntax-directed translation
- Typing judgements `Γ ⊢ e : τ` and inference rules

### Type Systems & Inference
- Static/dynamic, strong/weak, nominal/structural
- Soundness — "well-typed programs don't go wrong"
- Hindley–Milner, Algorithm W, unification, let-polymorphism

### Richer Checks & Diagnostics
- Subtyping & variance, generics, type classes, coercions
- Definite assignment, returns, exhaustiveness, borrows
- Why HM error messages are notoriously hard

---

## What Semantic Analysis Does

Parsing answers "*is this a sentence of the grammar?*" Semantic analysis answers "*does this sentence mean anything sensible?*" It is the last phase of the **front end** — its output is an **annotated, type-checked AST** plus a populated symbol table.

> *Diagram:* Parser → raw AST → Semantic Analysis (resolve · scope · type, + symbol table) → typed AST → IR gen; with type/scope errors branching off.

**Core jobs:**
- *Name resolution* — bind each use to a declaration
- *Type checking / inference* — give every node a type
- *Static checks* — definite assignment, returns, exhaustiveness

**Why a separate phase?** These rules are *context-sensitive* — whether `x` is in scope depends on declarations elsewhere. Context-free grammars cannot express that, so it lives outside the parser. Once this phase signs off, the rest of the compiler may *assume* the program is meaningful — IR generation never re-checks a type.

---

## Static vs Dynamic Semantics

A language's *semantics* — what programs mean — splits into what can be decided before the program runs and what only emerges as it runs.

**Static semantics** — rules checkable at compile time from the text alone (the job of semantic analysis):
- Names declared before use; types match operators
- Function arity & argument types agree
- Every path through a non-`void` function returns
- Often called the language's *well-formedness* rules

**Dynamic semantics** — what a well-formed program *does* when executed, defined by an operational/denotational/axiomatic semantics and enforced at run time:
- Array index in bounds; pointer non-null; divisor non-zero
- Order of evaluation; effects; the actual computed values
- Caught by the runtime (a trap/exception), not the compiler

**The dividing line is a design choice.** The more a language decides statically, the more errors are caught before shipping — but the more the programmer must spell out. A null-dereference is dynamic in Java but moved *static* in Kotlin/Rust via the type system. Pushing checks left is a recurring theme of modern language design.

---

## What a Parser Can't Catch

Every snippet below is a perfectly valid *parse tree* — grammatically flawless. Each is rejected only by semantic analysis, because the offending property is context-sensitive.

```c
int main(void) {
    y = 3;            // undeclared 'y'
    int x = "hello";  // type error: char* → int
    int x = 7;        // redeclaration of 'x'
    foo(1, 2);        // arity: foo takes 1 arg
    return;           // missing return value
}
```

| Error | Why the parser misses it |
|-------|--------------------------|
| Undeclared name | Needs the set of names in scope — not in the CFG |
| Type mismatch | Needs each subexpression's type |
| Redeclaration | Needs to know what's already bound here |
| Wrong arity | Needs the callee's signature |
| Missing return | Needs control-flow reachability |
| Break outside loop | Needs surrounding context |

"Variable declared before use" is the classic example of a *non*-context-free property. You'd need a context-sensitive grammar to encode it — impractical, so we use ad-hoc checks plus a symbol table instead.

---

## Symbol Tables — Structure & Entries

The **symbol table** maps names to everything the compiler knows about them. Lookups are fast and frequent, so it is usually a hash table — or a stack of them, one per scope.

| Field | Holds |
|-------|-------|
| name | the identifier (the hash key) |
| kind | var / function / type / param / field |
| type | `int`, `(int,int)→bool`, … |
| scope | nesting level / owning block |
| location | stack offset / register / global addr |
| attrs | const, mutable, visibility, line/col |

Operations: `insert(name, entry)` on declaration; `lookup(name)` on use — searching the current scope, then enclosing scopes outward. `enterScope()` / `exitScope()` push and pop tables.

```cpp
struct Symbol {
  std::string  name;
  Kind         kind;   // Var, Func, Type, ...
  Type*        type;
  int          scopeLevel;
  Location     loc;    // offset / reg / addr
  Attrs        attrs;  // const, mut, visibility
};

struct Scope {                                  // one hash table; scopes nest
  std::unordered_map<std::string, Symbol> table;
  Scope* parent;   // enclosing scope (or null)
};

Symbol* lookup(Scope* s, const std::string& n) {
  for (; s; s = s->parent)
    if (auto it = s->table.find(n); it != s->table.end())
      return &it->second;   // first hit wins → shadowing
  return nullptr;           // undeclared
}
```

---

## Scope Tree & the Symbol-Table Stack

Nested blocks form a tree of scopes. At any point the live scopes are a **stack**, root at the bottom. `lookup` walks the stack outward; the *innermost* binding wins — that is shadowing.

```c
int g = 0;            // global
void f(int p) {       // fn f
  int x = p;
  {                   // block B1
    int x = 9;        // shadows outer x
    int y = x + g;    // uses inner x (=9) and global g
  }
  return x;           // outer x (=p)
}
```

> *Diagram:* a three-level scope tree — `Global {g, f}` → `f {p, x}` → `B1 {x (shadows), y}` — with `lookup` walking outward from the top of the stack. The inner `x` and outer `x` coexist; a reference resolves to whichever scope is nearer on the stack. Scopes are pushed on `enterScope()` and popped on `exitScope()`.

---

## Name Resolution & Binding

**Binding** connects each *use* of a name to the *declaration* it refers to. Resolution is the algorithm that finds that declaration — and the rules are surprisingly subtle.

- **Lexical (static) scope** — a name binds to the nearest enclosing declaration *in the program text*, decidable at compile time. The dominant rule (C, Java, Rust, ML, Scheme…).
- **Dynamic scope** — a name binds to the most recent definition on the *call stack* at run time. Rare (early Lisp, Bash, Emacs Lisp specials); hard to reason about, mostly abandoned.
- **Forward references** — a use that precedes its declaration. Handled by a declaration-gathering pass: collect all top-level names first, then resolve bodies — so mutual recursion just works.
- **Namespaces & overloading** — one identifier can name different things in different *namespaces* (C: tags vs ordinary vs labels). With overloading, a name maps to a *set* of entries; resolution picks one by argument types.

**Why lexical won:** under dynamic scope, what `x` means depends on *who called you* — you can't read a function in isolation. Lexical scope makes a binding determinable from the source alone, essential for both humans and closures.

---

## Attribute Grammars

Knuth (1968) gave a formal way to attach *meaning* to a grammar: decorate each production with **attributes** and **semantic rules** that compute them. This is the theory behind syntax-directed translation.

- **Synthesized attributes** — computed bottom-up: a node's value is built from its *children* (a node's type from its operands; a constant's value). Flow *up* the tree.
- **Inherited attributes** — computed top-down: a node's value comes from its *parent* and/or *siblings* (the current scope, the expected type). Flow *down* and across.
- **S-attributed** — uses *only* synthesized attributes; evaluable in a single bottom-up pass, exactly what a bottom-up (LR) parser does on reductions.
- **L-attributed** — synthesized + inherited, but each inherited attribute depends only on the parent and on *left* siblings; evaluable in one left-to-right depth-first pass, fitting top-down (LL) / recursive-descent parsing. Every S-attributed grammar is L-attributed.

In practice few compilers run a full attribute-grammar engine; they hand-write tree walks. But the *vocabulary* — what flows up, what flows down, and the legal dependency order — is exactly how we reason about an AST traversal.

---

## Syntax-Directed Translation — an Example

Attaching a synthesized `.type` attribute to an expression grammar gives a one-pass bottom-up type-checker. Each rule combines its children's types into its own.

```
E → E1 + E2   { E.type = unify(E1.type, E2.type) }
E → E1 * E2   { E.type = numeric(E1.type, E2.type) }
E → ( E1 )    { E.type = E1.type }
E → num       { E.type = int }
E → real      { E.type = float }
E → id        { E.type = lookup(id.name).type }
```

`.type` is synthesized — this grammar is S-attributed, so an LR parser can type-check during reduction.

> *Diagram:* typing `3 + x * 2.0` with `x : float`. Leaves get types (`3:int`, `x:float`, `2.0:float`); types flow up — `x * 2.0 : float`, then `3 + (…) : float`, with the `int` literal `3` coerced to `float`.

---

## Annotating the AST — the Typed Tree

The product of type checking is a **typed AST**: every expression node carries a resolved type, every identifier a pointer into the symbol table, and implicit conversions are made explicit as `Coerce` nodes.

> *Diagram:* `double r = n + 1;` with `n:int, r:double`. The tree is `Assign:double` → `Var r:double` (→ symtab[r]) and `Coerce int→double` → `Add:int` → `Var n:int`, `Lit 1:int`.

**Implicit made explicit.** The `int` result of `n + 1` must become a `double` to store in `r`. The checker inserts a `Coerce` node so the back end never re-derives the conversion.

**Decorations carried.** Each node gains a resolved *type*, a *symbol* reference for identifiers, and sometimes a constant-folded value or an effect annotation.

---

## Type Systems — the Axes

A **type system** is a tractable, syntactic method for proving the absence of certain bad behaviours by classifying values. Languages sit at different points on several independent axes.

| Axis | One end | Other end | Examples |
|------|---------|-----------|----------|
| When checked | Static (compile time) | Dynamic (run time) | Rust / C++ vs Python / JS |
| Strength | Strong (few implicit holes) | Weak (free reinterpretation) | Haskell / Python vs C |
| Manifest? | Manifest (annotations written) | Inferred (reconstructed) | Java vs ML / Haskell |
| Identity | Nominal (by name) | Structural (by shape) | Java / Rust vs TypeScript / Go interfaces |

**Strong ≠ static** — orthogonal axes, endlessly confused. Python is *dynamically* but *strongly* typed (`"a" + 1` raises, never silently coerces). C is *statically* but *weakly* typed (casts reinterpret bits). "Strong" is informal; "static" is precise.

**Nominal vs structural** — two records with identical fields are the *same* type structurally but *different* types nominally unless one declares it implements the other. Nominal catches "accidental" matches; structural is more flexible.

---

## Soundness

Milner's 1978 slogan, **"well-typed programs don't go wrong"**, is the promise a type system makes: if the checker accepts a program, certain run-time errors *cannot* happen. Soundness is proved as two lemmas.

- **Progress** — a well-typed term is either a *value* or it can take an evaluation *step*; it never gets "stuck" in a meaningless state (like adding a function to an integer).
- **Preservation (subject reduction)** — if a well-typed term takes a step, the result is still well-typed, *with the same type*. Types are an invariant of execution.
- **Progress + Preservation = Safety** — together they guarantee a well-typed program never reaches a stuck state (Wright & Felleisen, 1994).

**The two ways to fail:**
- *Unsound* — accepts some program that *does* go wrong (a hole; e.g. Java array covariance throws `ArrayStoreException`).
- *Incomplete* — rejects some program that would have been fine (the conservative price of decidability).

Type-checking a Turing-complete property is undecidable, so every sound, decidable system is necessarily *incomplete* — it must reject some safe programs. Good design minimises the surprising rejections.

---

## Typing Judgements — Γ ⊢ e : τ

Type rules are written as **inference rules**: premises above the line, conclusion below. The judgement `Γ ⊢ e : τ` reads "*in context Γ, expression e has type τ*". Γ is the typing environment — the symbol table, formally.

```
                              x : τ ∈ Γ
(T-Int) ─────────────   (T-Var) ─────────────
        Γ ⊢ n : int            Γ ⊢ x : τ

         Γ ⊢ e1 : bool   Γ ⊢ e2 : τ   Γ ⊢ e3 : τ
(T-If) ───────────────────────────────────────────
            Γ ⊢ if e1 then e2 else e3 : τ

         Γ ⊢ e1 : σ → τ      Γ ⊢ e2 : σ
(T-App) ───────────────────────────────
                Γ ⊢ e1 e2 : τ
```

Reading the rules:
- **T-Int** — a literal needs no premises; an axiom.
- **T-Var** — look the name up in Γ.
- **T-If** — condition `bool`; both branches *the same* τ.
- **T-App** — argument type must match the function's domain σ; result is the codomain τ.

**Type checking = proof search.** To type an expression is to build a *derivation tree* of these rules with the expression at its root. A type checker is a proof searcher; a type *error* is a failure to find any derivation.

**T-Abs (functions):** to type `λx:σ. e`, extend the context: if `Γ, x:σ ⊢ e : τ` then `Γ ⊢ λx:σ.e : σ→τ`. Entering a binder *extends* Γ — exactly `enterScope`.

---

## A Worked Derivation

Type `if b then x else 0` in context `Γ = { b : bool, x : int }`. We compose the rules into a derivation tree — leaves are axioms, the root is the goal.

```
 b:bool∈Γ        x:int∈Γ       (T-Int)
─────────────   ─────────────  ─────────────
Γ ⊢ b : bool    Γ ⊢ x : int    Γ ⊢ 0 : int
──────────────────────────────────────────── (T-If)
        Γ ⊢ if b then x else 0 : int
```

Top to bottom: the leaves discharge T-Var / T-Int; the `if`'s three premises are satisfied (guard `bool`, both arms `int`); T-If fires, so the whole term is `int`.

**Where it would fail:** had the `else` arm been `"hi"`, the two branch premises (`int` vs `string`) couldn't both equal one τ — no derivation exists, so it's a type error.

---

## Type Inference — Hindley–Milner

The rules above need types written down. **Hindley–Milner** (HM) *reconstructs* them: with *no* annotations it finds the most general type, if one exists. Discovered independently by Hindley (1969) and Milner (1978); Milner's **Algorithm W** is the implementation.

```haskell
compose f g x = f (g x)
-- W infers:  compose :: (b -> c) -> (a -> b) -> a -> c

twice f x = f (f x)
-- twice :: (a -> a) -> a -> a

id x = x
-- id :: a -> a   (a is a fresh type variable)
```

The algorithm, in three moves:
1. Give every unknown a *fresh type variable*.
2. Walk the AST emitting *equality constraints* between types.
3. *Unify* all constraints into one substitution.

**Principal types** — HM guarantees a single most-general type from which every other valid type is an instance. `id`'s principal type is `a → a`; `Int → Int` is just one instance.

**Why HM is the sweet spot:** decidable & efficient (near-linear in practice); no annotations required; powers ML, OCaml, Haskell, F#, plus Rust & Swift local inference and Elm.

**The boundary:** HM trades power for decidability — no first-class (rank-N) polymorphism, no subtyping. Add those and full inference becomes undecidable, so annotations creep back in.

---

## Unification & the Occurs-Check

**Unification** (Robinson, 1965) is HM's engine: given two type terms, find the *most general substitution* making them equal — or fail. It solves the equality constraints W generates.

```
unify(t1, t2):
  t1, t2 = resolve(t1), resolve(t2)   # follow bound vars
  if t1 == t2:                  return            # done
  if t1 is a var α:    occurs_check(α, t2); bind α := t2; return
  if t2 is a var β:    occurs_check(β, t1); bind β := t1; return
  if t1 = C(a1..an) and t2 = C(b1..bn):  # same constructor
       for i: unify(ai, bi)              # arity must match
       return
  fail("cannot unify " + t1 + " with " + t2)
```

**The occurs-check** — before binding `α := t`, verify `α` does *not* occur inside `t`. Binding `α := α → β` would build an *infinite* type. The occurs-check is exactly what rejects the untypable `λx. x x`.

Three outcomes: same constructor → recurse on arguments pairwise; variable → bind it (after the occurs-check); clash (`int` vs `bool`, or wrong arity) → type error.

**Most general unifier (mgu)** — unification returns the substitution that commits to nothing more than necessary. That minimality is precisely why HM yields *principal* types. Implementation: type vars are mutable cells (union-find); `resolve` path-compresses, which is why HM is near-linear.

---

## Unification — a Worked Example

Infer the type of `apply f x = f x`. Give fresh variables, generate the one constraint `f` imposes, and unify.

```
f : α          x : β        (fresh)
apply : α → β → γ           (γ fresh, result)

the call  f x  demands:
   α  =  β → γ        ← f used as a function

unify(α, β → γ):
   α is a var, occurs-check(α, β→γ) ✓ (α ∉ β→γ)
   bind  α := β → γ

substitute back:  apply : (β → γ) → β → γ
generalise free vars β, γ:   apply :: (b -> c) -> b -> c     ✓ principal
```

> *Diagram:* `unify(α, β → γ)` — α is a variable, `β → γ` is an arrow constructor; the occurs-check `α ∈ β→γ?` is *no*, so it is safe to bind, giving the mgu `{ α ↦ β → γ }`.

---

## Let-Polymorphism & Generalisation

HM's polymorphism lives at `let`. When a binding is generalised, its free type variables are **closed over** into a *type scheme* `∀α. τ` — each *use* then gets fresh copies (instantiation).

```haskell
let id = \x -> x in
  (id 3, id True)
-- id generalised to  ∀a. a -> a
-- use 1: instantiate a := Int   → id 3
-- use 2: instantiate a := Bool  → id True
-- BOTH type-check ✓

-- λ-bound params are NOT generalised:
\id -> (id 3, id True)   -- TYPE ERROR
-- one monomorphic α can't be both Int and Bool
```

- **Generalisation (gen)** — at a `let`, quantify over type variables free in τ but *not* free in the environment Γ; those are still constrained elsewhere and must stay fixed.
- **Instantiation (inst)** — at each use of a scheme `∀α.τ`, replace the bound α with fresh variables so distinct uses don't constrain each other.
- **The value restriction** — generalising arbitrary expressions is *unsound* with mutable references (a polymorphic `ref` leaks). ML restricts generalisation to *syntactic values* — the famous value restriction.

---

## Subtyping & Variance

**Subtyping** `S <: T` means an `S` may stand in wherever a `T` is expected (Liskov substitution). The hard question: how does subtyping lift through *type constructors*?

| Variance | Rule (if S <: T) | Where |
|----------|------------------|-------|
| Covariant | `F<S> <: F<T>` | read-only, return types |
| Contravariant | `F<T> <: F<S>` | function argument positions |
| Invariant | neither direction | mutable cells |

**The function rule:** `(A→B) <: (C→D)` iff `C <: A` (contravariant in the argument) *and* `B <: D` (covariant in the result). Arguments flip; results follow.

**Why mutable = invariant:** if `Cat <: Animal` made `List<Cat> <: List<Animal>`, you could insert a `Dog` into the aliased `List<Animal>` and break the `List<Cat>`. Java's covariant *arrays* have exactly this hole — checked at run time via `ArrayStoreException`.

In practice: Scala `+T`/`-T` declaration-site variance; C# `out`/`in` on generic params; Java use-site `? extends` / `? super`.

---

## Generics, Type Classes & Overloading

"Polymorphism" — one name, many behaviours (Strachey, Cardelli–Wegner) — comes in distinct flavours, each resolved differently by the checker.

- **Parametric** — one body, *any* type, uniformly: `List<T>`, `fn id<T>(x:T)->T`. Generics / ML polymorphism; the code can't inspect `T`. *Impl:* erasure (Java/HM) or monomorphisation (Rust/C++ templates).
- **Ad-hoc** — different code per type, chosen by type. *Overloading* & **type classes / traits**: `(+)` on `Int` vs `Float`, `Show a`. *Impl:* the compiler passes a hidden *dictionary* of methods.
- **Subtype** — inclusion polymorphism via `S <: T`, the OO kind. A `Circle` works wherever a `Shape` does, dispatched dynamically. *Impl:* vtables.

**Type classes vs overloading** — Haskell's classes are *principled* overloading: a class declares an interface (`class Eq a where (==) :: a→a→Bool`); instances supply it; constraints (`Eq a =>`) thread through inference. Rust *traits* are the same idea with coherence rules.

**Overload resolution** — with C++/Java overloading the checker ranks candidates by how well argument types match (exact > promotion > conversion), preferring the most specific; ambiguity is an error. This interacts badly with full inference, so HM languages avoid raw overloading.

---

## Coercions, Conversions & Aliases

Beyond accepting or rejecting, the checker sometimes *inserts work*: implicit conversions that bridge nearly-compatible types — the source of much convenience and many bugs.

**Implicit coercions:**
- *Widening* — `int → long → double`, value-preserving, usually safe & silent.
- *Narrowing* — `double → int`, lossy; many languages require an explicit cast.
- Inserted as the `Coerce` nodes from the typed-AST slide.

**The hazard:** C's integer promotions & usual arithmetic conversions are a notorious footgun — `unsigned`/`signed` mixing flips comparisons. Strongly-typed languages keep coercions few and explicit.

**Type aliases vs new types** — an *alias* (`typedef`, `type X = Y`) is the *same* type under a new name, interchangeable, no safety added. A *nominal newtype* (Haskell `newtype`, Rust tuple-struct) is a *distinct* type, so `Metres` can't be mixed with `Seconds`.

**Coercion as subtyping** — one can model coercions as a subtype relation with an inserted conversion function. Scala's `implicit` conversions and C++ converting-constructors formalise exactly this — powerful, and easy to abuse.

---

## Other Semantic Checks

Type checking is the headline, but the same phase runs a battery of other **flow-sensitive** analyses — many are small dataflow problems over the AST/CFG.

- **Definite assignment** — every local is written on *all* paths before it is read. Java & C# reject reads of possibly-unassigned variables.
- **Reachability & returns** — code after a `return` is unreachable; a non-`void` function must return on every path. A forward CFG dataflow check.
- **Exhaustiveness** — `match` must cover every case of a sum type; missing variants warn or error. Rust/ML/Swift enforce this — a key safety win.
- **Const-ness & lvalues** — you may not assign to a `const`/`final`; the LHS of `=` must be an *lvalue* (has an address), not a temporary.
- **Effects & purity** — effect systems track I/O / exceptions / async in the type (Koka, checked exceptions, `async` colouring) — what a function may *do*, not just return.
- **Initialisation order** — use-before-init of globals, cyclic `static` init, `this` escaping a constructor.

These reuse the symbol table and run as AST/CFG traversals — the attribute-grammar machinery applied to control flow rather than types.

---

## Frontiers — Borrows, Dependent & Refinement Types

The richest semantic checks push the type system to encode *memory safety*, *resource ownership* and even *logical specifications*.

- **Rust's borrow checker** — every value has one *owner*; ownership *moves*. Either many `&` shared *or* one `&mut`, never both. **Lifetimes** `'a` are type variables bounding how long a borrow is valid. A flow-sensitive (NLL) analysis — no GC, no use-after-free, all at compile time.
- **Dependent types** — types that depend on *values*: `Vec n A`, a vector of statically-known length `n`, so `append` provably returns length `n+m`. Idris, Agda, Coq, Lean. Type-checking subsumes *proof*-checking.
- **Refinement types** — a base type plus a predicate: `{ v:Int | v > 0 }`. An SMT solver discharges the obligations. Liquid Haskell, F*, Dafny — lighter than full dependent types, still catches off-by-one & division-by-zero statically.

**The common thread:** all three move properties that were *dynamic* (a crash, a leak, a bad index) into the *static* type system — the same "shift-left" principle that motivated type checking in the first place, taken to its limit.

---

## Diagnostics & Error Recovery

A type system is only as good as its **error messages**. The cruel irony: the more inference a system does, the *harder* good diagnostics become — the error surfaces far from its cause.

**Why HM errors are hard:**
- Unification fails at *one* constraint, but the real mistake may be *anywhere* that contributed to it.
- "Cannot unify `Int` with `Bool`" names the symptom, not the bug.
- Reported location depends on traversal *order* — left-to-right bias blames the wrong expression.
- Inferred types can be huge & full of fresh variables (`t47 → t48`).

**Making them better:** track the *provenance* of every constraint to point at source spans; prefer the user's annotation as "expected" and the inferred type as "actual"; suggest fixes and show the minimal conflicting set (type-error slicing).

```
error[E0308]: mismatched types
  --> src/main.rs:3:18
   |
 3 |     let x: i32 = "hello";
   |            ---   ^^^^^^^ expected `i32`, found `&str`
   |            expected due to this
   = help: try parsing with `.parse()`
```

**Error recovery in the checker** — after an error, don't cascade: assign the broken node an *error type* (⊥) that unifies with *anything* and suppresses downstream complaints. One mistake → one message, not fifty.

---

## Summary & Further Reading

### Key Takeaways
- Semantic analysis enforces *context-sensitive* rules the CFG can't — names, types, arity.
- Symbol tables (scoped hash tables) drive name resolution; lexical scope & shadowing fall out of stack lookup.
- Attribute grammars frame the AST walk: *synthesized* flows up, *inherited* flows down (S- / L-attributed).
- Typing rules are inference rules `Γ ⊢ e : τ`; checking = building a derivation.
- Hindley–Milner reconstructs *principal* types via fresh vars + constraint *unification* (with the occurs-check).
- Let-polymorphism generalises; subtyping, variance, generics & type classes extend the system.
- Soundness = progress + preservation: "well-typed programs don't go wrong".

### Exercise
Hand-run Algorithm W on `map f xs = case xs of [] -> []; (y:ys) -> f y : map f ys` and confirm the principal type `(a -> b) -> [a] -> [b]`.

### Further Reading
- **Pierce** — *Types and Programming Languages* (TAPL), 2002: the definitive text on type systems, soundness, subtyping & inference.
- **Aho, Lam, Sethi & Ullman** — the *Dragon Book*, ch. 5 (syntax-directed translation) & ch. 6 (type checking).
- **Milner (1978)** — "A Theory of Type Polymorphism in Programming".
- **Damas & Milner (1982)** — principal-type theorem for Algorithm W.
- **Robinson (1965)** — the unification algorithm.
- **Krishnamurthi** — *PLAI* (free online).

### Next → Part 06
**Intermediate Representations** — lowering the typed AST into three-address code & SSA, control-flow graphs, and the IR that the optimiser will chew on.

`Part 05 ✓` -> `Part 06: Intermediate Representations`
