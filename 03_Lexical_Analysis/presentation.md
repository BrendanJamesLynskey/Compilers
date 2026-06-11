# Compilers — Part 03: Lexical Analysis

**Part 03 — From Characters to Tokens: the Scanner**

> The first phase of the front end: how a stream of raw characters becomes a stream of tokens. We build the full theory — regular expressions, finite automata, Thompson's construction and the subset construction — then put it to work in real, table-driven and hand-written scanners, and confront the hard cases that break the clean theory.

`Regex` -> `NFA` -> `DFA` -> `Minimise` -> `Token Stream`

Tokens - Regular Languages - Automata - Scanner Generators

---

## Table of Contents

1. [Topics](#topics)
2. [The Scanner's Role in the Pipeline](#the-scanners-role-in-the-pipeline)
3. [Why a Separate Phase?](#why-a-separate-phase)
4. [Tokens, Lexemes & Patterns](#tokens-lexemes--patterns)
5. [The Token Stream — What Survives](#the-token-stream--what-survives)
6. [Regular Expressions & Regular Languages](#regular-expressions--regular-languages)
7. [Finite Automata — DFA vs NFA](#finite-automata--dfa-vs-nfa)
8. [Thompson's Construction: Regex → NFA](#thompsons-construction-regex--nfa)
9. [Worked Example: NFA for (a|b)\*abb](#worked-example-nfa-for-abab)
10. [Subset Construction: NFA → DFA](#subset-construction-nfa--dfa)
11. [The Subset-Construction Table → DFA](#the-subset-construction-table--dfa)
12. [DFA Minimisation](#dfa-minimisation)
13. [The Construction Pipeline, End to End](#the-construction-pipeline-end-to-end)
14. [Maximal Munch & Tie-Breaking](#maximal-munch--tie-breaking)
15. [Lookahead & Historical Hazards](#lookahead--historical-hazards)
16. [Hand-Written vs Generated Scanners](#hand-written-vs-generated-scanners)
17. [The Generated Table-Driven DFA](#the-generated-table-driven-dfa)
18. [Keywords vs Identifiers](#keywords-vs-identifiers)
19. [Literals: Numbers, Strings, Escapes](#literals-numbers-strings-escapes)
20. [Source Handling: Encodings & Buffering](#source-handling-encodings--buffering)
21. [Location Tracking & Error Recovery](#location-tracking--error-recovery)
22. [Hard Case: Significant Indentation](#hard-case-significant-indentation)
23. [Hard Case: The C "Lexer Hack"](#hard-case-the-c-lexer-hack)
24. [Hard Cases: Interpolation & Nested Comments](#hard-cases-interpolation--nested-comments)
25. [Summary & Further Reading](#summary--further-reading)

---

## Topics

Lexing is the most mathematically settled phase of a compiler — it rests on the theory of regular languages, which gives us a mechanical path from a specification to fast code. This deck walks that path, then the messy reality of real languages.

- **Foundations** — the scanner's role and why it is a separate phase; tokens vs lexemes vs patterns; regular expressions and regular languages; finite automata (DFA, NFA, ε-moves).
- **The construction pipeline** — Thompson's construction (regex → NFA); the subset construction (NFA → DFA); DFA minimisation (Myhill–Nerode / Hopcroft); the worked example `(a|b)*abb`.
- **Scanning rules & implementation** — maximal munch, rule order, lookahead; hand-written vs generated scanners; keywords, identifiers and literals; source handling and the two-buffer scheme.
- **The hard cases** — location tracking and error recovery; significant indentation (INDENT/DEDENT); the C lexer hack; string interpolation and nested comments.

---

## The Scanner's Role in the Pipeline

The lexer (scanner) is the first phase. It reads the source as a flat stream of characters and groups them into **tokens** — the atomic words of the language — which it hands, one at a time, to the parser.

```
Source Text  ->  Lexer / Scanner  ->  Token Stream  ->  Parser
(char stream)    (DFA over chars)     (ID PLUS NUM …)   (builds tree)
```

The parser is the lexer's only client. It typically *pulls* tokens on demand — calling `getNextToken()` each time it needs to advance — rather than the lexer running to completion and materialising a whole array first. The symbol table is shared between phases.

---

## Why a Separate Phase?

In principle the parser could read characters directly. In practice every serious compiler splits lexing off. The reasons are simplicity, speed and modularity — plus a clean theoretical boundary.

- **Simplicity** — the parser's grammar deals in tokens, not characters. It never has to know that whitespace, comments, or the spelling of `!=` exist; the lexer absorbs all that lexical noise.
- **Speed** — lexing is the one phase that touches every byte. A specialised DFA scanner with buffered I/O is far faster than running the parser's machinery per character.
- **Modularity** — input encodings, buffering and character sets live in the lexer; grammar rules live in the parser. Each can be specified, tested and replaced alone.

**The theoretical boundary:** token structure is *regular* (recognised by finite automata, constant memory); program structure is *context-free* (needs a stack). Drawing the line where the language jumps from regular to context-free is exactly the right place to split.

The split is a convention, not a law. Some "scannerless" parsers (PEG, GLR) fold lexing into parsing to handle context-dependent tokenisation, but most production compilers keep the phases separate.

---

## Tokens, Lexemes & Patterns

Three terms easy to conflate:

- **Pattern** — the rule (a regex) defining a token class, e.g. `[a-zA-Z_][a-zA-Z0-9_]*` for an identifier.
- **Lexeme** — one concrete string from the source matching the pattern. In `int count;` the lexemes are `int`, `count`, `;`.
- **Token** — the classified result: the pair `(type, attribute)`. `count` → `(IDENT, "count")`; `;` → `(SEMICOLON)`.

A token carries more than a type: an attribute (the lexeme, an interned id, or a parsed value) and a source location for diagnostics.

```rust
struct Token {
    kind:  TokenKind,   // IDENT, NUMBER, PLUS, LPAREN, KW_IF, ...
    value: Attribute,   // interned symbol id, parsed integer, string contents
    span:  Span,        // { file, byte_start, byte_end, line, column }
}
```

| Source   | Lexeme   | Token (type, attribute)      |
|----------|----------|------------------------------|
| `x`      | `x`      | `(IDENT, sym#42)`            |
| `0xFF`   | `0xFF`   | `(INT, 255)`                 |
| `"hi\n"` | `"hi\n"` | `(STRING, "hi\n")`           |
| `<=`     | `<=`     | `(LE)` — no attribute needed |

---

## The Token Stream — What Survives

The lexer flattens rich source text into a thin stream. Most whitespace and comments are discarded — they exist for humans, not the grammar — though not always.

```c
  int  total = price * qty;   // running cost
```

becomes `KW_INT IDENT("total") ASSIGN IDENT("price") STAR IDENT("qty") SEMI EOF`, with the spaces and the comment dropped.

**Usually stripped:** spaces, tabs, newlines (in free-form languages); line and block comments. The lexer emits nothing and loops back.

**Sometimes kept:** newlines where they are significant (Python, JavaScript ASI, Go); doc comments needed by tooling. *Lossless* lexers (formatters, IDEs) keep "trivia" attached to tokens so the exact text can be reconstructed.

---

## Regular Expressions & Regular Languages

Token patterns are written as regular expressions. A regular expression denotes a *regular language* — a set of strings — built from three operators over an alphabet Σ.

| Operator             | Regex   | Language                       |
|----------------------|---------|--------------------------------|
| Union (alternation)  | `r \| s`| L(r) ∪ L(s)                    |
| Concatenation        | `r s`   | { xy : x∈L(r), y∈L(s) }        |
| Kleene star          | `r*`    | zero or more r's               |
| (base) symbol        | `a`     | { "a" }                        |
| (base) empty         | `ε`     | { "" } the empty string        |

Derived shorthand: `r+` = `rr*`, `r?` = `r|ε`, `[a-z]` = union of ranges, `.` = any char, `r{2,5}` = bounded repetition.

```
digit   = [0-9]
letter  = [a-zA-Z_]
ident   = letter (letter | digit)*
int     = digit+
float   = digit+ "." digit+ ([eE][+-]?digit+)?
ws      = (" " | "\t" | "\n")+
```

**Regular ≠ PCRE.** Lexer regex is the *formal* kind: no backreferences, no lookbehind, no recursion. That restriction guarantees a finite-automaton implementation — and linear-time, backtrack-free scanning.

**Limits:** a regular language cannot count unboundedly — `aⁿbⁿ` and balanced parentheses are *not* regular. That is precisely the parser's job.

---

## Finite Automata — DFA vs NFA

A finite automaton is the machine that recognises a regular language: a finite set of states, a start state, accepting states, and a transition function on input symbols. Two flavours, equivalent in power.

**DFA — Deterministic.** δ : Q × Σ → Q, exactly one move per (state, symbol); no ε-transitions. Acceptance follows the unique path. Trivial and fast to run — a table lookup per character.

**NFA — Non-deterministic.** δ : Q × (Σ ∪ {ε}) → 2^Q, a *set* of next states; may have ε-transitions (move with no input). Accepts if *some* path reaches a final state. Easy to build from a regex, awkward to run directly.

By convention a double-ring state is accepting. An NFA might offer two moves on the same symbol `a`; a DFA always has exactly one.

---

## Thompson's Construction: Regex → NFA

Ken Thompson's 1968 algorithm builds an NFA from a regex compositionally — one small fragment per operator, glued with ε-transitions. Each fragment has exactly one start and one accept state.

- **symbol `a`** — start --a--> accept.
- **concat `r·s`** — accept of r connects by ε to start of s.
- **union `r|s`** — a new start ε-branches to both r and s; both ends ε-join a new accept.
- **Kleene star `r*`** — a new start with ε to r's start and ε straight to accept (zero r's); r's accept ε-loops back to its start (more r's) and ε to the final accept.

The result is an NFA with at most 2× the number of regex symbols in states — linear size, built in linear time. The ε-transitions encode "choose a branch" and "loop or exit" without committing.

---

## Worked Example: NFA for (a|b)*abb

The textbook example: all strings over {a,b} ending in `abb`. A compact NFA has a `(a|b)*` loop on the start state, then a three-step tail `a b b`:

```
        a,b (self-loop on 0)
       ┌───┐
       ▼   │
start →(0)──a──►(1)──b──►(2)──b──►((3))   ((3)) = accepting
```

State 0 is non-deterministic on input `a`: both the self-loop (stay 0) and the edge to state 1 are valid — the NFA "guesses" when the final `abb` begins. A real run must explore both possibilities, which is exactly what the subset construction makes deterministic.

---

## Subset Construction: NFA → DFA

To run an NFA efficiently we make it deterministic. The subset (powerset) construction builds a DFA whose each state is a *set* of NFA states — "all the places the NFA could currently be".

Two helper operations:

- **ε-closure(S)** — S plus every state reachable by ε-moves alone.
- **move(S, c)** — states reachable from S on input `c`.

The algorithm:

1. Start state = ε-closure({start}).
2. For each unmarked subset S and each input `c`: T = ε-closure(move(S, c)); add T as a DFA state if new.
3. A subset is accepting if it contains *any* NFA accept state.

Worst case the DFA has 2^Q states (exponential blow-up), but for real token sets it is almost always small. The `(a|b)*abb` NFA has no ε-moves, so closures are trivial here.

---

## The Subset-Construction Table → DFA

Running the construction on the `(a|b)*abb` NFA. Each row is a DFA state (a subset of {0,1,2,3}); we tabulate where `a` and `b` lead until closed.

| DFA state    | NFA subset | on a | on b | accept?       |
|--------------|------------|------|------|---------------|
| **A** (start)| {0}        | B    | A    |               |
| **B**        | {0,1}      | B    | C    |               |
| **C**        | {0,2}      | B    | D    |               |
| **D**        | {0,3}      | B    | A    | ✓ (contains 3)|

Reading it: from {0} on `a` the NFA could be in {0,1} (loop *and* step) → state B. Only D contains accept-state 3, so only D accepts. The result is a complete, deterministic 4-state DFA:

```
        b              a (self)
      ┌────┐          ┌────┐
      ▼    │          ▼    │
A ──a──► B ──b──► C ──b──► D    (D accepting)
▲ ▲                │        │
│ └────── a ───────┘        │
└────────── b ──────────────┘
```

Every state has exactly one outgoing edge per symbol — deterministic.

---

## DFA Minimisation

The subset construction can leave redundant states — states indistinguishable by any input. Minimisation merges them into the unique smallest DFA for the language.

**Myhill–Nerode.** Two states are equivalent iff for every input suffix they agree on accept/reject. The number of equivalence classes equals the number of states in the minimal DFA — which is unique up to renaming.

**Partition refinement (Hopcroft).**

1. Start with two classes: accepting vs non-accepting.
2. Repeatedly split a class if some input sends its members to different classes.
3. Stop at a fixed point; each class becomes one state.

Hopcroft's 1971 algorithm runs in O(n log n). Example: two "dead" trap states that reject everything and loop to themselves are Myhill–Nerode equivalent and merge into one. A smaller DFA means a smaller transition table and better cache behaviour. `flex` minimises by default.

---

## The Construction Pipeline, End to End

A token specification is a list of regexes, and a fully mechanical pipeline turns it into a fast scanner — exactly what `lex` and `flex` do at build time:

```
Regex specs  ->  NFA       ->  DFA            ->  Minimal DFA  ->  Scanner table
(per token)      (Thompson)    (subset constr)    (Hopcroft)       (+ actions)
```

An alternative is the **derivatives of regular expressions** (Brzozowski) approach, which builds the DFA directly from the regex without an explicit NFA — used by tools like `re2c`.

---

## Maximal Munch & Tie-Breaking

A DFA tells us *whether* a string matches a pattern. A scanner must also decide *where one token ends* when many patterns and lengths are possible. Two rules resolve this.

- **Rule 1 — Longest match (maximal munch).** Always take the longest lexeme that matches any pattern. Run the DFA forward, remembering the last accepting position; when stuck, back up to it. `>=` scans as one `GE`, not `>` then `=`; `index` is one IDENT, not `in` + `dex`.
- **Rule 2 — Rule order (priority).** When two patterns match the *same* longest lexeme, the one listed first wins. This is how keywords beat identifiers: list `if → KW_IF` before the identifier rule. `while` matches both patterns; rule order picks the keyword.

**When maximal munch bites — C++ `>>`.** In `vector<vector<int>>` the lexer greedily eats `>>` as a shift operator, breaking nested templates. Pre-C++11 you had to write `> >` with a space; C++11 added a special parser rule to re-interpret it. Maximal munch is locally correct but context-blind.

---

## Lookahead & Historical Hazards

Backing up to the last accepting state is a form of lookahead — the scanner reads past the token to find where it ends, then rewinds. A few languages need more.

**FORTRAN — no reserved words, ignorable spaces.**

```
DO 5 I = 1,10    ! loop: DO-statement
DO 5 I = 1.10    ! assign DO5I = 1.10
```

Identical until the comma vs dot — arbitrary lookahead decides whether `DO` is a keyword.

**C — `..` vs `.`.** In `1..10` (range operator) the scanner must not munch `1.` as a float. One character of lookahead after the dot disambiguates.

**Maximal munch can over-eat.** In C, `a---b` lexes as `a -- - b` (munch `--` first), which is then a *parse* error, not a lex one.

**Trailing context.** `flex` supports `r/s` — match `r` only when followed by `s`, without consuming `s`. This expresses bounded lookahead declaratively, e.g. the FORTRAN `DO`.

---

## Hand-Written vs Generated Scanners

Two roads: generate a table-driven DFA from a spec, or hand-write a state machine as a big `switch`. Production compilers split surprisingly evenly.

**Generators (spec → code):**

- **lex** (1975, Lesk & Schmidt) / **flex** — the classic, emits a C DFA.
- **re2c** — emits straight-line `goto` code, very fast, no tables.
- **ragel** — state machines with embedded actions; great for protocols.
- Pros: declarative, correct, maintainable. Cons: build dependency, generated code is opaque to debug.

**Hand-written (a `switch` machine):**

- GCC, Clang, V8 and rustc all hand-write their lexers.
- Pros: total control, best error messages, no build-time tool, easy custom lookahead, trivial to special-case the lexer hack / interpolation / raw strings.
- Cons: more code; you own the correctness of the state logic.

```rust
// Hand-written scanner core — the dispatch loop
fn next_token(&mut self) -> Token {
    self.skip_trivia();                       // whitespace + comments
    let start = self.pos;
    let c = self.bump();                      // consume one char
    let kind = match c {
        c if is_id_start(c) => self.ident_or_keyword(start),  // maximal munch
        '0'..='9'          => self.number(start),
        '"'                => self.string_literal(start),
        '+' => if self.eat('=') { PlusEq } else { Plus },     // 1-char lookahead
        '=' => if self.eat('=') { EqEq }   else { Assign },
        '\0' => Eof,
        _   => self.error("unexpected character"),
    };
    Token { kind, span: Span::new(start, self.pos) }
}
```

---

## The Generated Table-Driven DFA

A generated scanner is a tiny driver loop over a 2-D transition table `next[state][char]`, plus an `accept[state]` map saying which token a final state emits.

```c
/* The universal table-driven DFA driver */
int next_token(void) {
    int state = START, last_accept = -1, last_pos = pos;
    while (state != DEAD) {
        int c = input[pos];
        int ns = next[state][c];   /* table lookup */
        if (ns == DEAD) break;
        state = ns; pos++;
        if (accept[state]) {       /* remember longest match */
            last_accept = accept[state];
            last_pos = pos;
        }
    }
    pos = last_pos;                /* back up to last accept */
    return last_accept;            /* the token kind, or error */
}
```

A flex spec is a list of pattern → action rules:

```
%%
[ \t\n]+        ;            /* skip ws */
"if"            return KW_IF;
[a-z]+          return IDENT;
[0-9]+          return INT;
"=="            return EQ;
.               return yytext[0];
%%
```

Why it's fast: one table lookup and compare per input character; no backtracking within the DFA (only the longest-match back-up); tables are compressed (row/column equivalence classes) to fit cache.

---

## Keywords vs Identifiers

Encoding every keyword as its own DFA path bloats the automaton. The standard trick: recognise any identifier, then look the lexeme up in a keyword table and reclassify if it hits.

```rust
fn ident_or_keyword(&self, s: &str) -> TokenKind {
    // one identifier rule in the DFA; keywords are
    // a post-lookup, not separate automaton paths
    match s {
        "if"     => KW_IF,
        "else"   => KW_ELSE,
        "while"  => KW_WHILE,
        "return" => KW_RETURN,
        _        => IDENT,   // interned symbol id
    }
}
```

A perfect hash (`gperf`) makes the lookup O(1) with no collisions — common in C compilers.

- **Reserved keywords** cannot be used as identifiers anywhere (`if`, `class`, `return`). The lexer settles them outright.
- **Contextual keywords** are reserved only in certain positions — `async`, `await`, `yield` in JS/C#; `get`/`set`. The *lexer* emits them as identifiers; the *parser* decides their meaning from context, preserving backward compatibility.

A single identifier rule plus a table keeps the DFA small and lets keyword sets evolve without regenerating the automaton.

---

## Literals: Numbers, Strings, Escapes

Literals carry the most lexical complexity. The lexer both recognises them and decodes their value into the token attribute — parsing `0xFF` to 255, resolving escapes in strings.

**Numeric:**

- Decimal `42`, hex `0xFF`, octal `0o17`/`017`, binary `0b1010`.
- Floats `3.14`, `1e-9`, `.5`, `6.022e23`.
- Digit separators `1_000_000`; suffixes `10u8`, `3.0f`, `100L`.
- Edge cases: `1.` vs `1..2` (float vs range — lookahead); `0x` with no digits (error); overflow (lex the text, range-check on decode); leading-zero octals (C) vs errors (Python 3).

**String & char:**

- Escapes: `\n \t \\ \" \u{1F600} \x41`.
- Char literals `'a'`, multi-byte handling.
- An unterminated string must be a *recoverable* error, not an infinite read.

**Raw & multiline:**

```rust
r"C:\path\no\escapes"      // raw
r#"he said "hi""#          // raw w/ quotes
"""triple
quoted"""                  // multiline
```

Raw strings disable escape processing; the lexer must track the delimiter (e.g. the `#` count) to find the true end.

---

## Source Handling: Encodings & Buffering

Before any theory runs, the lexer must read bytes correctly and feed itself characters efficiently.

**Encoding & normalisation:**

- Decode UTF-8 (the modern default) to code points.
- Skip a leading BOM (U+FEFF) if present.
- Normalise line endings: `\r\n`, `\r`, `\n` → one newline.
- Unicode identifiers (UAX #31); NFC normalisation and confusable concerns.

**The two-buffer scheme:**

- Two halves loaded alternately so a lexeme can straddle a refill.
- A *sentinel* (EOF marker) at each half's end means one bounds-check per char, not two.
- `lexemeBegin` and `forward` pointers bracket the current lexeme.

---

## Location Tracking & Error Recovery

Every token carries a source span so later phases can point at the exact characters in a diagnostic. The lexer is also the first place errors surface — and it must keep going.

**Tracking position:** maintain byte offset, line and column as you bump; a newline resets column and increments line. Tabs and wide chars complicate *column* — many compilers store byte offsets and compute line/column lazily from a line-start index.

A span lets the compiler render a caret line:

```
error: unterminated string literal
  --> main.rs:4:13
   |
 4 |   let s = "oops;
   |           ^ unterminated
```

**Lexer-level errors:** stray characters (a lone `$`, backtick); unterminated string/block comment; invalid escape or numeric literal; bad UTF-8.

**Recovery — don't stop at one:** emit the error, skip the offending char(s), resynchronise and continue (report many errors per run); often emit an `ERROR` token so the parser stays in sync; "panic mode" skips to the next whitespace or delimiter.

---

## Hard Case: Significant Indentation

Python, Haskell and friends use indentation for block structure — but indentation is *not regular* (it needs to count nesting). The fix: a small stack in the lexer that synthesises `INDENT`/`DEDENT` tokens.

```python
if x:
    a = 1
    if y:
        b = 2
    c = 3
```

tokenises (schematically) to:

```
KW_IF IDENT COLON NEWLINE
INDENT IDENT = INT NEWLINE
KW_IF IDENT COLON NEWLINE
  INDENT IDENT = INT NEWLINE
  DEDENT
IDENT = INT NEWLINE
DEDENT
```

**The indent stack:** push the column width at each new, deeper block; on each line compare its indent to the stack top; deeper → push and emit `INDENT`; shallower → pop and emit `DEDENT` until it matches; a level with no equal on the stack is an error.

**Subtleties:** tabs-vs-spaces ambiguity (Python forbids mixing); continuation lines inside brackets suppress NEWLINE/INDENT; at EOF, emit a closing `DEDENT` for every open level.

---

## Hard Case: The C "Lexer Hack"

C's grammar is ambiguous without knowing whether an identifier is a type name or an ordinary variable — and the lexer cannot know that from the characters alone.

```c
typedef int T;

T  * x;     // declaration: pointer-to-T named x
y  * z;     // expression:  multiply y by z
```

The two lines have identical token-shapes (`ID * ID ;`) yet mean completely different things. Whether `T` is a type decides the parse.

**The resolution:** the parser/semantic phase records each `typedef` name in the symbol table; the lexer consults that table and emits `TYPENAME` for a known typedef-name, else `IDENT`. So lexer and parser are *coupled* — information flows *back* from semantic analysis into lexing, violating the clean one-way flow. That's why it's "a hack".

**Alternatives:** Clang lexes plain identifiers and resolves typedef-vs-var in the parser using its own scope tables (a cleaner version of the same idea); GLR / scannerless parsers sidestep it by carrying ambiguity into the parse.

---

## Hard Cases: Interpolation & Nested Comments

Two more places where a pure regular lexer breaks down because the structure is *recursive* — needing a stack or mode switching the bare DFA cannot provide.

**String interpolation:**

```javascript
`sum = ${a + f(b)} done`
```

The `${ … }` embeds full expressions, which can contain more strings, nesting arbitrarily. The lexer runs a *mode stack*: it switches from string-mode into code-mode at `${`, lexes tokens normally, and pops back on the matching `}`. It usually emits pieces: `STR_START`, expression tokens, `STR_MID`, …, `STR_END`.

**Nested comments:**

```rust
/* outer /* inner */ still comment */
```

Languages allowing nesting (Rust, OCaml, D) cannot match comments with a single regex — balanced `/* … */` nesting is *context-free*, not regular. The lexer keeps a *depth counter*, incrementing on `/*` and decrementing on `*/`, ending only at depth 0. C deliberately forbids nesting precisely so a plain DFA suffices.

The pattern is the same each time: when lexical structure becomes recursive, the lexer keeps a tiny piece of explicit state (stack or counter) outside the DFA — a controlled, pragmatic exception to "tokens are regular".

---

## Summary & Further Reading

**Key takeaways:**

- The lexer turns a char stream into a token stream — a separate phase for simplicity, speed and modularity.
- Token = (type, attribute, location); whitespace and comments are usually stripped.
- Token patterns are regular expressions — recognised by finite automata.
- Regex → NFA (Thompson) → DFA (subset) → minimal DFA (Hopcroft) is a fully mechanical pipeline.
- Scanning rules: maximal munch plus rule-order tie-breaking, with bounded lookahead.
- Real scanners are hand-written switches or generated tables (flex, re2c, ragel).
- Hard cases (indentation, the lexer hack, interpolation, nesting) need state beyond a pure DFA.

**Hands-on exercise:** build the DFA for `(a|b)*abb` by hand and check it against the subset-construction table; write a hand-coded scanner for integers, identifiers and `+ - * /` with maximal munch.

**Further reading:**

- Aho, Lam, Sethi & Ullman — *Compilers: Principles, Techniques, and Tools* (the Dragon Book), ch. 3.
- Cooper & Torczon — *Engineering a Compiler*, ch. 2.
- Appel — *Modern Compiler Implementation*, ch. 2.
- Thompson — *Regular Expression Search Algorithm* (CACM, 1968).
- Russ Cox — *Regular Expression Matching Can Be Simple And Fast*.
- Hopcroft — *An n log n Algorithm for Minimizing States in a Finite Automaton* (1971).
- The `flex` and `re2c` manuals.

---

`Part 03 ✓` -> `Part 04: Syntax Analysis`

**Next → Part 04: Syntax Analysis** — context-free grammars, parse trees vs ASTs, top-down (LL) and bottom-up (LR / LALR) parsing, and how the token stream becomes a tree.
