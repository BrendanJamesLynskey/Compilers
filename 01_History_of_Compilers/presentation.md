# Compilers — Part 01: A History of Compilers

**Part 01 — From hand-punched machine code to reusable compiler infrastructure**

> Seventy years of turning human notation into machine instructions -- the people, the breakthroughs and the ideas (analysis/synthesis, SSA, a common IR) that still shape every compiler you use today.

`1950s Autocodes` -> `FORTRAN / ALGOL` -> `Theory & SSA` -> `LLVM / MLIR`

Hopper - Backus - Naur - Knuth - Aho & Ullman - Lattner

---

## Table of Contents

1. [Topics](#topics)
2. [Seventy Years at a Glance](#seventy-years-at-a-glance)
3. [Prehistory -- Hand-Coding the Machine](#prehistory--hand-coding-the-machine)
4. [Grace Hopper & the A-0 System (1952)](#grace-hopper--the-a-0-system-1952)
5. [Early Autocodes (1952-1957)](#early-autocodes-19521957)
6. [FORTRAN (1957) -- The First Great Optimiser](#fortran-1957--the-first-great-optimiser)
7. [Theory Foundations -- Chomsky & BNF](#theory-foundations--chomsky--bnf)
8. [ALGOL 58 / 60 -- A Landmark Report](#algol-58--60--a-landmark-report)
9. [The Parsing-Theory Era (1960s-70s)](#the-parsing-theory-era-1960s70s)
10. [The Unix Toolchain -- C, lex & yacc](#the-unix-toolchain--c-lex--yacc)
11. [The Dragon Book Lineage](#the-dragon-book-lineage)
12. [The Portability Dream -- UNCOL, P-code, Bootstrapping](#the-portability-dream--uncol-p-code-bootstrapping)
13. [The NxM Problem -- Why a Common IR Wins](#the-nxm-problem--why-a-common-ir-wins)
14. [GCC & the Free-Software Toolchain (1987)](#gcc--the-free-software-toolchain-1987)
15. [SSA Form (1991) -- The Optimisation Turning Point](#ssa-form-1991--the-optimisation-turning-point)
16. [The JIT Lineage -- Smalltalk, Self, HotSpot](#the-jit-lineage--smalltalk-self-hotspot)
17. [LLVM (2003) & Clang -- Compiler as a Library](#llvm-2003--clang--compiler-as-a-library)
18. [A Language & Compiler Family Tree](#a-language--compiler-family-tree)
19. [The Modern Era -- Infrastructure as a Platform](#the-modern-era--infrastructure-as-a-platform)
20. [Key People & Contributions](#key-people--contributions)
21. [The Enduring Split -- Analysis & Synthesis](#the-enduring-split--analysis--synthesis)
22. [Recurring Themes & Two Famous Laws](#recurring-themes--two-famous-laws)
23. [Summary & Further Reading](#summary--further-reading)

---

## Topics

A roughly chronological tour. We trace how compilation grew from a clever bookkeeping trick into a rigorous engineering discipline and, finally, into shared **infrastructure** -- and we name the people who made each leap.

### Prehistory & First Compilers

- Machine code, the first assemblers (Booth, Wheeler)
- Grace Hopper's A-0 -- the word "compiler"
- Glennie & Brooker autocodes
- FORTRAN (1957) -- the first great optimiser

### Theory Takes Hold

- Chomsky hierarchy, Backus-Naur Form
- ALGOL 58 / 60 and its report
- LR, LALR, Earley, recursive descent
- The Dragon Book lineage

### Tools, Portability & Freedom

- C, PCC, lex & yacc -- the Unix toolchain
- UNCOL, P-code & bootstrapping
- GCC and the free-software toolchain
- SSA form -- the optimisation turning point

### The Modern Era

- JIT: Smalltalk, Self, Java HotSpot
- LLVM & Clang -- the reusable library
- WebAssembly, Rust, Swift, MLIR
- Enduring themes & Proebsting's Law

---

## Seventy Years at a Glance

Each milestone below is unpacked later in the deck. Note the rhythm: a burst of **theory** in the late 1950s and 60s, a wave of **tooling** in the 70s-80s, and a shift to **reusable infrastructure** from the 2000s on.

```
1950 ........ 1960 ........ 1970 ........ 1980 ........ 1990 ........ 2000 ........ 2010 ...... 2020
  A-0 '52      ALGOL 60      yacc '75      Dragon '86    SSA '91       LLVM '03     Rust/Swift  Wasm '17
  FORTRAN '57  COBOL '59     PCC '75       GCC '87       HotSpot '99   Clang '07    Cranelift   MLIR '19
               LR '65        lex '75
```

- *amber* = compilers & languages
- *green* = theory / standards
- *purple* = tools / infrastructure
- *blue* = optimisation / runtime

---

## Prehistory -- Hand-Coding the Machine

The first stored-program computers (EDSAC 1949, Manchester Baby 1948, EDVAC) were programmed in raw **machine code**: numeric opcodes and absolute addresses, entered in binary, octal or via teleprinter codes. Every jump target had to be computed by hand, and inserting an instruction renumbered everything after it.

### The Pain

- Absolute addresses -- no symbolic labels
- Reusing code meant copying & re-addressing it
- Programmers wrote in opcode numbers, not mnemonics
- A single inserted line could break every branch

### The First Assemblers

- **Kathleen Booth** (~1947) wrote some of the earliest assembly notation for the ARC2 / SEC at Birkbeck College, London.
- **Nathaniel Rochester** built an assembler for the IBM 701 (1954).
- An *assembler* maps mnemonics 1:1 to machine instructions -- it is not yet a compiler.

### EDSAC & the Initial Orders

Cambridge's EDSAC (1949) booted via **David Wheeler**'s *Initial Orders* -- a tiny bootstrap loader (35 instructions) that read symbolic, relocatable order codes from paper tape and placed them in memory, converting decimal addresses as it went. It is arguably the first practical assembler / loader.

### The "Wheeler Jump"

With no hardware return-address mechanism, Wheeler devised the idiom for calling and returning from a subroutine: the caller plants its return address into the subroutine's final jump. This **closed subroutine** made the first reusable code libraries possible -- the ancestor of every function call.

Wheeler, Wilkes & Gill's 1951 book *The Preparation of Programs for an Electronic Digital Computer* was the first textbook on programming -- and on subroutine libraries.

---

## Grace Hopper & the A-0 System (1952)

On the UNIVAC I, **Grace Hopper** built **A-0** (1952) and coined the word **"compiler"**. Crucially, its *original* meaning differs from ours: A-0 read a sequence of subroutine call numbers with arguments and *compiled* -- in the librarian's sense of "gathering together" -- the pre-written routines from tape, linking and relocating them into one runnable program.

```
call-number list  ->  A-0 compiler  ->  runnable UNIVAC program
(+ arguments)         (linker /
                       relocator)
                          ^
                          | pulls & relocates
                    subroutine tape (library of routines)
```

### Why It Mattered

- First system to translate a higher-level description into machine code automatically
- Successors A-1, A-2, then **ARITH-MATIC** / **MATH-MATIC**
- Proved machines could write machine code -- against fierce scepticism at the time

### FLOW-MATIC -> COBOL

Hopper's **FLOW-MATIC** (1955-59) used English-like statements for business data processing. It became the primary template for **COBOL** (1959), defined by the CODASYL committee -- making Hopper a guiding force behind the most widely deployed language of the century.

---

## Early Autocodes (1952-1957)

In Britain, the equivalent term was **autocode**. These were the first systems to compile genuine algebraic expressions into machine code -- pre-dating, and partly inspiring, the more famous American efforts.

### Glennie's Autocode (1952)

- **Alick Glennie**, at Fort Halstead, for the Manchester Mark 1.
- Widely regarded as the *first compiled programming language*.
- Translated a readable algebraic notation into Mark 1 machine code.
- Little used -- the notation stayed close to the hardware and the win over hand-coding was modest.

### Brooker's Mark 1 Autocode (1954)

- **Tony Brooker** at Manchester replaced Glennie's system with a far more usable autocode.
- Machine-independent in spirit; it hid drum/store details from the programmer.
- The later *Mercury Autocode* (1957) was influential across UK universities.
- Brooker's later *Compiler Compiler* (1960, with Morris) was an early metacompiler.

```
# Flavour of an early autocode line (Brooker-style):
   n1 = n2 + n3 x n4        # arithmetic in near-mathematical form
   j1, 3 >= n1              # conditional jump to label 3 if n1 <= 0
# The autocode expands each line into a sequence of machine orders,
# allocating working store and handling the accumulator for you.
```

The lesson the autocodes taught: programmers would adopt a higher notation *only* if the translator's output was efficient enough to be worth it. That bar is exactly what FORTRAN set out to clear.

---

## FORTRAN (1957) -- The First Great Optimiser

**John Backus**'s IBM team shipped FORTRAN for the IBM 704 in **1957**. It was not the first compiler, but it was the first *widely successful* one -- and its central challenge was economic: it had to generate code nearly as fast as expert hand-written assembly, or no one would trust it.

```fortran
C     FORTRAN II — sum 1..N (fixed-form, col 7+)
      SUM = 0.0
      DO 10 I = 1, N
         SUM = SUM + A(I) * B(I)
   10 CONTINUE
      WRITE (6, 20) SUM
   20 FORMAT (1H , F12.4)
```

### Why It Had to Beat Assembly

Machine time was vastly more expensive than programmer time in 1957. A compiler that produced slow code was useless. Backus reportedly spent the bulk of the effort not on parsing but on the **optimiser** -- especially efficient use of the 704's three index registers.

### Optimisations It Pioneered

- **Register allocation** for index registers
- **Common-subexpression** elimination
- Array-subscript / loop address computation
- Basic control-flow analysis & section sorting

### Legacy

- Established that compilers could rival humans
- The *analysis / synthesis* split begins here
- Still in use (Fortran 2018, LLVM Flang)
- Backus later co-created BNF and won the 1977 Turing Award

---

## Theory Foundations -- Chomsky & BNF

FORTRAN's parser was largely ad hoc. The late 1950s gave compiler writers a **formal theory of grammar** -- and a notation to write languages down precisely.

### The Chomsky Hierarchy (1956)

Noam Chomsky classified grammars into four nested types. Two of them became the bedrock of compiling:

- **Type-3 / Regular** -> recognised by finite automata -> *lexing*
- **Type-2 / Context-free** -> recognised by pushdown automata -> *parsing*
- Type-1 context-sensitive & Type-0 unrestricted sit above, beyond practical parsing.

(The hierarchy nests: Type-3 inside Type-2 inside Type-1 inside Type-0.)

### Backus-Naur Form

To define ALGOL, **John Backus** (1959) and **Peter Naur** (1960) introduced a metasyntax for context-free grammars. BNF made a language's syntax a precise, machine-readable artefact rather than English prose.

```ebnf
<expr>   ::= <expr> "+" <term>
           | <term>
<term>   ::= <term> "*" <factor>
           | <factor>
<factor> ::= "(" <expr> ")"
           | <number>
```

This grammar -- expressions over `+` and `*` -- reappears in every parsing chapter to come, encoding precedence and associativity structurally.

---

## ALGOL 58 / 60 -- A Landmark Report

ALGOL 60, defined by an international committee and written up in the famous **Revised Report** (Naur, ed., 1963), was "a language so far ahead of its time that it was an improvement on its successors" (Hoare). It introduced ideas now universal.

### Innovations

- **Block structure** & lexical scope (`begin...end`)
- **Nested functions** & recursion as standard
- Call-by-name and call-by-value parameters
- A syntax defined formally in **BNF**
- Clean separation of the language from any machine

### Lasting Influence

ALGOL is the common ancestor of the entire *imperative / block-structured* family: CPL -> BCPL -> B -> C, plus Pascal, Simula (and thus object orientation), Ada and beyond. "ALGOL-like" is still shorthand for the mainstream syntax tradition.

```
begin
   integer procedure fact(n);
      value n; integer n;
      fact := if n < 2 then 1
              else n * fact(n - 1);

   integer k;
   for k := 1 step 1 until 5 do
      outinteger(1, fact(k))
end
```

### The Report as Artefact

For the first time, a language's *reference* was a rigorous document -- syntax in BNF, semantics in careful prose. It set the standard for how languages are specified, and made independent, conforming compilers possible.

---

## The Parsing-Theory Era (1960s-70s)

With context-free grammars formalised, the question became: how do you build a parser *mechanically and efficiently* for any reasonable grammar? A decade of beautiful algorithms answered it.

| Technique | Who / When | Idea |
|-----------|-----------|------|
| Recursive descent | folklore, late 50s/60s | One function per non-terminal; hand-written, top-down (LL) |
| **LR(k)** | Knuth, 1965 | Bottom-up, shift-reduce; the largest class of deterministic CFGs |
| **LALR(1)** | DeRemer, 1969 | Merged LR states -- small tables, practical; powers yacc |
| **Earley** | Earley, 1970 | Handles *any* CFG, even ambiguous; O(n^3), with O(n^2)/O(n) special cases |
| SLR / GLR | DeRemer '71 / Tomita '86 | Simpler tables / generalised parallel parsing |

### Knuth's LR (1965)

*"On the Translation of Languages from Left to Right"* showed a single DFA over *items* can deterministically parse a huge class of grammars in linear time -- the theoretical heart of bottom-up parsing.

The deck shows a parse tree for `a + b * c`, with precedence baked into the grammar so `*` binds tighter than `+`:

```
        E
      / | \
     E  +  T
     |    /|\
     T   T * F
     |   |   |
     a   b   c
```

---

## The Unix Toolchain -- C, lex & yacc

At Bell Labs in the early 1970s, theory met practice. **Dennis Ritchie** created **C** (~1972) to write Unix itself; alongside it grew the generator-based toolchain that made building compilers routine.

### C & PCC

- **C** (Ritchie, ~1972) -- a portable systems language; Unix v4 was rewritten in it.
- **PCC**, the Portable C Compiler (**Steve Johnson**, ~1975), retargeted easily by splitting a machine-independent front-end from a table-driven back-end.
- PCC made C available on dozens of architectures -- a key reason Unix and C spread.

### lex (1975) & yacc (1975)

- **lex** (Mike Lesk & Eric Schmidt) -- generates a lexer from regular expressions (Type-3 grammar -> DFA).
- **yacc** -- "Yet Another Compiler-Compiler" (Steve Johnson) -- generates an **LALR(1)** parser from a BNF-like grammar.
- Their GNU successors *flex* & *bison* are still everywhere.

```
tokens.l  --lex-->  lex.yy.c  \
                               +--> C source --> compiled compiler
grammar.y --yacc--> y.tab.c   /
```

Generators turn declarative specs into C source -- the front-end stops being hand-written.

---

## The Dragon Book Lineage

No book shaped how compilers are taught more than the "Dragon Book" series -- named for its cover, a knight battling a dragon labelled "complexity of compiler design". It codified the standard **phase pipeline** a generation learned by heart.

| Edition | Authors | Nickname |
|---------|---------|----------|
| 1977 | Aho & Ullman, *Principles of Compiler Design* | green dragon |
| **1986** | Aho, Sethi, Ullman | **red dragon** |
| **2006** | Aho, Lam, Sethi, Ullman (2nd ed.) | **purple dragon** |

### What It Standardised

- The canonical front-end / back-end phase breakdown
- Regular expressions -> NFA -> DFA for lexing
- LL, LR, LALR parsing tables, worked end to end
- Syntax-directed translation, three-address code, DAGs
- Data-flow analysis & classic optimisations

The canonical pipeline (analysis above the line, synthesis below):

```
Lexical analysis  -.
Syntax analysis    |  analysis
Semantic analysis -'
IR generation     -.
Optimisation       |  synthesis
Code generation   -'
   |
target machine code
```

---

## The Portability Dream -- UNCOL, P-code, Bootstrapping

With *M* languages and *N* machines, writing every front-end against every back-end means **M x N** compilers. The recurring dream: insert a common **intermediate language** so you write only **M + N** halves. It took 50 years to fully realise.

### UNCOL (1958)

The *UNiversal Computer-Oriented Language* -- proposed by Melvin Conway and a SHARE committee as a single intermediate target all compilers would emit and all machines would implement. Hugely influential as an idea; never realised in its time (the language-design problem was too hard for the era).

### Pascal P-code & the P-System

**Niklaus Wirth**'s Pascal-P compiler (Zurich, mid-1970s) emitted **P-code** for a hypothetical stack machine. To port Pascal you wrote a small P-code interpreter -- so Pascal spread to micros via the UCSD *p-System*. The same idea reappears as the JVM bytecode, the CLR and WebAssembly.

### Bootstrapping & Self-Hosting

A compiler for language L written *in* L. You bootstrap by first compiling a subset with another tool, then compiling the full compiler with itself. C, Pascal, Go, Rust and Swift are all self-hosting today.

```
stage 0 (subset, other lang) --> stage 1 (compiler in L) --> stage 2 (self-compiled)
                                                                  |
                            stage1 output == stage2 output  <-----'  (fixpoint reached)
```

---

## The NxM Problem -- Why a Common IR Wins

The single most important structural idea in compiler engineering, stated plainly. A shared intermediate representation collapses a quadratic problem into a linear one -- the insight UNCOL named and LLVM finally delivered.

**Without a common IR (M x N):** every language wires directly to every target.

```
C, Rust, Swift   x   x86, ARM, RISC-V   =   9 back-ends to write
```

**With a common IR (M + N):** every front-end lowers to one IR; every back-end reads it.

```
C, Rust, Swift  -->  [ common IR ]  -->  x86, ARM, RISC-V
3 front-ends    +    shared optimiser  +   3 back-ends   =   6, optimiser shared once
```

The win compounds: the optimiser, written once against the IR, benefits every language and every target.

---

## GCC & the Free-Software Toolchain (1987)

**Richard Stallman** released the **GNU C Compiler** in 1987 as a cornerstone of the GNU project -- a production-quality, retargetable, *free* compiler. More than any other single program, GCC made open-source operating systems (and later Linux) possible.

### Architecture -- RTL

- Front-ends lower to **RTL** (Register Transfer Language), a Lisp-like low-level IR.
- Machine descriptions (`.md` files) drive instruction selection -- one back-end per target.
- It grew to compile C, C++, Objective-C, Fortran, Ada, Go and more (now the "GNU Compiler Collection").
- Later added the higher-level **GIMPLE** / Tree-SSA IR for modern optimisation.

### The EGCS Fork & Remerge

- Mid-1990s: GCC development stalled under tight release control.
- 1997: the **EGCS** fork formed to move faster and merge experimental work.
- 1999: EGCS was blessed as the official GCC -- the fork *won*, then remerged.
- A landmark in open-source governance, not just compilers.

### Why It Mattered

A single high-quality compiler everyone could read, modify and target turned the compiler from a vendor product into shared infrastructure -- a theme LLVM would take even further.

---

## SSA Form (1991) -- The Optimisation Turning Point

**Static Single Assignment** form -- Cytron, Ferrante, Rosen, Wegman & Zadeck (1991) -- gave every variable exactly *one* definition. This makes use-def chains explicit and turns many optimisations into near-trivial graph walks. Almost every serious optimiser since is built on it.

```
; before SSA — x reassigned
x = 1
if cond:
    x = 2
y = x + 3
```

```
; after SSA — each def is unique
x1 = 1
if cond:
    x2 = 2
x3 = phi(x1, x2)   ; pick by predecessor
y1 = x3 + 3
```

### The phi-function

At a control-flow merge, **phi** selects the value of the variable depending on which edge was taken. phi-nodes are placed at the *dominance frontier* -- the elegant insight that made SSA construction efficient.

---

## The JIT Lineage -- Smalltalk, Self, HotSpot

A parallel story: instead of compiling ahead of time, **compile at runtime**, using information only available then. Dynamic languages drove the breakthroughs that today power JavaScript and Java alike.

### Smalltalk-80 (1980s)

- Deutsch & Schiffman's VM compiled bytecode to native code *on the fly*.
- Coined the practical **JIT** approach: translate lazily, cache the result.
- Showed dynamic dispatch need not be slow.

### Self (late 1980s-90s)

- **Ungar & Holzle** at Stanford/Sun.
- **Maps** (hidden classes) for object layout.
- **Inline caches** & polymorphic inline caches for fast dispatch.
- **Dynamic deoptimisation** -- fall back to the interpreter when assumptions break.

### Java HotSpot (1999)

- Sun's VM, directly descended from Self (the Animorphic team).
- **Adaptive optimisation**: interpret first, profile, then JIT-compile hot methods.
- Tiered C1 / C2 compilers; speculative inlining + deopt.
- The template for V8, SpiderMonkey, the CLR & more.

### The Big Idea: Profile-Guided, Speculative Compilation

Self's hidden classes are exactly V8's "maps"; its inline caches are why modern JavaScript is fast. Speculate that types are stable, compile aggressively, and *deoptimise* to a safe interpreter if a guard fails. Every fast dynamic-language runtime today is a Self descendant.

---

## LLVM (2003) & Clang -- Compiler as a Library

**Chris Lattner** began LLVM as a research project at the University of Illinois (UIUC, ~2003), advised by Vikram Adve. Its radical move was treating the compiler as a set of **reusable libraries** around a well-specified, typed, SSA-based **IR** -- the long-promised common IR, finally done right.

```llvm
; LLVM IR — typed, SSA, target-independent
define i32 @square(i32 %x) {
entry:
  %r = mul nsw i32 %x, %x
  ret i32 %r
}
```

### The Philosophy

- A stable IR usable at compile-, link-, install- and run-time.
- Front-ends (Clang, Rust, Swift...) share one optimiser & many back-ends.
- This is the **M+N** solution made real -- the N x M problem solved.

### Clang

A from-scratch C/C++/Objective-C front-end (Lattner et al., ~2007) with fast compilation and famously good diagnostics. Apple adopted LLVM/Clang as its system toolchain, displacing GCC.

### Why It Won

- Permissive licence -- usable in commercial & research tools.
- Library design -- reuse the optimiser anywhere.
- Became the substrate for Rust, Swift, Julia, Halide, CUDA...

Lattner & Adve received the 2012 ACM Software System Award for LLVM.

---

## A Language & Compiler Family Tree

How the threads connect. The **ALGOL** line gives us C and the systems languages; the dynamic line runs through Smalltalk and Self; modern languages converge on shared back-ends (**LLVM**).

```
FORTRAN '57 ----------------------------\
                                         \  (optimisation ideas: SSA, analysis)
ALGOL 60 --> CPL/BCPL --> C '72 --> C++ '85 \--> LLVM '03 (common IR) --> Swift '14
        \--> Pascal '70                  Java '95 |                       Rust '15
        \--> Simula '67 ---------------------------/                      Julia
                                                                          Clang C/C++
Lisp '58
Smalltalk '80 --> Self '87 --> JS / V8 --------> LLVM
```

The dynamic line (Smalltalk -> Self -> V8) and the static line (ALGOL -> C -> LLVM clients) both ultimately feed shared infrastructure.

---

## The Modern Era -- Infrastructure as a Platform

From the 2010s on, compilers stopped being monolithic programs and became **platforms** -- reusable engines you build languages, accelerators and tools *on top of*.

### New Languages, LLVM Back-Ends

- **Rust** (1.0, 2015) -- memory safety without GC; borrow checker in the front-end, LLVM behind.
- **Swift** (2014) -- Lattner again; adds SIL, a high-level SSA IR, above LLVM.
- Julia, Zig, Clang all share the substrate.

### WebAssembly (2017)

- Wasm 1.0 became a W3C standard in 2017.
- A portable, sandboxed compile target -- UNCOL's dream for the web.
- Now far beyond the browser: edge, plugins, serverless.

### New Engines

- **Cranelift** -- a fast, secure code generator (Wasmtime, Rust).
- **GraalVM / Truffle** -- build a fast language from an AST interpreter via partial evaluation.
- **MLIR** (2019) -- multi-level IR with reusable *dialects*; born of ML compilers, now general.

### MLIR -- the Next Generalisation

Where LLVM IR is one level, **MLIR** (Lattner et al., Google, 2019) lets you define many coexisting IR *dialects* -- from tensor ops down to LLVM IR -- and reuse passes across them. It powers ML compilers (TensorFlow, IREE) and increasingly general toolchains, extending the "reusable infrastructure" idea another level up.

---

## Key People & Contributions

A compressed who's-who of the field. Several won the **Turing Award** (the discipline's highest honour) for the work below.

| Person | Era | Contribution | Honour |
|--------|-----|--------------|--------|
| David Wheeler | 1949 | EDSAC Initial Orders; the closed subroutine ("Wheeler jump") | FRS |
| **Grace Hopper** | 1952 | A-0 system; coined "compiler"; FLOW-MATIC -> COBOL | Nat. Medal of Tech. |
| Alick Glennie | 1952 | Autocode -- arguably the first compiled language | -- |
| Tony Brooker | 1954 | Mark 1 Autocode; the Compiler Compiler (metacompiler) | -- |
| **John Backus** | 1957 | FORTRAN; co-creator of BNF | Turing 1977 |
| Noam Chomsky | 1956 | The grammar hierarchy underlying lexing & parsing | -- |
| Peter Naur | 1960 | BNF; editor of the ALGOL 60 report | Turing 2005 |
| **Donald Knuth** | 1965 | LR parsing; attribute grammars; *TAOCP* | Turing 1974 |
| Frank DeRemer | 1969 | LALR & SLR parsing -- made LR practical (yacc) | -- |
| Dennis Ritchie | 1972 | C; co-creator of Unix | Turing 1983 |
| Steve Johnson | 1975 | yacc; the Portable C Compiler (PCC) | -- |
| **Aho, (Lam,) Sethi, Ullman** | 1977-2006 | The Dragon Book; parsing & algorithms | Aho & Ullman, Turing 2020 |
| Niklaus Wirth | 1970s | Pascal, P-code, Modula, Oberon | Turing 1984 |
| Cytron, Ferrante, Rosen, Wegman, Zadeck | 1991 | Efficient SSA construction via dominance frontiers | -- |
| Ungar & Holzle | 1987-94 | Self: maps, inline caches, dynamic deoptimisation | -- |
| **Chris Lattner** | 2003 | LLVM, Clang, Swift, MLIR | ACM SW Award 2012 |

---

## The Enduring Split -- Analysis & Synthesis

One structural idea has survived intact since FORTRAN: a compiler is a **front-end** that *analyses* the source into a meaning-bearing IR, and a **back-end** that *synthesises* target code from it. Every later breakthrough -- BNF, the Dragon pipeline, GCC's RTL, LLVM IR, MLIR -- is a refinement of where to draw that line and what the IR should be.

```
Front-end (Analysis)           [ IR ]            Back-end (Synthesis)
lex -> parse -> semantic/types  --the-->  opt passes -> instr select -> reg alloc
                                contract
```

The IR is "the contract" between the halves. Its history is the history of the field:

```
UNCOL -> P-code -> RTL/GIMPLE -> LLVM IR -> MLIR dialects
```

---

## Recurring Themes & Two Famous Laws

History rhymes. A handful of debates and "laws" resurface in every compiler generation.

### Proebsting's Law (1998)

Todd Proebsting's wry counter to Moore's Law: *compiler optimisation advances double program performance roughly every 18 **years*** -- not 18 months. The provocative claim: optimisation research delivers far less than hardware. It reframed where compiler value really lies (safety, productivity, new languages, portability) rather than raw speed.

### The "Sufficiently Smart Compiler"

A recurring myth / jargon-file joke: performance excuses that begin "a sufficiently smart compiler could...". The reality -- many high-level abstractions are *not* automatically optimised away, which is precisely why low-level languages and explicit control persist.

### Themes That Keep Returning

- **Portability via a common IR** -- UNCOL -> P-code -> JVM -> LLVM -> Wasm
- **AOT vs JIT** -- static safety vs runtime profile information
- **Generation vs hand-writing** -- yacc/ANTLR vs hand-written recursive descent (which most production compilers now prefer)
- **Compiler as product vs as library** -- vendor cc -> GCC -> LLVM/MLIR platforms
- **Analysis / synthesis** -- unbroken since 1957

### Where the Field Is Heading

ML-driven heuristics, polyhedral & tensor compilers, verified compilers (CompCert), and ever-higher reusable IRs (MLIR). The 70-year arc bends towards *shared, composable, trustworthy* infrastructure.

---

## Summary & Further Reading

### Key Takeaways

- The first "compiler" (Hopper's A-0, 1952) *linked subroutines*; the modern meaning grew from it.
- FORTRAN (1957) proved compilers could rival hand assembly -- via optimisation.
- Chomsky's hierarchy + BNF made lexing & parsing a science (ALGOL 60, LR, LALR).
- lex/yacc, PCC, GCC then LLVM turned compiler-building into reusable infrastructure.
- SSA (1991) reshaped optimisation; Self's ideas power every modern JIT.
- The **analysis/synthesis** split and the **common-IR** dream are the through-lines.

### Reflection

Almost every "new" compiler idea has a 1950s-90s ancestor. Knowing the lineage tells you *why* today's tools are shaped as they are -- and where the next generalisation will come from.

### Further Reading

- Aho, Lam, Sethi & Ullman -- *Compilers: Principles, Techniques & Tools* (the "Dragon Book", 2nd ed. 2006).
- Robert Nystrom -- *Crafting Interpreters* (free online; the best modern hands-on intro).
- The LLVM project docs & the "LLVM" CGO 2004 paper (Lattner & Adve).
- Kurt Beyer -- *Grace Hopper and the Invention of the Information Age*.
- Knuth, *"On the Translation of Languages from Left to Right"* (1965); Cytron et al., *"Efficiently Computing Static Single Assignment Form"* (1991).

### Next -> Part 02

**The Compiler Pipeline** -- we zoom into the phases this history produced: source -> tokens -> AST -> IR -> optimised IR -> machine code, and how a real compiler is structured.

`Part 01 done` -> `Part 02: The Compiler Pipeline`
