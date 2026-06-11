# Compilers — Part 10: Building a Mini-Compiler

> The capstone. We design a tiny language — **MiniLang** — and build a complete compiler for it in ~400 lines of Python: a hand-written lexer, a recursive-descent + Pratt parser, a type checker, a three-address SSA-ish IR, two optimisations, a bytecode emitter and a stack VM. One running example — an iterative **factorial** — threads through *every* section so you watch the same program transform, stage by stage, until it prints `120`.

`Lex → Parse → Check → Lower → Optimise → Emit → Run`

---

## Table of Contents

1. [Topics](#topics)
2. [What We'll Build — The Toolchain at a Glance](#what-well-build)
3. [MiniLang — Design Choices](#minilang-design)
4. [The Grammar — EBNF](#the-grammar)
5. [Lexer — A Hand-Written Scanner](#lexer-code)
6. [Lexer — The Token Stream](#lexer-tokens)
7. [Parser — The AST Node Definitions](#ast-nodes)
8. [Parser — Recursive Descent + Pratt](#parser-code)
9. [The AST of fact.ml](#the-ast)
10. [Semantic Pass — Symbol Table & Type Checker](#semantic-code)
11. [Semantic Pass — An Error It Catches](#semantic-error)
12. [IR Lowering — To Three-Address Code](#ir-lowering)
13. [The IR & Control-Flow Graph](#ir-cfg)
14. [Two Optimisations — Constant Folding & DCE](#optimisations)
15. [Code Generation — A Tiny Stack-VM ISA](#opcodes)
16. [Code Generation — Disassembly of fact](#disassembly)
17. [The VM — The Interpreter Loop](#the-vm)
18. [Running It — The Whole Pipeline](#running-it)
19. [Testing & Error Messages](#testing)
20. [Where to Go Next](#where-next)
21. [The Whole Series, In One Picture](#series-recap)
22. [Summary — The Series, Closed](#summary)

---

## Topics <a name="topics"></a>

A capstone, not a lecture. Every section carries the *same* program forward one phase. By the end you have seen a real compiler — small, but complete and internally consistent — from source text to a printed result.

- **Front end** — MiniLang design & the running example; the grammar in EBNF; a hand-written lexer; a recursive-descent + Pratt parser; AST node definitions & the tree.
- **Middle end** — semantic pass (symbols, scopes, types); lowering the AST to three-address IR; basic blocks & the CFG; constant folding & dead-code elimination.
- **Back end & runtime** — a stack VM (~14 opcodes); code generation & disassembly; the interpreter loop; an aside on emitting LLVM IR instead.
- **Engineering & closing** — testing each stage; golden tests; good error messages with source spans; where to go next; a recap of the whole 10-part series.

---

## What We'll Build — The Toolchain at a Glance <a name="what-well-build"></a>

Seven passes, each a small function over the previous pass's output. Nothing here is toy in *shape* — this is exactly the skeleton of a real compiler, just scaled down. Same example all the way through.

*(Diagram: source `fact.ml` → Lexer → Parser → Checker → Lower → Optimise → Emit → Stack VM → Output `120`. Front-end passes in green, middle-end in purple, back-end in blue.)*

Each pass is something you already met across the series: lexing (03), parsing (04), type checking (05), IR (06), optimisation (07), codegen (08), and a runtime (09). Here they fit together on one desk.

We implement in **Python** throughout — readable, no build step, and the dataclasses map cleanly onto AST/IR nodes. The *compiled* language is MiniLang; the *host* language is Python.

---

## MiniLang — Design Choices (Small on Purpose) <a name="minilang-design"></a>

Every feature we add is a feature we must lex, parse, check, lower, optimise and codegen. So we keep MiniLang **deliberately tiny** — just enough to be Turing-complete and recognisably a real language.

**In scope:** two value types (`int`, `bool`); variables via `let` plus assignment; arithmetic `+ - * /`; comparison `< > == !=`; `if`/`else` and `while`; first-order functions `fn` with `return`; `print` as a built-in statement.

**Out of scope (on purpose):** no strings, floats, arrays or structs; no closures, generics or modules; no garbage collector (values are unboxed ints); no overloading (one numeric type).

The running example — `fact.ml`:

```rust
fn fact(n) {
  let acc = 1;
  while (n > 1) {
    acc = acc * n;
    n = n - 1;
  }
  return acc;
}
print(fact(5));
```

**Why iterative factorial?** It exercises nearly every feature in one screen: a function, a `let`, a `while` loop with a comparison, mutation of two variables, a call, and a `print`. Its result — **120** — is easy to check by hand. We will see this exact program at every stage.

---

## The Grammar — EBNF <a name="the-grammar"></a>

A compact grammar drives the parser's structure. Statements are **LL(1)** — one token of lookahead picks the rule. Expressions use a precedence-driven sub-grammar (the Pratt parser) rather than the layered rules shown here.

```ebnf
program   = { decl } ;
decl      = fnDecl | statement ;
fnDecl    = "fn" IDENT "(" params? ")" block ;
params    = IDENT { "," IDENT } ;
block     = "{" { statement } "}" ;

statement = letStmt | assign | ifStmt
          | whileStmt | returnStmt | printStmt | exprStmt ;

letStmt    = "let" IDENT "=" expr ";" ;
assign     = IDENT "=" expr ";" ;
ifStmt     = "if" "(" expr ")" block [ "else" block ] ;
whileStmt  = "while" "(" expr ")" block ;
returnStmt = "return" expr ";" ;
printStmt  = "print" "(" expr ")" ";" ;
exprStmt   = expr ";" ;

(* expression grammar — by precedence, lowest to highest *)
expr     = equality ;
equality = compare  { ("==" | "!=") compare } ;
compare  = term     { ("<"  | ">" ) term    } ;
term     = factor   { ("+"  | "-" ) factor  } ;
factor   = unary    { ("*"  | "/" ) unary   } ;
unary    = [ "-" | "!" ] unary | call ;
call     = primary  { "(" args? ")" } ;
args     = expr { "," expr } ;
primary  = NUMBER | "true" | "false" | IDENT | "(" expr ")" ;
```

Reading the notation: `{ x }` is zero or more (a left-associative loop); `[ x ]` is optional; lower rules bind *tighter* (`factor` with `*` before `term` with `+`); `UPPERCASE` names are terminal token classes from the lexer.

---

## Lexer — A Hand-Written Scanner <a name="lexer-code"></a>

The lexer turns the character stream into a **token stream**, tracking line/column for error messages. It is a single pass with a few `if`-ladders — no regex engine, no table generator. (Deck 03 covered the theory; this is the practice.)

```python
KEYWORDS = {"fn","let","if","else","while",
            "return","print","true","false"}

class Tok:
    def __init__(s, kind, val, line, col):
        s.kind, s.val = kind, val
        s.line, s.col = line, col
    def __repr__(s):
        return f"{s.kind}:{s.val!r}@{s.line}:{s.col}"

def lex(src):
    toks, i, line, col = [], 0, 1, 1
    def adv(n=1):
        nonlocal i, col
        i += n; col += n
    while i < len(src):
        c = src[i]
        if c in " \t":      adv()
        elif c == "\n":     i+=1; line+=1; col=1
        elif c.isdigit():
            j = i
            while i < len(src) and src[i].isdigit(): adv()
            toks.append(Tok("NUM", int(src[j:i]), line, col))
        elif c.isalpha() or c == "_":
            j = i
            while i < len(src) and (src[i].isalnum() or src[i] == "_"):
                adv()
            w = src[j:i]
            kind = w if w in KEYWORDS else "IDENT"
            toks.append(Tok(kind, w, line, col))
        else:
            two = src[i:i+2]
            if two in ("==","!=","<=",">="):
                toks.append(Tok(two, two, line, col)); adv(2)
            elif c in "+-*/()<>{},;=!":
                toks.append(Tok(c, c, line, col)); adv()
            else:
                raise LexError(line, col, f"bad char {c!r}")
    toks.append(Tok("EOF", None, line, col))
    return toks
```

Two-character operators (`==`, `!=`) are matched *before* single characters — **maximal munch**. Keywords are just identifiers looked up in a set, so adding one is a one-line change.

---

## Lexer — The Token Stream for fact.ml <a name="lexer-tokens"></a>

Running `lex(open("fact.ml").read())` on our example yields this stream (whitespace and newlines dropped, line:col elided for brevity). This is the *only* thing the parser ever sees — the source text is gone.

```text
fn  IDENT:fact  (  IDENT:n  )  {
let  IDENT:acc  =  NUM:1  ;
while  (  IDENT:n  >  NUM:1  )  {
IDENT:acc  =  IDENT:acc  *  IDENT:n  ;
IDENT:n  =  IDENT:n  -  NUM:1  ;
}
return  IDENT:acc  ;
}
print  (  IDENT:fact  (  NUM:5  )  )  ;
EOF
```

What the lexer decided:

- `fact`, `n`, `acc` → `IDENT` (not keywords).
- `fn`, `let`, `while`, `return`, `print` → keyword kinds.
- `1` and `5` → `NUM` with an *integer* value, already parsed.
- `>`, `*`, `-`, `=` → single-char operator tokens.
- A synthetic `EOF` token terminates the stream.

Note the two faces of `=`: the lexer emits a single `=` token in both `let acc = 1` and `acc = acc * n`. Disambiguating *declaration* from *assignment* is the parser's job, not the lexer's — lexing is context-free. The 9-line program becomes **43** tokens, each carrying a source position so any later phase can point back at the exact character that caused a problem.

---

## Parser — The AST Node Definitions <a name="ast-nodes"></a>

Before parsing, define the tree. Python `@dataclass`es give us cheap, typed, comparable nodes — ideal for golden tests. Statements and expressions are two small families.

```python
from dataclasses import dataclass

# ─── expressions ───
@dataclass
class Num:   value: int
@dataclass
class Bool:  value: bool
@dataclass
class Var:   name: str
@dataclass
class Unary: op: str;  operand: object
@dataclass
class Bin:   op: str;  lhs: object;  rhs: object
@dataclass
class Call:  fn: str;  args: list

# ─── statements ───
@dataclass
class Let:    name: str;  init: object
@dataclass
class Assign: name: str;  value: object
@dataclass
class If:     cond: object; then: list; els: list
@dataclass
class While:  cond: object; body: list
@dataclass
class Return: value: object
@dataclass
class Print:  value: object
@dataclass
class ExprStmt: expr: object

@dataclass
class Fn:  name: str; params: list; body: list
@dataclass
class Program: decls: list
```

Each node also carries a `line` field (omitted above for space) copied from its first token — the thread that lets a type error say *"line 4, column 9"*. Nodes are pure data: no methods, no behaviour. Passes are functions *over* nodes.

---

## Parser — Recursive Descent + a Pratt Core <a name="parser-code"></a>

Statements use **recursive descent** — one method per grammar rule. Expressions use a **Pratt** (precedence-climbing) parser: one loop driven by a binding-power table, far terser than the layered EBNF rules.

```python
class Parser:
    def __init__(s, toks): s.toks, s.i = toks, 0
    def peek(s):  return s.toks[s.i]
    def next(s):  t = s.toks[s.i]; s.i += 1; return t
    def eat(s, k):
        t = s.next()
        if t.kind != k:
            raise ParseError(t, f"expected {k}, got {t.kind}")
        return t

    def statement(s):
        k = s.peek().kind
        if k == "let":    return s.let_stmt()
        if k == "if":     return s.if_stmt()
        if k == "while":  return s.while_stmt()
        if k == "return": return s.return_stmt()
        if k == "print":  return s.print_stmt()
        return s.assign_or_expr()

    # Pratt: left binding powers per operator
    LBP = {"==":1,"!=":1,"<":2,">":2,"+":3,"-":3,"*":4,"/":4}

    def expr(s, min_bp=0):
        left = s.unary()                 # nud
        while s.peek().kind in s.LBP and s.LBP[s.peek().kind] > min_bp:
            op = s.next().kind           # led
            right = s.expr(s.LBP[op])    # climb
            left = Bin(op, left, right)
        return left

    def unary(s):
        if s.peek().kind in ("-","!"):
            op = s.next().kind
            return Unary(op, s.unary())
        return s.call()                  # then primary
```

Because `*` has a higher binding power (4) than `+` (3), `acc * n` binds before any surrounding `+` automatically — precedence falls out of the `min_bp` comparison, no grammar layering needed.

---

## The AST of fact.ml <a name="the-ast"></a>

Parsing the token stream produces a tree rooted at `Program`, holding two children: the `Fn` node for `fact` and the top-level `Print` of `Call fact (Num 5)`.

The `Fn` body is a list of three statements:

- `Let acc = Num 1`
- `While`, whose condition is `Bin ">" (Var n) (Num 1)` and whose body is `[Assign acc, Assign n]`, where:
  - `Assign acc = Bin "*" (Var acc) (Var n)`
  - `Assign n = Bin "-" (Var n) (Num 1)`
- `Return (Var acc)`

*(Diagram: the tree drawn out, with `Program` at the root, `Fn` and `Print` beneath it, and the `While` expanded into its condition and two-statement body, each statement's right-hand side a `Bin` node.)*

---

## Semantic Pass — Symbol Table & Type Checker <a name="semantic-code"></a>

One walk of the AST builds a **scoped symbol table** and checks types. MiniLang has just two types, `int` and `bool`, so the checker is a small recursive function that returns a type or raises with a source span.

```python
class Scope:
    def __init__(s, parent=None):
        s.vars, s.parent = {}, parent
    def define(s, name, ty): s.vars[name] = ty
    def lookup(s, name):
        sc = s
        while sc:
            if name in sc.vars: return sc.vars[name]
            sc = sc.parent
        return None

INT, BOOL = "int", "bool"
ARITH = {"+","-","*","/"}
CMP   = {"<",">","==","!="}

def check_expr(e, scope):
    if isinstance(e, Num):  return INT
    if isinstance(e, Bool): return BOOL
    if isinstance(e, Var):
        ty = scope.lookup(e.name)
        if ty is None:
            err(e.line, f"undefined variable {e.name!r}")
        return ty
    if isinstance(e, Bin):
        lt, rt = check_expr(e.lhs, scope), check_expr(e.rhs, scope)
        if lt != INT or rt != INT:
            err(e.line, f"operator {e.op!r} needs int operands")
        return INT if e.op in ARITH else BOOL   # cmp → bool
```

Statements thread the scope: `Let` calls `scope.define(name, check_expr(init))`; entering a `block` pushes a child `Scope`; `if`/`while` require their condition to be **bool**. Shadowing is resolved by the `lookup` chain — innermost scope wins.

---

## Semantic Pass — An Error It Catches <a name="semantic-error"></a>

Our `fact.ml` passes cleanly. To see the checker bite, break it: use a **bool** where an **int** is required.

```rust
fn fact(n) {
  let acc = 1;
  while (n > 1) {
    acc = acc * (n > 0);  // ← bool * int
    n = n - 1;
  }
  return acc;
}
```

The diagnostic produced:

```text
error: operator '*' needs int operands
 --> fact.ml:4:11
  |
4 |     acc = acc * (n > 0);
  |           ^^^^^^^^^^^^^  rhs has type bool
  |
note: comparison '>' yields bool, not int
```

Other errors the pass catches: using an undefined variable (`lookup` returns `None`); a non-`bool` condition in `if`/`while`; calling an undeclared function or with the wrong arity; `return` outside a function body.

**Why check before lowering?** The IR and VM assume well-typed input — the multiply opcode just pops two ints. Catching type errors in the front end keeps every later phase simple: by the time we lower, "it type-checks" is an invariant we can rely on.

---

## IR Lowering — To Three-Address Code <a name="ir-lowering"></a>

We lower the typed AST to a linear **three-address IR** over **basic blocks**. Every instruction has at most one operator and writes one temporary. Control flow becomes explicit `br` / `cbr` between blocks — the `while` loop unfolds into a header, body and exit block.

```python
@dataclass
class Instr: op: str; args: tuple; dst: str = None
@dataclass
class Block: label: str; instrs: list; term: object

class Lower:
    def __init__(s): s.n = 0; s.blocks = []
    def temp(s): s.n += 1; return f"t{s.n}"

    def expr(s, e, blk):
        if isinstance(e, Num):
            t = s.temp(); blk.append(Instr("const",(e.value,),t)); return t
        if isinstance(e, Var):  return e.name
        if isinstance(e, Bin):
            a = s.expr(e.lhs, blk); b = s.expr(e.rhs, blk)
            t = s.temp(); blk.append(Instr(e.op,(a,b),t)); return t

    def while_(s, node):
        head, body, end = s.new_blocks(3)
        s.emit_br(head)
        # header: test, then conditional branch
        c = s.expr(node.cond, head.instrs)
        head.term = Cbr(c, body.label, end.label)
        # body: statements, then back-edge to header
        for st in node.body: s.stmt(st, body.instrs)
        body.term = Br(head.label)
        s.cur = end           # fall through to exit block
```

**SSA-ish:** each temporary `t1, t2, …` is assigned *once* — the defining property of SSA. Named locals (`acc`, `n`) are still reassigned here; a full SSA pass (deck 06) would rename them and insert **φ**-nodes at the loop header.

---

## The IR & Control-Flow Graph <a name="ir-cfg"></a>

Here is `fact` lowered to four basic blocks. The **φ**-nodes at the loop header pick the value from the entry edge or the back-edge — this is the SSA form (deck 06) the optimiser actually rewrites.

```llvm
fn fact(n):
entry:
  acc.0 = const 1
  br header

header:                       ; loop test
  n.1   = phi [n.0, entry], [n.2, body]
  acc.1 = phi [acc.0, entry], [acc.2, body]
  t1    = gt n.1, 1
  cbr t1, body, exit

body:
  acc.2 = mul acc.1, n.1
  n.2   = sub n.1, 1
  br header

exit:
  ret acc.1
```

*(Diagram: a four-node CFG. `entry` → `header`; `header` branches `true` → `body` and `false` → `exit`; `exit` → `ret acc`; and a dashed **back-edge** `body` → `header`.)*

The dashed back-edge `body → header` is what makes this a loop. The φ at the header is the only place two definitions of `n` / `acc` meet.

---

## Two Optimisations — Constant Folding & DCE <a name="optimisations"></a>

SSA makes peephole optimisations almost trivial. We add two passes (deck 07): **constant folding** evaluates compile-time-constant instructions, and **dead-code elimination** deletes instructions whose result is never used.

```python
# Constant folding: fold ops on literal args
FOLD = {"add":lambda a,b:a+b, "sub":lambda a,b:a-b,
        "mul":lambda a,b:a*b, "gt":lambda a,b:int(a>b)}

def fold(blocks, consts):
    for blk in blocks:
        for ins in blk.instrs:
            a = [consts.get(x, x) for x in ins.args]
            if ins.op in FOLD and all(isinstance(x,int) for x in a):
                v = FOLD[ins.op](*a)        # evaluate now
                ins.op, ins.args = "const", (v,)
                consts[ins.dst] = v         # propagate

def dce(blocks):
    used = {x for b in blocks for i in b.instrs
              for x in i.args} | live_in_terms(blocks)
    for b in blocks:
        b.instrs = [i for i in b.instrs
                    if i.dst in used or i.op == "call"]
```

Before (a constant sub-expression plus a dead `let`):

```llvm
  t1 = const 2
  t2 = const 3
  t3 = mul t1, t2     ; 2 * 3
  r  = add t3, n      ; r = 6 + n
  u  = const 99       ; let unused = 99
  ret r
```

After fold + DCE:

```llvm
  t3 = const 6        ; 2*3 folded
  r  = add t3, n
  ret r               ; 'u' deleted — never used
```

**Order matters:** fold first (it creates new constants and may make values dead), then DCE sweeps them up. Real compilers iterate the two to a fixpoint.

---

## Code Generation — A Tiny Stack-VM ISA <a name="opcodes"></a>

The back end targets a **stack machine** (like JVM / CPython bytecode, deck 09). Values live on an operand stack; locals live in numbered slots per call frame. Fourteen opcodes are enough for all of MiniLang.

| Opcode | Stack effect | Meaning |
|---|---|---|
| `CONST k` | → k | push literal k |
| `LOAD s` | → locals[s] | push local slot s |
| `STORE s` | v → | pop into slot s |
| `ADD/SUB/MUL/DIV` | a b → r | integer arithmetic |
| `LT / GT / EQ` | a b → 0\|1 | comparison |
| `JMP t` | — | jump to offset t |
| `JZ t` | v → | jump if popped == 0 |
| `CALL f n` | args → r | call f with n args |
| `RET` | v → | return top of stack |
| `PRINT` | v → | pop & print |
| `HALT` | — | stop the VM |

**Lowering IR → bytecode:** each three-address instruction expands to a short opcode sequence — `t = mul a, b` becomes `LOAD a; LOAD b; MUL; STORE t`. A `cbr` becomes the condition followed by `JZ exit`. Block labels resolve to integer offsets in a second pass (back-patching).

**Aside — emitting LLVM IR instead.** Our three-address IR maps almost 1:1 onto LLVM's SSA form (`mul` → `mul i64`, `cbr` → `br i1`, `phi` → `phi`). Emit that, then `llc` does register allocation and instruction selection for you:

```llvm
define i64 @fact(i64 %n) {
  ; ... our IR translated to LLVM SSA ...
}
```

---

## Code Generation — Disassembly of fact <a name="disassembly"></a>

Here is the emitted bytecode for the `fact` function after labels are back-patched to offsets. Locals: slot **0 = n** (the argument), slot **1 = acc**. Compare it instruction-by-instruction with the IR above.

```text
fn fact/1:            ; arity 1
 0   CONST   1        ; acc = 1
 1   STORE   1
                      ; --- header (offset 2) ---
 2   LOAD    0        ; n
 3   CONST   1
 4   GT               ; n > 1
 5   JZ      15       ; if !cond → exit
 6   LOAD    1        ; acc
 7   LOAD    0        ; n
 8   MUL              ; acc * n
 9   STORE   1        ; acc = ...
10   LOAD    0        ; n
11   CONST   1
12   SUB              ; n - 1
13   STORE   0        ; n = ...
14   JMP     2        ; back to header
15   LOAD    1        ; exit: acc
16   RET
```

Top-level code:

```text
 0  CONST 5
 1  CALL  fact 1   ; fact(5) → 120
 2  PRINT
 3  HALT
```

Reading it: offsets 0–1 are the loop pre-header (`acc = 1`); offset 5's `JZ 15` is the `cbr` (exit when `n > 1` is false); offset 14's `JMP 2` is the loop back-edge. Stack discipline: each `LOAD`/`CONST` pushes, each binary op pops two and pushes one. The whole function is **17** bytecodes. No registers to allocate — the stack *is* the calling convention.

---

## The VM — The Interpreter Loop <a name="the-vm"></a>

A stack VM is a `while` loop with a dispatch on the opcode. Each frame holds an instruction pointer, a local array and shares the operand stack. This loop executes the bytecode above and prints the result.

```python
class Frame:
    def __init__(s, code, locals_):
        s.code, s.ip, s.locals = code, 0, locals_

def run(program, entry):
    stack, frames = [], [Frame(entry, [])]
    while frames:
        f = frames[-1]
        op, *a = f.code[f.ip]; f.ip += 1
        if   op == "CONST": stack.append(a[0])
        elif op == "LOAD":  stack.append(f.locals[a[0]])
        elif op == "STORE":
            v = stack.pop()
            while len(f.locals) <= a[0]: f.locals.append(0)
            f.locals[a[0]] = v
        elif op == "ADD": b=stack.pop(); stack.append(stack.pop()+b)
        elif op == "SUB": b=stack.pop(); stack.append(stack.pop()-b)
        elif op == "MUL": b=stack.pop(); stack.append(stack.pop()*b)
        elif op == "GT":
            b=stack.pop(); stack.append(int(stack.pop() > b))
        elif op == "JMP": f.ip = a[0]
        elif op == "JZ":
            if stack.pop() == 0: f.ip = a[0]
        elif op == "CALL":
            fn, n = program[a[0]], a[1]
            args = [stack.pop() for _ in range(n)][::-1]
            frames.append(Frame(fn.code, args))
        elif op == "RET":
            frames.pop()                  # back to caller
        elif op == "PRINT": print(stack.pop())
        elif op == "HALT":  return
    # RET leaves the return value on the shared stack
```

**Why so short?** No registers, no addressing modes, no calling-convention bookkeeping. The operand stack carries arguments and results; `RET` just pops the frame and leaves the value behind for the caller. About 25 lines runs the whole language.

---

## Running It — The Whole Pipeline <a name="running-it"></a>

The driver wires every pass together. From source file to printed answer is six function calls — and out comes **120**, matching `5! = 5×4×3×2×1`.

```python
def compile_and_run(path):
    src    = open(path).read()
    toks   = lex(src)                 # 03 — lexer
    ast    = Parser(toks).program()   # 04 — parser
    check(ast)                        # 05 — semantics
    ir     = Lower().run(ast)         # 06 — lowering
    fold(ir.blocks, {}); dce(ir.blocks)  # 07 — optimise
    program = emit(ir)                # 08 — codegen
    run(program, program["__main__"]) # 09 — VM

if __name__ == "__main__":
    compile_and_run("fact.ml")
```

```text
$ python minilang.py fact.ml
120
```

Trace of the loop (slot values):

| iter | n | acc | n>1? |
|---|---|---|---|
| start | 5 | 1 | true |
| 1 | 4 | 5 | true |
| 2 | 3 | 20 | true |
| 3 | 2 | 60 | true |
| 4 | 1 | 120 | false |
| exit | 1 | **120** | — |

**End-to-end consistency:** the same program produced the tokens, AST, IR, bytecode, and now the answer — every artefact in this deck describes *this one execution*. That consistency is the whole point of the capstone.

---

## Testing & Error Messages <a name="testing"></a>

A compiler is a function; test it like one. Each stage takes a value and returns a value, so each is testable in isolation — and the whole pipeline is testable with **golden files**.

Per-stage tests:

- **Lexer:** assert the token list for a snippet.
- **Parser:** `repr(ast)` equality — dataclasses compare structurally.
- **Checker:** assert that bad inputs raise with the right line.
- **Optimiser:** assert IR-before → IR-after.
- **VM:** feed hand-written bytecode, check the result.

Golden / snapshot tests keep `tests/*.ml` beside `*.expected`. Run each through the full pipeline and diff stdout. A failing diff is a regression — cheap, high-coverage, and they double as documentation.

```python
import subprocess, glob

def test_golden():
    for ml in glob.glob("tests/*.ml"):
        expected = open(ml.replace(".ml",".expected")).read()
        out = subprocess.run(
            ["python","minilang.py",ml],
            capture_output=True, text=True).stdout
        assert out == expected, ml

# tests/fact.ml  +  tests/fact.expected ("120\n")
# tests/fib.ml   +  tests/fib.expected  ("55\n")
```

**Good errors = source spans.** Every token carries `line`/`col`; every AST and IR node copies it. So a message can always point at the offending source — `fact.ml:4:11` — with a caret and a note. Carry positions *everywhere*; never throw them away.

---

## Where to Go Next <a name="where-next"></a>

MiniLang is a skeleton. Every shortcut we took is a door to a deeper topic.

**Extend MiniLang:** add `float`, then strings & a heap (now you need a **GC** — deck 09); real SSA with φ-insertion & proper liveness; a register VM, or emit LLVM IR and let `llc` finish; closures (you'll meet upvalues & capture); better diagnostics with error recovery and multiple errors per run.

**Build a real front end:** parse a non-trivial subset of C or Python; wire your IR into **LLVM** via its C API or `llvmlite`; or target **WebAssembly** and run in a browser.

**Canonical reading:**

- **Crafting Interpreters** — Bob Nystrom. Build *two* Lox implementations, tree-walking then a bytecode VM. The best place to start (free online).
- **LLVM Kaleidoscope Tutorial** — a JIT-compiled language on LLVM, front to back.
- **Modern Compiler Implementation** — Andrew Appel (ML / Java / C editions). The classic course text.
- **The Dragon Book** — Aho, Lam, Sethi & Ullman. The reference.
- **Nand2Tetris** — build the whole stack, logic gates to compiler.
- **Engineering a Compiler** — Cooper & Torczon, strong on optimisation & SSA.

---

## The Whole Series, In One Picture <a name="series-recap"></a>

Ten decks, one journey: from the history of the idea to a working compiler you built yourself. Each phase we coded today was a deck of its own:

1. **History** — Hopper's A-0 → FORTRAN → the Dragon Book → GCC → LLVM.
2. **Pipeline** — the phases of a compiler, end to end.
3. **Lexing** — regular languages, DFAs, tokens (*front end*).
4. **Parsing** — LL / LR, grammars, the AST (*front end*).
5. **Semantics** — symbol tables, scopes, type systems (*front end*).
6. **IR** — three-address code, the CFG, SSA & φ (*middle end*).
7. **Optimisation** — folding, DCE, CSE, LICM (*middle end*).
8. **Code Generation** — instruction selection, register allocation (*back end*).
9. **Runtime, JIT & Backends** — VMs, JITs, garbage collection (*back end*).
10. **Building a Mini-Compiler** — *you are here* — all of it, in ~400 lines.

---

## Summary — The Series, Closed <a name="summary"></a>

**Key takeaways:**

- A compiler is a *pipeline of functions*: each pass maps one representation to the next.
- The front end (lex, parse, check) turns text into a typed AST.
- The middle end lowers to an **IR** over basic blocks and optimises it — SSA makes that easy.
- The back end emits code — bytecode for a VM, or machine code via LLVM.
- Carry **source positions everywhere** for good diagnostics.
- Test each stage in isolation; golden-test the whole.
- ~400 lines of Python is enough for a complete, real-shaped compiler.

**One program, seven forms:** `fact(5)` → 43 tokens → an AST → 4 basic blocks of SSA-ish IR → folded IR → 17 bytecodes → **120**. Same program, every section.

**Further reading:** *Crafting Interpreters* (Nystrom, free online); the *LLVM Kaleidoscope* tutorial (llvm.org); *Modern Compiler Implementation* (Appel); *Engineering a Compiler* (Cooper & Torczon); *Nand2Tetris* (nand2tetris.org); *The Dragon Book* (Aho/Lam/Sethi/Ullman).

From Grace Hopper's A-0 (deck 01) to your own working VM (deck 10) — you've now seen, and built, the whole arc from source text to running code.

`Part 10 ✓ → Series Complete → Back to the Hub`
