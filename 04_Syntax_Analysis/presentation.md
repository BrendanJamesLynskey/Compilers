# Compilers — Part 04: Syntax Analysis (Parsing)

**Part 04 — From a Token Stream to a Tree of Meaning**

> The parser is where a flat stream of tokens becomes a tree. We map the grammar-class landscape — LL, LR, SLR, LALR(1), GLR, PEG — build parsers by hand and by generator, climb expressions with Pratt parsing, trace a shift-reduce automaton, and resolve the conflicts that every real grammar throws at you.

`Grammar` -> `Top-Down` -> `Bottom-Up` -> `Conflicts` -> `AST`

Grammars - FIRST/FOLLOW - LR Tables - AST Construction

---

## Table of Contents

1. [Topics](#topics)
2. [What the Parser Does](#what-the-parser-does)
3. [Context-Free Grammars](#context-free-grammars)
4. [Derivations & Sentential Forms](#derivations--sentential-forms)
5. [Parse Tree vs AST](#parse-tree-vs-ast)
6. [Ambiguity](#ambiguity)
7. [Encoding Precedence & Associativity](#encoding-precedence--associativity)
8. [The Grammar-Class Hierarchy](#the-grammar-class-hierarchy)
9. [Who Can Parse What — a Map](#who-can-parse-what--a-map)
10. [Top-Down: Recursive Descent](#top-down-recursive-descent)
11. [FIRST, FOLLOW & the LL(1) Table](#first-follow--the-ll1-table)
12. [Pratt Parsing / Precedence Climbing](#pratt-parsing--precedence-climbing)
13. [Left-Recursion & Left-Factoring](#left-recursion--left-factoring)
14. [Bottom-Up: Shift-Reduce Parsing](#bottom-up-shift-reduce-parsing)
15. [LR(0) Items & the Canonical Automaton](#lr0-items--the-canonical-automaton)
16. [SLR vs LR(1) vs LALR(1)](#slr-vs-lr1-vs-lalr1)
17. [The ACTION / GOTO Tables](#the-action--goto-tables)
18. [Conflicts: Shift/Reduce & Reduce/Reduce](#conflicts-shiftreduce--reducereduce)
19. [Parser Generators & Toolkits](#parser-generators--toolkits)
20. [Error Recovery & Diagnostics](#error-recovery--diagnostics)
21. [AST Design & Construction](#ast-design--construction)
22. [Top-Down vs Bottom-Up — Choosing](#top-down-vs-bottom-up--choosing)
23. [Summary & Further Reading](#summary--further-reading)

---

## Topics

Parsing is the heart of the front end. We start from context-free grammars, contrast the two great parsing families, and finish at the AST that the rest of the compiler consumes.

- **Grammars & Trees** — the parser's job; CFGs; derivations & sentential forms; parse trees vs ASTs; ambiguity; precedence & associativity.
- **Top-Down** — recursive descent & predictive LL(1); FIRST/FOLLOW & the LL(1) table; left-recursion & left-factoring; Pratt parsing.
- **Bottom-Up** — shift-reduce, the stack & handles; LR(0) items & the canonical automaton; SLR/LR(1)/LALR(1); ACTION/GOTO; conflicts.
- **Tools & Practice** — the grammar-class hierarchy; yacc/bison, ANTLR, menhir, tree-sitter, PEG; error recovery; AST design & semantic actions.

---

## What the Parser Does

The lexer (Part 03) handed us a flat stream of tokens. The parser checks that those tokens form a valid sentence of the source language's grammar and, in the process, recovers the *structure* the flat stream lost — nesting, grouping, precedence.

Three jobs:

- **Recognise** — is this a sentence of the language?
- **Build** — produce a tree capturing structure.
- **Diagnose** — report and recover from errors.

Why a grammar at all? Regular expressions (the lexer's tool) cannot count nesting — they can't match balanced brackets. Recursion in the language needs a context-free grammar and a stack. The lexer/parser split mirrors the Chomsky hierarchy: regular languages for tokens, context-free languages for phrase structure. Anything beyond — declared-before-use, type agreement — is left to *semantic* analysis (Part 05).

---

## Context-Free Grammars

A CFG is a 4-tuple `G = (N, Σ, P, S)`: non-terminals `N`, terminals `Σ`, productions `P`, and a distinguished start symbol `S`. Each production rewrites *one* non-terminal into a string of terminals and non-terminals.

```ebnf
Expr   → Expr "+" Term
       | Expr "-" Term
       | Term
Term   → Term "*" Factor
       | Term "/" Factor
       | Factor
Factor → "(" Expr ")"
       | number
       | ident
```

- **Terminals (`Σ`)** — the token classes from the lexer; the leaves of every tree. They never appear on a production's left-hand side.
- **Non-terminals (`N`)** — syntactic variables naming *phrases*: `Expr`, `Term`, `Stmt`. Defined by their productions.
- **Productions (`P`)** — rewrite rules `A → α`. "Context-free" because the LHS is a *single* non-terminal, replaceable regardless of context.
- **Start symbol (`S`)** — the root non-terminal. The language `L(G)` is every terminal string derivable from `S`.

---

## Derivations & Sentential Forms

A derivation applies productions step by step, each rewriting one non-terminal, until only terminals remain. Each intermediate string is a *sentential form*; one with no non-terminals left is a *sentence*.

**Leftmost derivation** of `a + b * c` (always rewrite the leftmost non-terminal):

```
Expr
⇒ Expr + Term         (Expr→Expr+Term)
⇒ Term + Term         (Expr→Term)
⇒ Factor + Term       (Term→Factor)
⇒ a + Term            (Factor→ident)
⇒ a + Term * Factor   (Term→Term*Factor)
⇒ a + Factor * Factor
⇒ a + b * Factor
⇒ a + b * c
```

A **rightmost derivation** always rewrites the rightmost non-terminal instead. The key insight:

- LL parsers trace a *leftmost* derivation, top-down.
- LR parsers trace a *rightmost* derivation **in reverse**, bottom-up — this is why they matter.
- Different leftmost/rightmost derivations can still yield the *same* parse tree.

Notation: `α ⇒ β` is one step; `α ⇒* β` is zero-or-more steps. `L(G) = { w ∈ Σ* | S ⇒* w }`.

---

## Parse Tree vs AST

The **parse tree** (concrete syntax tree) records *every* grammar symbol used — including chains like `Term→Factor` and the parentheses. The **AST** (abstract syntax tree) keeps only the essential structure.

For `a + b * c`, the concrete parse tree has eleven nodes (`Expr`, `Term`, `Factor` chains plus leaves); the AST collapses to five:

```
    +
   / \
  a   *
     / \
    b   c
```

The AST drops single-child chains and punctuation; `*` sits below `+`, so precedence survives as *shape*, not as token order.

---

## Ambiguity

A grammar is **ambiguous** if some sentence has more than one parse tree (equivalently, more than one leftmost derivation). Ambiguity means the structure — and therefore the *meaning* — is undefined.

**The classic expression grammar:**

```ebnf
E → E "+" E
  | E "*" E
  | number
```

For `1 + 2 * 3` this allows two trees: `(1+2)*3` and `1+(2*3)`. The grammar never says which — it has no built-in precedence.

**The dangling else:**

```ebnf
S → "if" E "then" S "else" S
  | "if" E "then" S
  | other
```

In `if a then if b then s1 else s2`, does `else` bind to the inner or outer `if`? Two trees.

Fixes: rewrite to an unambiguous grammar (stratify by precedence); add a disambiguating rule ("`else` matches the *nearest* `if`"); or supply precedence/associativity declarations to the generator. Some ambiguity is *inherent* — no equivalent unambiguous grammar exists — but the cases compilers meet are almost always fixable.

---

## Encoding Precedence & Associativity

The cure for the ambiguous expression grammar is to **stratify** it into one non-terminal per precedence level. Higher precedence sits *lower* in the grammar (closer to the leaves), so it binds tighter.

```ebnf
Expr   → Expr "+" Term  | Term      // + is left-assoc, lowest precedence
Term   → Term "*" Factor | Factor
Factor → "-" Factor | Power          // unary minus
Power  → Atom "^" Power | Atom       // ^ is RIGHT-assoc
Atom   → number | "(" Expr ")"
```

Associativity falls out of the recursion direction: *left*-recursion (`Expr→Expr+Term`) gives left-associativity; *right*-recursion (`Power→Atom^Power`) gives right-associativity.

| Level | Operators | Assoc. |
|---|---|---|
| lowest | `+ -` | left |
| ↓ | `* / %` | left |
| ↓ | unary `-` `!` | right |
| highest | `^` (power) | right |

Two routes to the same end: bake precedence into the grammar's *shape* (above), keep a flat grammar and supply precedence declarations to the generator (bison's `%left` / `%right`), or climb it numerically (Pratt parsing — below).

---

## The Grammar-Class Hierarchy

Not every CFG can be parsed deterministically with bounded lookahead. The parsing classes form a **containment hierarchy** — each is the set of *grammars* a given method can handle. The practical *k*=1 picture:

```
LL(1) ⊊ SLR(1) ⊊ LALR(1) ⊊ LR(1) ⊊ unambiguous CFGs ⊊ all CFGs (GLR)
```

Recursive descent realises LL(1). PEG / packrat use ordered choice, are unambiguous by construction, run in linear time, but sit in a *different* class. Note that LL and LR are incomparable in general for arbitrary *k* — the clean chain above is the *k*=1 practical view. ANTLR's ALL(*) and GLR sit outside it, trading determinism for power.

---

## Who Can Parse What — a Map

Each class trades grammar generality against implementation simplicity and error quality. The dominant production choice is hand-written recursive descent (an LL flavour) or LALR(1) via a generator.

| Class | Direction | Derivation | Lookahead | Typical use |
|---|---|---|---|---|
| LL(k) | top-down | leftmost | *k* tokens, predictive | recursive descent; hand-written |
| LL(*) / ALL(*) | top-down | leftmost | adaptive, unbounded | ANTLR 4 |
| SLR(1) | bottom-up | rightmost (rev.) | FOLLOW-based | teaching; small tables |
| LALR(1) | bottom-up | rightmost (rev.) | merged LR(1) states | yacc, bison, menhir |
| LR(1) | bottom-up | rightmost (rev.) | full 1-token contexts | menhir `--canonical` |
| GLR | bottom-up | all parses | forks on conflict | tree-sitter, Bison `%glr`, Elkhound |
| PEG / packrat | top-down | ordered choice | unbounded, memoised | parser combinators, packrat |

**Rule of thumb:** want the best diagnostics and full control → hand-written recursive descent. Want a grammar that's a near-direct transcription of the spec → an LALR(1)/LR generator. Need to parse a fuzzy or ambiguous real-world language (editors) → GLR (tree-sitter).

**Power vs cost:** moving down the list, you accept more grammars but pay in table size, build complexity, or error-message quality. GLR can be O(n³) worst case; LL/LR are linear O(n); PEG is linear but memo-hungry.

---

## Top-Down: Recursive Descent

The most direct parser: **one function per non-terminal**. The call stack *is* the parse stack; the grammar reads straight off the code. This is how GCC, Clang and most hand-written parsers work.

```c
// Expr → Term (('+'|'-') Term)*
Node *parse_expr(Parser *p) {
    Node *lhs = parse_term(p);
    while (peek(p) == '+' || peek(p) == '-') {
        Tok op = advance(p);          // consume operator
        Node *rhs = parse_term(p);
        lhs = mk_binary(op, lhs, rhs); // build AST node
    }
    return lhs;
}

// Factor → number | '(' Expr ')'
Node *parse_factor(Parser *p) {
    if (peek(p) == '(') {
        expect(p, '(');
        Node *e = parse_expr(p);
        expect(p, ')');               // error if missing
        return e;
    }
    return mk_num(expect(p, NUMBER));
}
```

Why it's loved: code reads like the grammar; arbitrary code at any point gives superb error messages; easy to add lookahead, backtracking and error recovery. The one trap is direct *left-recursion* (`Expr→Expr+Term`), which makes `parse_expr` call itself forever — we rewrite it into the iterative loop above. When one lookahead token uniquely picks the production, no backtracking is needed — that's LL(1), *predictive* parsing.

---

## FIRST, FOLLOW & the LL(1) Table

To decide which production to take from one lookahead, a predictive parser precomputes two sets. **FIRST(α)** = terminals that can *begin* a string derived from α. **FOLLOW(A)** = terminals that can appear *immediately after* A.

```ebnf
E  → T E'
E' → "+" T E' | ε
T  → F T'
T' → "*" F T' | ε
F  → "(" E ")" | id
```

| X | FIRST | FOLLOW |
|---|---|---|
| E | ( id | $ ) |
| E' | + ε | $ ) |
| T | ( id | + $ ) |
| T' | * ε | + $ ) |
| F | ( id | * + $ ) |

**FIRST rules:** terminal `a` → FIRST = {`a`}; for `A→X₁X₂…` add FIRST(X₁), and if X₁ ⇒* ε also FIRST(X₂)…; if all can vanish, add ε.

**FOLLOW rules:** put `$` in FOLLOW(start); for `A→αBβ` add FIRST(β)\{ε} to FOLLOW(B); if β ⇒* ε (or is empty) add FOLLOW(A) to FOLLOW(B).

**Building the LL(1) table:** for each `A→α`, put it under `M[A,t]` for every `t ∈ FIRST(α)`; if ε ∈ FIRST(α), also for every `t ∈ FOLLOW(A)`. Two entries in one cell means the grammar is *not* LL(1).

---

## Pratt Parsing / Precedence Climbing

For expressions, recursive descent needs a function per precedence level. **Pratt parsing** (Vaughan Pratt, 1973) collapses them into one loop driven by **binding powers** — the technique behind many real parsers (Go, Clang's expressions, rustc, *Crafting Interpreters*).

```python
# binding power table: (left_bp, right_bp)
BP = {'+':(10,11), '-':(10,11),    # left-assoc
      '*':(20,21), '/':(20,21),
      '^':(31,30)}                  # right-assoc

def expr(p, min_bp=0):
    lhs = atom(p)                   # number / (expr) / unary
    while True:
        op = p.peek()
        if op not in BP: break
        lbp, rbp = BP[op]
        if lbp < min_bp: break      # stop: weaker than caller
        p.next()                    # consume operator
        rhs = expr(p, rbp)          # recurse with op's right bp
        lhs = ('binary', op, lhs, rhs)
    return lhs
```

The trick: each operator carries a *left* and *right* binding power. Asymmetry encodes associativity — `(31,30)` for `^` lets a later `^` bind, giving *right*-association. Real parsers like it because one small function handles *all* operators, adding an operator is adding a table row, it's linear time with no grammar stratification, and unary/postfix/ternary forms slot in easily. "Precedence climbing" (Clarke) and "Pratt parsing" are essentially the same algorithm framed differently.

---

## Left-Recursion & Left-Factoring

Top-down parsers can't handle **left-recursion** (infinite loop) or **common prefixes** (can't choose with one token). Two mechanical transforms make a grammar LL-friendly while preserving the language.

**Eliminate left-recursion:**

```ebnf
// before  (left-recursive)
A → A α | β
// after   (right-recursive + ε)
A  → β A'
A' → α A' | ε

// concretely
Expr → Expr "+" Term | Term
   ⇓
Expr  → Term Expr'
Expr' → "+" Term Expr' | ε
```

**Left-factoring:**

```ebnf
// before  (common prefix — can't pick)
S → "if" E "then" S "else" S
  | "if" E "then" S
// after   (factor the shared prefix)
S  → "if" E "then" S S'
S' → "else" S | ε
```

In practice, hand-written recursive descent rarely applies these literally — it just writes the *loop* form, which *is* the eliminated grammar. The transforms matter most when feeding an LL generator.

---

## Bottom-Up: Shift-Reduce Parsing

Bottom-up parsers build the tree from the *leaves up*, tracing a rightmost derivation in reverse. They keep a **stack** and repeatedly choose between two moves: **shift** the next token onto the stack, or **reduce** a stack suffix matching a production's RHS back to its LHS.

- **Shift** — push the next input token onto the stack.
- **Reduce** — the stack top matches a production RHS (a **handle**); pop it, push the LHS non-terminal. This builds one AST node.
- **Handle** — the substring whose reduction is the correct next step in the reverse rightmost derivation. The whole problem is: *find the handle*. The LR automaton answers it.
- **Accept** when the stack is just `S` and input is `$`; **error** when no shift or reduce is legal.

Trace for `id * id`:

| Stack | Input | Action |
|---|---|---|
| `$` | id * id $ | shift |
| `$ id` | * id $ | reduce F→id |
| `$ F` | * id $ | reduce T→F |
| `$ T` | * id $ | shift |
| `$ T *` | id $ | shift |
| `$ T * id` | $ | reduce F→id |
| `$ T * F` | $ | reduce T→T*F |
| `$ T` | $ | reduce E→T |
| `$ E` | $ | **accept** |

---

## LR(0) Items & the Canonical Automaton

How does the parser know when a handle is on the stack? It runs a DFA over *grammar symbols*. Each state is a set of **LR(0) items** — productions with a dot `•` marking how much has been seen. The dot at the end means "handle complete → reduce".

- **closure** — if the dot precedes `A`, add all `A→•…` items.
- **goto(I, X)** — advance the dot over `X`, take closure → next state.

**The canonical collection:** start from `closure({S'→•S})` and apply `goto` until no new item sets appear. That set of states *is* the LR(0) automaton driving the parse stack. The parser stack actually holds *states*, not symbols; the current state's items tell it whether to shift or which rule to reduce.

A fragment: `I0` (`E'→•E`, `E→•E+T`, `E→•T`, …) on `E` goes to `I1` (`E'→E•`, `E→E•+T`); on `T` to `I2` (`E→T•`, a reduce state); `I1` on `+` goes to `I5` (`E→E+•T`, `T→•F`, …).

---

## SLR vs LR(1) vs LALR(1)

LR(0) alone often hits states with both a shift and a reduce item and no way to choose. The three flavours differ in *how much lookahead context* they attach to reduce decisions.

- **SLR(1)** — reduce `A→α` only when the next token is in FOLLOW(A). Cheapest, but FOLLOW is too coarse, so it rejects many real grammars.
- **LR(1)** — each item carries an *exact* lookahead token: `[A→α•β, a]`. Most powerful deterministic LR, but state counts explode.
- **LALR(1)** — build LR(1), then *merge* states with identical cores (same items, different lookaheads). Nearly LR(1) power at LR(0) table size — the sweet spot.

Why LALR(1) won: DeRemer's 1969 construction gives compact tables that handle almost every practical programming-language grammar. yacc (Johnson, 1975) made it the industry default; bison and menhir continue it. The catch: state merging can introduce *new* reduce/reduce conflicts that pure LR(1) wouldn't have — rare, but it's why menhir offers a `--canonical` (full LR(1)) mode.

---

## The ACTION / GOTO Tables

The automaton is compiled into two tables. **ACTION[state, terminal]** → shift *s*, reduce *r*, accept, or error. **GOTO[state, non-terminal]** → the state to enter after a reduction. The driver is a tiny, fixed loop.

```c
// Table-driven LR driver (the heart of yacc output)
for (;;) {
  State s = top(stack);
  Token a = peek();
  Action act = ACTION[s][a];
  switch (act.kind) {
    case SHIFT:                 // sN
      push(act.state); advance(); break;
    case REDUCE:                // rN: A → β
      pop(|β| states);
      State t = top(stack);
      push(GOTO[t][A]);
      emit_ast_node(A);         // semantic action
      break;
    case ACCEPT: return root;
    case ERROR:  recover();
  }
}
```

A fragment of the table for the expression grammar:

| State | id | + | * | $ | E | T |
|---|---|---|---|---|---|---|
| 0 | s5 | | | | 1 | 2 |
| 1 | | s6 | | acc | | |
| 2 | | r2 | s7 | r2 | | |
| 5 | | r6 | r6 | r6 | | |
| 6 | s5 | | | | | 9 |

Each token causes a bounded number of moves → O(n) total. The tables are generated *once* at build time; parsing is just array lookups and a stack.

---

## Conflicts: Shift/Reduce & Reduce/Reduce

A conflict is a table cell with two actions — the grammar isn't in the class the generator targets. Every yacc/bison user meets these; resolving them is a craft.

**Shift/reduce** — the state could shift *or* reduce. The canonical case is the dangling else: after `if E then S`, on seeing `else`, shift (bind inner) or reduce (bind outer)?

```
state: S → if E then S •  [• else …]
on 'else':  shift  → inner if   (usual)
            reduce → outer if
```

Generators default to **shift**, which happens to give the desired "nearest `if`" rule — benign, but warned about.

**Reduce/reduce** — two different productions could both reduce the same handle. Almost always a real grammar bug; never resolve it by ignoring — fix the grammar.

```
A → x •     (reduce A→x)
B → x •     (reduce B→x)   ← which?
```

**Precedence declarations** — bison's `%left '+'`, `%left '*'`, `%right '^'`, `%nonassoc` assign precedence and associativity to tokens, automatically resolving operator shift/reduce conflicts *without* rewriting the grammar.

---

## Parser Generators & Toolkits

You rarely write LR tables by hand. A generator reads a grammar with embedded **semantic actions** and emits the parser. The choice of tool fixes the grammar class, the error story, and the workflow.

| Tool | Algorithm | Language | Notable for |
|---|---|---|---|
| yacc / bison | LALR(1) (bison: GLR opt.) | C/C++ | the classic; powers many real compilers & tools |
| menhir | LALR(1) & full LR(1) | OCaml | excellent conflict explanations, modular grammars |
| ANTLR 4 | ALL(*) adaptive LL | Java/many | readable LL grammars, great tooling & IDE support |
| tree-sitter | GLR (incremental) | C / many | editor parsing: incremental, error-tolerant |
| PEG / packrat | ordered-choice PEG | many | combined lex+parse, linear with memoisation |
| parser combinators | recursive descent (lib) | Haskell, Rust… | parsers as composable values (parsec, nom) |

**PEG & packrat:** a Parsing Expression Grammar replaces ambiguous `|` with *ordered choice* `/` — the first alternative that matches wins, so PEGs are unambiguous by construction. Packrat parsing memoises every (rule, position) result, giving guaranteed linear time at the cost of memory. Downsides: ordered choice can silently mask alternatives, and error reporting is weaker.

**Combinators:** in functional languages, parsers are first-class functions combined with `<|>`, `many`, `seq`. They *are* recursive descent expressed compositionally — great ergonomics, but watch backtracking cost and left-recursion (still forbidden).

---

## Error Recovery & Diagnostics

A compiler that stops at the first error is useless. Good parsers **recover** — report the error, then resynchronise to keep finding more — without an avalanche of bogus follow-on errors.

- **Panic-mode** — on error, discard tokens until a *synchronising* token (`;`, `}`, `end`) appears, then resume. Simple, robust, the workhorse of recursive descent.
- **Phrase-level** — locally repair: insert a missing `;`, delete a stray token, swap two. Lets the parser limp on as if the input were correct.
- **Error productions** — add grammar rules for *common mistakes* (bison's `error` token) so the parser recognises and reports them precisely.

**Why hand-written wins on errors:** full context at the failure point gives specific, actionable messages; custom recovery per construct (e.g. an unclosed brace); "did you mean…", fix-its and ranges are hard to retrofit onto table-driven parsers.

**Production reality:** GCC moved from a bison grammar to a *hand-written* recursive-descent C/C++ parser (2004) chiefly for diagnostics. Clang and Roslyn (C#/VB) are hand-written recursive descent for the same reason — and to support IDEs with incremental, error-tolerant parsing.

---

## AST Design & Construction

The parser's real product is the **AST**. It's built on the fly by **semantic actions** — code attached to productions (or written inline in recursive descent) that constructs a node as each rule completes.

```rust
// A typical AST as a Rust enum (sum type)
enum Expr {
    Num(f64),
    Var(String),
    Unary  { op: UnOp,  rhs: Box<Expr> },
    Binary { op: BinOp, lhs: Box<Expr>, rhs: Box<Expr> },
    Call   { callee: String, args: Vec<Expr> },
}
enum Stmt {
    Let   { name: String, init: Expr },
    If    { cond: Expr, then: Box<Stmt>, els: Option<Box<Stmt>> },
    Block(Vec<Stmt>),
    Expr(Expr),
}
```

```
# bison semantic action ($$ = result, $1.. = children)
expr : expr '+' term  { $$ = mk_bin(ADD, $1, $3); }
     | term           { $$ = $1; }
```

What goes in a node: its kind & children (the structure); **source spans** (line/col) for diagnostics & tooling; slots for later phases (resolved type, symbol, constant value). Keep the AST *abstract* — drop parentheses, semicolons and single-child chains. Some tools (rust-analyzer, tree-sitter) keep a *lossless* CST for refactoring/formatting and derive an AST view from it. This tree is exactly what Part 05 walks to resolve names and check types — the bridge from syntax to meaning.

---

## Top-Down vs Bottom-Up — Choosing

Both produce the same AST; they differ in *how* they find it and in their engineering trade-offs. The industry has largely converged on hand-written recursive descent + Pratt for serious compilers, generators for everything else.

| | Top-Down (LL / recursive descent) | Bottom-Up (LR / LALR) |
|---|---|---|
| Derivation | leftmost, predict from the top | rightmost-reverse, recognise from leaves |
| Grammar power | smaller class; no left-recursion | larger class; left-recursion is natural |
| Readability | code mirrors grammar — very readable | opaque tables; debug via the generator |
| Error messages | excellent, full context | harder; improving (menhir, ANTLR) |
| Who uses it | GCC, Clang, Roslyn, V8, Go, rustc | Ruby (parse.y), PHP, Bash, many DSLs |

**Pick recursive descent** when diagnostics & IDE support matter, the language is stable, you want full control, and the grammar fits LL with a Pratt expression sub-parser (almost every modern production compiler). **Pick a generator** when the grammar is large/evolving, you want the spec to *be* the parser, left-recursion abounds, or you're prototyping a DSL (menhir & ANTLR give the best modern experience).

---

## Summary & Further Reading

**Key takeaways:**

- The parser turns tokens into a tree, driven by a context-free grammar.
- Parse tree = concrete; AST = abstract; precedence lives in the tree's *shape*.
- Ambiguity (expr grammar, dangling-else) is fixed by stratifying or by precedence rules.
- Classes nest: LL(1) ⊊ SLR ⊊ LALR(1) ⊊ LR(1) ⊊ GLR; PEG sits apart.
- Top-down: recursive descent + FIRST/FOLLOW; Pratt parsing for expressions.
- Bottom-up: shift-reduce, LR(0) items, ACTION/GOTO tables, LALR(1) tooling.
- Conflicts = grammar out of class; precedence declarations resolve operator ones.
- Hand-written recursive descent wins on diagnostics — GCC, Clang, Roslyn.

**Hands-on exercise:** write a recursive-descent + Pratt parser for arithmetic with `+ - * / ^ ()`; add unary minus and verify `^` right-associates; compute FIRST/FOLLOW for the grammar and build its LL(1) table.

**Further reading:**

- Aho, Lam, Sethi & Ullman — *Compilers: Principles, Techniques & Tools* (the Dragon Book), ch. 4
- Cooper & Torczon — *Engineering a Compiler*, ch. 3
- Grune & Jacobs — *Parsing Techniques* (the encyclopaedia)
- Vaughan Pratt — *Top Down Operator Precedence* (1973)
- Bryan Ford — *Parsing Expression Grammars* (2004)
- Parr & Fisher — *ALL(*) Parsing* (the ANTLR 4 paper, 2014)
- Nystrom — *Crafting Interpreters* (Pratt parsing chapter)

**Mental model to keep:** two ways to find the same tree — predict it from the top (LL) or recognise it from the leaves (LR). The grammar decides which is possible; engineering decides which you pick.

`Part 04 ✓` -> `Part 05: Semantic Analysis & Types`
