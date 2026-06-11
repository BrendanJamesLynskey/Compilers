# ⚙️ Compilers — From Source Text to Running Code

A **ten-part, deeply illustrated interactive series** on how a compiler turns the text you type into instructions a machine runs — the history, the classic pipeline, and every stage in between, ending by building a small compiler end to end. Each part is a self-contained [Reveal.js](https://revealjs.com) presentation with hand-drawn inline-SVG diagrams and copy-and-run code, plus a plain-Markdown companion.

## ▶ [Open the Series Landing Page](https://brendanjameslynskey.github.io/Compilers/)

```
source → tokens → AST → typed AST → IR / SSA → optimised IR → asm → machine code
```

---

## Presentations

| # | Presentation | Stage | Description |
|---|--------------|-------|-------------|
| 01 | [A History of Compilers](https://brendanjameslynskey.github.io/Compilers/01_History_of_Compilers/) ([md](01_History_of_Compilers/presentation.md)) | History | First assemblers (Booth, Wheeler), Hopper's A-0 & the word "compiler", FORTRAN's optimiser, ALGOL & BNF, the Chomsky hierarchy, LR/LALR parsing, lex & yacc, the Dragon Book, GCC, SSA, the JIT lineage (Self → HotSpot), LLVM, and the modern infrastructure era (Wasm, MLIR) |
| 02 | [The Compiler Pipeline](https://brendanjameslynskey.github.io/Compilers/02_The_Compiler_Pipeline/) ([md](02_The_Compiler_Pipeline/presentation.md)) | Overview | Compiler vs interpreter vs JIT, analysis vs synthesis, the phases end to end, a worked trace of one statement through every stage, front/middle/back ends and the N×M "narrow waist", the full toolchain, bootstrapping & T-diagrams, cross-compilation |
| 03 | [Lexical Analysis](https://brendanjameslynskey.github.io/Compilers/03_Lexical_Analysis/) ([md](03_Lexical_Analysis/presentation.md)) | Front end | Regular expressions & regular languages, NFA vs DFA, Thompson's construction, the subset construction & DFA minimisation, maximal munch & lookahead, hand-written vs generated scanners (lex/flex/re2c), literals & Unicode, the Python INDENT/DEDENT and C lexer-hack cases |
| 04 | [Syntax Analysis (Parsing)](https://brendanjameslynskey.github.io/Compilers/04_Syntax_Analysis/) ([md](04_Syntax_Analysis/presentation.md)) | Front end | Context-free grammars, derivations, parse trees vs ASTs, ambiguity & precedence; top-down (recursive descent, FIRST/FOLLOW, LL(1), Pratt parsing); bottom-up (shift-reduce, LR(0) items, SLR/LALR(1)); conflicts; generators (yacc/bison, ANTLR, tree-sitter, PEG); error recovery |
| 05 | [Semantic Analysis & Type Systems](https://brendanjameslynskey.github.io/Compilers/05_Semantic_Analysis_and_Types/) ([md](05_Semantic_Analysis_and_Types/presentation.md)) | Front end | Symbol tables & scope, name resolution & shadowing, attribute grammars & syntax-directed translation, type-system axes, typing judgements (Γ ⊢ e : τ), Hindley–Milner inference & unification, subtyping & variance, generics & type classes, diagnostics |
| 06 | [Intermediate Representations](https://brendanjameslynskey.github.io/Compilers/06_Intermediate_Representations/) ([md](06_Intermediate_Representations/presentation.md)) | Middle end | Why an IR, the HIR/MIR/LIR spectrum, three-address code, stack vs register bytecode (JVM/CPython/Wasm vs Lua/Dalvik), control-flow graphs, dominators, SSA & φ-functions (construction & destruction), LLVM IR in depth, CPS, sea-of-nodes, MLIR |
| 07 | [Optimization](https://brendanjameslynskey.github.io/Compilers/07_Optimization/) ([md](07_Optimization/presentation.md)) | Middle end | The "as-if" rule, scope levels up to LTO & PGO, the data-flow framework (lattices, transfer functions, fixed points), the four classic analyses, classic transforms (CSE, DCE, inlining, strength reduction), loop optimisations & vectorisation, SSA-based passes (SCCP, GVN), alias analysis, phase ordering |
| 08 | [Code Generation](https://brendanjameslynskey.github.io/Compilers/08_Code_Generation/) ([md](08_Code_Generation/presentation.md)) | Back end | The three interacting subproblems; instruction selection (tree tiling, BURS, SelectionDAG/GlobalISel); register allocation (interference graphs, graph colouring, Chaitin–Briggs, linear scan, coalescing); instruction scheduling; the System V AMD64 ABI, stack frames & prologue/epilogue; a worked x86-64 / AArch64 example |
| 09 | [Runtimes, JITs & Modern Backends](https://brendanjameslynskey.github.io/Compilers/09_Runtime_JIT_and_Backends/) ([md](09_Runtime_JIT_and_Backends/presentation.md)) | Runtime | Interpreters & dispatch (threaded code, computed goto), JITs (baseline vs optimising, tracing, tiered execution in V8/HotSpot), inline caches, deoptimisation & OSR, garbage collection (generational, concurrent), exception handling, dynamic linking (PLT/GOT), and modern backends: LLVM, Cranelift, GraalVM, MLIR, WebAssembly |
| 10 | [Building a Mini-Compiler](https://brendanjameslynskey.github.io/Compilers/10_Building_a_Mini_Compiler/) ([md](10_Building_a_Mini_Compiler/presentation.md)) | Capstone | One tiny language ("MiniLang") threaded through the whole pipeline in real, runnable code: grammar → hand-written lexer → recursive-descent + Pratt parser → type checker → three-address IR → constant folding & DCE → stack-VM bytecode → an interpreter that runs it — plus testing, error messages, and where to go next |

---

## Learning Arc

```
01–02  Frame it     →  history + the pipeline
03–05  Front end    →  lexing · parsing · semantics      (text → typed tree)
06–07  Middle end   →  IR / SSA · optimisation           (target-independent)
08     Back end      →  code generation                   (→ assembly)
09     Run it        →  runtimes · JITs · GC · Wasm
10     Build it      →  a working mini-compiler, end to end
```

Read in order for a full course, or jump to a topic. Every deck ends with a Summary, Further Reading, and a pointer to the next part.

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to the URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono. The landing page uses Space Grotesk + Inter + JetBrains Mono.

Each deck is a single self-contained `index.html` — no build step, no npm, no dependencies to install. Every diagram is hand-authored inline SVG.

## References

[Aho, Lam, Sethi & Ullman — *Compilers: Principles, Techniques & Tools* (the "Dragon Book")](https://www.pearson.com/en-us/subject-catalog/p/compilers-principles-techniques-and-tools/P200000003472) · [Nystrom — *Crafting Interpreters*](https://craftinginterpreters.com/) · [Appel — *Modern Compiler Implementation*](https://www.cs.princeton.edu/~appel/modern/) · Cooper & Torczon — *Engineering a Compiler* · [LLVM docs](https://llvm.org/docs/) · [Kaleidoscope tutorial](https://llvm.org/docs/tutorial/) · [WebAssembly](https://webassembly.org/)

## See also

- Series hub: [Software](https://github.com/BrendanJamesLynskey/Software) — presentations, playgrounds and reference projects.
- Companion decks in the same style: [Computational Complexity](https://github.com/BrendanJamesLynskey/Computational_Complexity), [transformer-explainer](https://github.com/BrendanJamesLynskey/transformer-explainer).

## License

Educational use. Code examples provided as-is.
