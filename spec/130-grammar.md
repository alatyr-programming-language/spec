# Alatyr Language Specification

## Chapter — Grammar (lexis and EBNF)

> **Status — draft (under review).** This chapter is a draft and is not yet
> accepted; the specification as a whole is under development. See the status note
> at the top of the Overview chapter.

This chapter is the **consolidated grammar reference**: the lexical structure
(decisions SYN-1, SYN-3/SYN-4) and the complete context-free grammar of Alatyr, in one place.
The grammar fragments scattered through the feature chapters (each chapter's *Syntax*
section) are gathered here; where a production also appears in a feature chapter, the
**two MUST agree**, and any divergence is a defect resolved in favor of the rule that
matches the feature chapter's prose. This chapter is **normative for the syntax**; the
**meaning** of each construct is normative in its feature chapter (referenced by
anchor).

The manifest is parsed with this source lexis and grammar where it is ordinary Alatyr
source; the manifest appendix adds the configuration **data-subset** and schema restrictions
for the `Package` value (Tooling §1; Manifest appendix). This grammar is also **not**
the GAS the compiler emits (Codegen).

---

### 1. Notation

The grammar is written in **EBNF** with this metasyntax (uniform across the whole
specification):

| Form          | Meaning                                          |
|---------------|--------------------------------------------------|
| `name ::= …`  | a production defining the nonterminal `name`     |
| `"text"`      | a literal terminal (a token spelled exactly so)  |
| `a \| b`       | alternation (a or b)                             |
| `{ a }`       | repetition — **zero or more** of `a`             |
| `[ a ]`       | option — **zero or one** of `a`                  |
| `( a b )`     | grouping                                         |
| `(* … *)`     | a comment in the metalanguage (not part of Alatyr) |

Lowercase hyphenated names (`type-expr`, `match-arm`) are nonterminals. Terminals are
either quoted literals or the named lexical classes of §2 (`ident`, `int`, `float`,
`string`, `char-lit`, `bool-lit`). Whitespace and comments (§2.5) may
appear between any two tokens and are not shown in productions; **statement and
element separators** (§2.6) are significant and are written explicitly as `sep` /
`item-sep`.

---

### 2. Lexical structure (SYN-1, SYN-3, SYN-4)

#### 2.1 Source text

Source files are **UTF-8** (Overview §1). A file is a sequence of Unicode code points
forming tokens, separators, whitespace, and comments. The file extension is `.al`.

Tokens are formed by **longest match** (maximal munch): at each point the lexer takes the
longest valid token. So `..=` is one token (not `..` then `=`), and **`...`** (the
variadic trailing-rest, §3.6) is one token, **distinct** from `..` and `..=`. Each
**compound-assignment** operator `+= -= *= /= %= &= |= ^=` (§3.2) is likewise a single
token (`+=`, not `+` then `=`); none of them collides with the comparison tokens
`== != <= >=`, since no compound-assignment glyph is `< > = !`.

#### 2.2 Identifiers

```ebnf
ident      ::= ident-start { ident-cont }
ident-start ::= letter | "_"
ident-cont  ::= letter | digit | "_"

letter     ::= "A" … "Z" | "a" … "z"      (* ASCII in v1; Unicode identifiers are additive (FND-6) *)
digit      ::= "0" … "9"
hex-digit  ::= digit | "a" … "f" | "A" … "F"
oct-digit  ::= "0" … "7"
bin-digit  ::= "0" | "1"
char       ::= any Unicode scalar value other than the enclosing quote or "\"
```

Identifiers are **case-sensitive and underscore-significant**: `my_func`, `myfunc`, and
`MyFunc` denote **three distinct** entities, `_tmp` ≠ `tmp`, and identifiers are compared
**by their code points, as written** (SYN-1). The underscore is a non-significant separator
only in **numeric literals** (`1_000` = `1000`, §2.4) — never in identifiers.

A linker symbol is the **spelling as declared**, with `::` paths mangled to `__`
(Modules §6; MOD-6), unless overridden by `@export("name")` (§3.11). Keywords (§2.3) are
matched by exact spelling.

**v1 identifiers are ASCII** (`letter` / `digit` above, plus `_`); **non-ASCII
identifiers are additive** (FND-6) — case-sensitivity removes the case-folding barrier, so a
future addition is NFC + UAX #31, with ASCII still required for **exported** symbols
(Modules §6.1). String and character literals are already full Unicode (§2.4).

#### 2.3 Keywords and reserved words

The following are **keywords** (SYN-2). They are reserved and may not be used as
ordinary identifiers (subject to the contextual notes):

```
pub  mut  comptime  unchecked  when
and  or  not
if  else  match  loop  while  for  in  break  continue  return  defer
struct  enum  union  fn  type  mod  abi  dyn
```

- **Verification axis** (CG-6): `unchecked` is a keyword; `checked` and `runtime` are
  **not** keywords — they are unnamed defaults.
- **Parameter directions** `in` / `out` are **contextual** keywords: they are keywords
  in a parameter position (and `in` also in `for x in …`); `in out` is the two-token
  direction. Elsewhere they are ordinary identifiers only where the grammar cannot
  expect a direction.
- **Value introducers** `struct` / `enum` / `union` / `fn` / `mod`
  introduce type-literals, function values, and modules (FN-1/SYN-6). A
  **generic type** is an ordinary type-returning function `fn(…) -> type`, **not** an
  introducer (SYN-6). `type` is **not** an introducer: in a type position (`T : type`) it
  names the **kind of types** (the type of all types) — that is its only role (SYN-6).
  **`abi` is not an introducer** either — an ABI value is the ordinary `Abi(...)` struct
  value (FN-5/SYN-6); the bare word is **reserved** (the `@abi` attribute spelling).
- **`dyn` is a type-constructor keyword** (FN-11): in a **type** position it prefixes a
  function-value type — `dyn fn(T…) -> R` — denoting the type-erased `{code, env}` fat
  closure (Functions §1.6). It switches the representation (thin code pointer → fat pair),
  so it is a **keyword**, not an `@`-attribute (which is compile-time only). It is **not** an
  introducer: construction is the prelude word-function `dyn_over` (below), not a `dyn{…}`
  literal.

**Reserved for additive growth** (semantics deferred; CC-6): `async`, `await`. They are
reserved now so adding the feature later does not break programs.

**Not keywords:** built-ins are **prelude identifiers** (OP-1) — flat word-functions
(`typeinfo`, `bitcast`, `compiles`, `resolves`, `embed`, `forget`, `panic`, `exit`, `assert`,
`asm`, the pointer ops `ptr`/`deref`, the layout words `size`/`align`, the dyn-closure builder
`dyn_over` (FN-11), instruction names, …)
and members of the **prelude modules** `atomic` / `volatile` (`atomic::load`, …, reached by
`::`). The constants `true`, `false`, `NaN`, `Infinity` — and the no-initializer marker `uninit` (Type System §9.4) — likewise. `extern` is **not** a keyword — it is the
attribute `@extern` (§3.11). `use` does not exist (import is a declaration; MOD-4).

#### 2.4 Literals

```ebnf
literal    ::= int | float | bool-lit | char-lit | string

int        ::= dec-int | hex-int | oct-int | bin-int
dec-int    ::= digit { digit | "_" }
tuple-index ::= digit { digit }                   (* a tuple element index: plain decimal, no `_` separators or type suffix — `t.0`, `t.10` (postfix §3.4) *)
hex-int    ::= "0x" hex-digit { hex-digit | "_" }
oct-int    ::= "0o" oct-digit { oct-digit | "_" }
bin-int    ::= "0b" bin-digit { bin-digit | "_" }

float      ::= dec-int "." dec-int [ exp ]
             | dec-int exp
             | hex-float
exp        ::= ("e" | "E") [ "+" | "-" ] dec-int
hex-float  ::= "0x" ( hex-digit { hex-digit | "_" } [ "." { hex-digit | "_" } ]
                    | "." hex-digit { hex-digit | "_" } )
               ( "p" | "P" ) [ "+" | "-" ] dec-int   (* C-style hex float; the binary
                                                        exponent `p` is mandatory — it
                                                        disambiguates from a `0x` int (SYN-3) *)

bool-lit   ::= "true" | "false"                   (* prelude constants, not keywords *)

char-lit   ::= "'" ( char | escape ) "'"          (* a Unicode code point *)
string     ::= '"' { char | escape } '"'          (* yields str: validated UTF-8 bytes *)
escape     ::= "\" ( "n" | "t" | "r" | "0" | "\\" | "'" | '"'
                     | "x" hex-digit hex-digit
                     | "u" "{" hex-digit { hex-digit } "}" )
```

- The underscore `_` is a non-significant digit separator in all integer and float
  literals (`1_000` = `1000`; SYN-3).
- A `string` literal has type `str` (validated UTF-8 bytes; base prelude). A `char-lit`
  is a **Unicode code point**.
- There is **no separate bytes literal**: exact bytes are written as an ordinary
  `[u8; N]` **array value** with integer elements (e.g. `[0x48, 0x69]`; §3.4
  `array-ctor`). The `\xNN` form is an **escape inside a `char`/`string` literal**, not
  a byte literal of its own (SYN-3; Assembly §11). Byte data is thus a value, not a
  lexical token.
- `NaN` / `Infinity` are float **prelude constants** (not literals; SYN-3).

#### 2.5 Comments

```ebnf
comment    ::= "#"  { any-char-until-eol }          (* line comment *)
doc-comment ::= "##" { any-char-until-eol }          (* documentation comment *)
```

Comments are **line-only**; there are no block comments (SYN-3). Adjacent comment lines
(no blank line between) **merge** into one logical comment block. A `##` doc-comment
block placed **immediately before a declaration** is captured as that declaration's
documentation (for tooling and this specification).

#### 2.6 Separators and line continuation (SYN-4)

Two separator contexts:

```ebnf
sep        ::= newline | ";"          (* between statements / declarations in a block; match arms *)
item-sep   ::= newline | ","          (* between sequence elements: args, fields, aggregate elements *)
```

- **Statements / declarations** in a block, and **`match` arms**, are separated by a
  **newline or `;`** (`;` allows several on one line).
- **Sequence elements** (call arguments, struct fields, array/tuple elements) are
  separated by a **comma or a newline** (one-per-line without commas is allowed; a
  trailing comma is allowed).
- **Line continuation** is deterministic (Go/Swift-style): a newline does **not**
  terminate when the line is plainly incomplete — the last token cannot end a
  construct (a trailing binary operator, `=`, `,`, `.`, or an open bracket), or the
  next line begins with a continuing token. **Inside `( … )` and `[ … ]` newlines are
  free** (never terminating).
  - **Exception — an operator-glyph binding head.** A *non-prefix* operator at a line
    start (`/`, `%`, `==`, `<=`, `|`, …) ordinarily **continues** the previous line
    (it cannot begin a fresh expression). But an **operator-name binding** (§4.5 / Type
    System §4.5 — the operator glyph **is** the declaration's name) begins exactly that
    way: `/ := fn(…)`. So when a line-leading operator glyph is **immediately followed
    by `:=` or `:`**, it is a **binding head**, not a continuation — the preceding
    newline stays a separator. (Prefix-capable glyphs `+`/`-`/`*`/`&`/`~` already begin
    a fresh line, so they need no exception; this disambiguates only the non-prefix
    operators, which are the rest of the bindable set.)

A trailing `item-sep` / `sep` before a closing bracket is permitted; consecutive
`sep`s are collapsed.

---

### 3. Grammar

The remaining sections give the consolidated context-free grammar. Productions are
grouped by area; each group cites the feature chapter normative for its meaning.

#### 3.1 Programs, modules, blocks

```ebnf
program     ::= [ file-attribute { sep } ] module-body  (* a file is a module body, optionally preceded by file attrs *)
module-body ::= { sep } { module-item { sep } }        (* a region of declarations and top-level-only items *)
module-item ::= declaration | test-item                 (* `@test` is top-level-only (Tooling §4.1; TOOL-5) *)
file-attribute ::= "@limits" "(" [ arg-list ] ")"       (* per-file stricter limits (Tooling §2.3; FND-11) *)

block       ::= "{" { sep } { statement { sep } } "}"  (* an executable body; value = final expr (SYN-6) *)
statement   ::= declaration                            (* a local binding (§3.2) *)
              | assignment                              (* store to a place (§3.2) *)
              | return-stmt | break-stmt | continue-stmt
              | while-stmt | for-stmt | labeled-loop-stmt (* @label(name) while/for (§3.7) *)
              | labeled-point                             (* @label(name) <instruction> — code-point/jmp target (§3.7; CF §11); CF-5 *)
              | defer-stmt
              | expr                                    (* expression statement: if/match/loop/block, call, … *)
```

**Module scope is a region of declarations plus top-level-only items** (order-independent;
Declarations §7.1): a file's top level, and an inline module body, contain `module-item`s
only. Executable statements — `assignment`, `return`/`break`/`continue`, `while`/`for`,
`defer`, and a bare expression — belong to **function and control-flow blocks**, not
module scope. Because `module-body` admits only `module-item`, such a statement at module
scope is **not in the grammar** and is rejected at the **Parse stage** (Tooling §5). An
inline module is an ordinary binding whose value is the `mod`-introducer (§3.4):
`Geometry := mod { … }` (Modules §10) — there is no separate `module-decl` production. A
file-level `@limits(...)` attribute may precede the module body and sets a stricter
per-file limits contract.

A `block` is an **expression** (Control Flow §3/§7; SYN-6). **Its value is its final
element when that element is an expression**; a non-final expression is an expression
*statement* evaluated for effect with its value discarded, and a trailing `sep` is
collapsed (§2.6) and does **not** suppress the value. If a block's last element is
**not** an expression (a declaration, assignment, `while`/`for`, `defer`), the block
yields **no value** — there is no unit type — and using such a block where a value is
required is ill-formed. A block whose last element **diverges** (`return`/`break`/
`continue`, or a call of type `Never`) is **not** in this list: it has type **`Never`**
(the bottom type, Control Flow §3), which unifies with any type, so it is well-formed in
value position (e.g. the `else { return }` of a value-position `if`). `if`/
`match`/`loop` and a nested `block` are expressions and reach statement position
through `expr`; `while`/`for` are statements, never expressions.

#### 3.2 Declarations, top-level items, and bindings (Declarations §9; SYN-5)

```ebnf
declaration ::= [ "pub" ] extern-decl                  (* a body-less import may be re-exported: pub printf := @extern … (Modules §3/§7) *)
              | { modifier } binding
test-item   ::= "@test" "(" string ")" fn-value        (* @test("desc") fn() { … } — a runtime test; collected by `alatyr test`, ignored by `build`. fn() takes no params, returns nothing, not comptime/generic (a Semantic check) *)
binding     ::= ident ":" type-expr [ when-clause ] "=" expr   (* typed + initialized; when gates the declaration (Comptime §7.1) *)
              | ident ":=" expr [ when-clause ]       (* inferred; when at the end *)
              | "(" ident { item-sep ident } ")" ":=" expr [ when-clause ]  (* multi-name projection — meaning by RHS kind: a module/namespace path → member projection by name `(fmt,vec):=std` (Modules §4.1.1); a multi-output call → positional value destructuring (Functions §3.2) *)
              | ident ":" type-expr [ when-clause ]   (* uninitialized (Declarations §4) *)
(* `when` is the general declaration guard (Comptime §7.1; CT-5): it gates ANY declaration's
   existence, not only a function's. A declaration carries **at most one** `when`; for a
   function *value* it conventionally sits in the `fn-sig` (next to the params it constrains,
   above) — the same `when`, not a second one. *)
modifier    ::= "pub" | "mut" | "comptime" | attribute

assignment  ::= place ( "=" | compound-op ) expr      (* store to an existing place — not a declaration *)
compound-op ::= "+=" | "-=" | "*=" | "/=" | "%=" | "&=" | "|=" | "^="  (* compound assignment, sugar: a ⊕= b ≡ a = a ⊕ b, place evaluated once (OP-2; Memory §1) *)
```

`:=` distinguishes a **declaration** from an **assignment** (`place = expr`, store).
A **compound assignment** `place ⊕= expr` is **sugar** for `place = place ⊕ expr` with
the place evaluated **once** (the desugaring is normative in Memory §1); `⊕` ranges over
exactly the binary glyph operators `+ - * / % & | ^` (OP-2) — there is no `and=`/`or=`
(short-circuit control, not functions) and no shift/rotate compound (those have no glyph,
OP-2). Because the operator is an overloadable operator-function (TYP-5), any type that
defines `⊕` gets `⊕=` for free.
`pub` re-exports/exports a name within the module tree (Modules §3); import / alias /
re-export are ordinary declarations whose value is a `::`-path (§3.3, Modules §10) —
there is no `use` (MOD-4).

#### 3.3 Paths, places, and types (Type System §11; Modules §10)

```ebnf
path        ::= ident { "::" ident }                  (* :: = namespace navigation (Modules §2) *)

place       ::= ident                                  (* an assignable location (lvalue) *)
              | place "." ident                        (* struct field / raw-union member *)
              | place "." tuple-index                  (* tuple element *)
              | place "[" expr "]"                      (* index *)
              | "deref" "(" expr ")"                      (* dereference as a place; ≡ expr.deref() (Memory §4.3) *)
              | path

type-expr   ::= { attribute } type-atom                (* prefix attributes decorate a type: @align(N) T, @require(pred) T, @nonzero u32 — the prefix surface of §8/§8.1 (CT-10); §3.6 *)
type-atom   ::= ident                                  (* bitsN, uN/iN/fN, usize, char, str, bool, … *)
              | path
              | struct-type | enum-type | union-type | array-type | tuple-type
              | type-atom "(" arg-list ")"             (* generic instantiation: Vec(T), Option(T), Result(T,E) *)
              | [ "scoped" ] "ptr" "(" [ "mut" ] type-expr ")"  (* pointer type (Memory §4.1); the `scoped` qualifier marks a **second-class return** — meaningful only as a result type; a `scoped` in any other type position is **ill-formed** (a Semantic error, not silently ignored — Memory §5.3.1 / MEM-1) *)
              | "brand" "(" type-expr ")"              (* nominal brand over a layout (Type System §5.4) *)
              | fn-type                                (* function-value type: a non-capturing function value = one code-address word (Functions §1.5, FN-10; §3.6) *)
              | "dyn" fn-type                          (* type-erased closure type: `dyn fn(T…)->R` = a {code, env} fat pair, env in explicit storage (Functions §1.6, FN-11; §3.6) *)

struct-type ::= "struct" "{" [ field { item-sep field } ] "}"
enum-type   ::= "enum"   "{" [ variant { item-sep variant } ] "}"
union-type  ::= "union"  "{" [ union-member { item-sep union-member } ] "}"
array-type  ::= "[" type-expr [ "=" expr ] ";" expr "]"   (* [T; N] | [T = v; N]; N comptime (Type System §6.4) *)
tuple-type  ::= "(" [ type-expr { item-sep type-expr } ] ")"

field       ::= { attribute } [ "mut" ] ident ":" type-expr [ "=" expr ]   (* a field takes any attribute (CT-10); target/kind legality is a semantic check, not a syntactic one *)
variant     ::= ident [ "(" [ type-expr { item-sep type-expr } ] ")" ] [ "=" int ]   (* "= N" pins the discriminant; Type System §6.2 *)
union-member ::= ident [ "(" type-expr { item-sep type-expr } ")" ]  (* zero components omit parentheses; 2+ components form one anonymous tuple payload; no discriminant — Type System §6.3 *)
(* `layout-attr` — the closed v1 machine-lever subset (Type System §8 / CT-10); not the whole field-attribute space (see `attribute`, §3.6). A non-lever prelude/library effector is also a valid field attribute, validated semantically. *)
layout-attr ::= "@repr"    "(" type-expr ")"          (* pin an enum tag's underlying type (Type System §8) *)
              | "@packed"
              | "@align"   "(" expr ")"
              | "@offset"  "(" expr ")"
              | "@endian"  "(" expr ")"
              | "@niche"   "(" expr ")"               (* constructive invalid-pattern producer (Type System §6.2) *)
              (* `@abi` is NOT a layout-attr: it is a function-declaration calling-convention attribute (`abi-attr`, §3.6) — v1 has no type-target `@abi` (Type System §8). *)
```

Within a run of prefix type attributes, decorators apply **nearest to `type-atom`
first**. In particular, `@require(p) @require(q) T` is
`require(require(T, q), p)`; the predicates and explicit constructor gates follow that
nested order (Type System §8.1 / TYP-12).

A named aggregate underlying and either predicate spelling require no special
declaration grammar: `R := U.require(pred)` and
`R := U.require(fn(in v : U) -> bool { … })` are ordinary UFCS call expressions whose
results are type values. In a type position, `@require(pred) U` is derived directly by
`type-expr`; `@require(pred) struct { … }` on a literal is also an `attributed-value`
(§3.4). Predicate-source spelling changes callable identity as specified by Type System
§8.1, not the grammar or required call interface.

`::` navigates **namespaces** (modules / types / associated items, and module-valued
bindings; MOD-3); `.` accesses a **value's** member and is the UFCS dot (§3.4). Pointee
mutability sits **inside** the `ptr` constructor; an absent pointer is the ordinary
type `Option(ptr(T))` — there is no null literal (Memory §4.2).

#### 3.4 Expressions

Expressions are layered by precedence (the full table is §4). The layered grammar:

```ebnf
expr        ::= range-expr
range-expr  ::= or-expr [ ( ".." | "..=" ) [ or-expr ] ]   (* half-open [a,b) | inclusive [a,b]; an open-ended `lo..` (upper bound omitted) is the slice-to-end form, valid only in index position `a[lo..]` (Control Flow §5.4) *)
or-expr     ::= and-expr  { "or"  and-expr }            (* short-circuit (OP-2) *)
and-expr    ::= cmp-expr  { "and" cmp-expr }            (* short-circuit (OP-2) *)
cmp-expr    ::= bitor-expr [ cmp-op bitor-expr ]        (* comparisons are non-associative *)
cmp-op      ::= "==" | "!=" | "<" | "<=" | ">" | ">="
bitor-expr  ::= bitxor-expr { "|" bitxor-expr }         (* bitwise, greedy (OP-2) *)
bitxor-expr ::= bitand-expr { "^" bitand-expr }
bitand-expr ::= add-expr    { "&" add-expr }
add-expr    ::= mul-expr    { ( "+" | "-" ) mul-expr }
mul-expr    ::= unary-expr  { ( "*" | "/" | "%" ) unary-expr }
unary-expr  ::= ( "-" | "~" | "not" ) unary-expr
              | postfix-expr
postfix-expr ::= primary { postfix }
postfix     ::= "." ident                               (* named field/member of a struct/raw union (§3.3; Type System §6.1/§6.3) *)
              | "." tuple-index                         (* positional element of a tuple — Type System §6.1; NOT arrays (use `[i]`) or enums (use `match`) *)
              | "." "(" expr ")"                        (* comptime field projection: a.(f) — f a comptime Field/str; ≡ a.<f.name> (TYP-9; Comptime §5.4) *)
              | "." path "(" [ call-args ] ")"          (* UFCS (β): a.f(args) always = f(a,args); a.ns::f(b) ≡ ns::f(a,b) (TYP-5/MOD-3/CT-10) *)
              | "(" [ call-args ] ")"                    (* call *)
              | "[" expr "]"                             (* index — the `index` operator: built-in for arrays/slices (§6.4), or a library type's own `index`/`index_range` (OP-3; Type System §4.5) *)
              | "?"                                       (* try: unwrap-or-propagate (Control Flow §8) *)
primary     ::= literal
              | path | ident
              | "(" expr ")"                              (* grouping *)
              | tuple-ctor | array-ctor | struct-ctor | variant-ctor
              | if-expr | match-expr | loop-expr | block
              | labeled-loop-expr | labeled-block         (* @label(name) loop/block — break/continue target (§3.7) *)
              | attributed-value                          (* optional @-attributes + value-introducer *)
              | comptime-block | comptime-if | comptime-match | comptime-for
              | unchecked-region                          (* verification grant (CG-6) *)
              | alloc-with-region                         (* ambient-allocator scope (MEM-5) *)
              | "uninit"                                  (* explicit no-initializer (Type System §9.4) *)
(* built-in calls (typeinfo/bitcast/forget/asm/instructions, the pointer/layout words ptr/deref/size/align, and the atomic::/volatile:: modules) *)
(* are NOT special grammar: they are prelude identifiers (word-functions) or prelude-module members, reached *)
(* through `ident` or a `path` + a postfix call (§3.8, §3.9, §3.10). *)

(* the value-introducer family — all usable as the RHS of `name := …` (FN-1/FN-5/SYN-6) *)
attributed-value ::= { attribute } value-introducer        (* @abi(c) fn(...), @packed struct{...}, ... *)
value-introducer ::= struct-type | enum-type | union-type   (* type literals (Type System §11) *)
              | fn-value                                  (* a function value (Functions §9); a generic type = fn(…) -> type (SYN-6) *)
              | "mod" "{" module-body "}"                 (* a module value — declarations only (Modules §10) *)
```

Value construction (Type System §9):

```ebnf
tuple-ctor   ::= "(" ")"                                      (* unit value; type `()` *)
               | "(" expr "," [ expr { item-sep expr } ] ")"  (* (1, 2) — the comma disambiguates from grouping *)
array-ctor   ::= "[" expr { item-sep expr } "]"                (* [1, 2, 3] — listed *)
               | "[" expr ";" expr "]"                         (* [v; N] — replicated *)
               | array-type "(" [ expr { item-sep expr } ] ")" (* [T = v; N](…) — defaulted/partly explicit *)
struct-ctor  ::= type-expr "(" [ field-init { item-sep field-init } ] ")"  (* Point(x = 1, y = 2) *)
field-init   ::= ident "=" expr
variant-ctor ::= type-expr "." ident [ "(" [ call-args ] ")" ] (* Color.Red | Option.Some(v) | Cell.pair(a,b); union arity is checked semantically (Type System §6.3) *)

call-args    ::= arg { item-sep arg }
arg          ::= expr                                          (* positional *)
               | ident "=" expr                                (* named — same `=` as struct ctor (Functions §5.2) *)
arg-list     ::= [ expr { item-sep expr } ]                    (* generic-instantiation arguments *)
```

Bit shifts and rotations are **operations**, not glyph operators (OP-2 assigns glyphs
only to `&` `|` `^` `~`); they are written in call or UFCS form (prelude operations) as
`shl` / `shr` / `rotl` / `rotr` (OP-6; Types §2.2).
Arithmetic and bitwise binary operators, and comparisons, are **operator-functions**
(overloadable; TYP-5); `and`/`or`/`not` are short-circuit **control flow**, not functions
(OP-2). **Compound assignment** (`+= -= *= /= %= &= |= ^=`, §3.2) is sugar over those
operator-functions and ordinary assignment (OP-2); its glyph set is exactly the binary
glyph operators, so it adds no operator and no overload point.

`T.V(args)` is a variant/member constructor when `T` resolves to an enum/raw-union type
and `V` names one of its variants/members; this semantic classification resolves the
grammar's intentional overlap between `variant-ctor` and postfix call syntax. A raw-union
member constructor accepts exactly its declared component count as **positional** arguments;
named arguments, `m()` for a zero-component member, and a tuple supplied as one argument
for a multi-component member are Semantic diagnostics (Type System §6.3). Otherwise
`a.f(args)` is **always UFCS** — `f(a, args)` (MOD-3) — even when `f` also names a field.
In particular, `u.m(args)` for a raw-union **value** `u` is UFCS, not a member write; a
whole-member write is `u.m = payload` (Type System §6.3). To call a **function-valued
field**, parenthesize: `(a.f)(args)`. A `.`-postfix without a following call is field/member
access.

`a.(f)` (a `.` immediately followed by `(`, **distinct** from UFCS `a.f(args)` / `a.ns::f(args)`,
which begin with an `ident`/`path`) is **comptime field projection**: `f` is a comptime value
naming a field of `a` — a `Field` (from `typeinfo(T).fields`) or its `str` `name` — and `a.(f)`
≡ the named access `a.<f.name>` (same place/type/permission), resolved at comptime and emitting
identical code (TYP-9; Comptime §5.4). It is the form that lets a structural derive read the field
it is iterating.

#### 3.5 Patterns (Control Flow §5; CF-1)

```ebnf
pattern      ::= "_"                                   (* wildcard *)
               | literal
               | range-pattern
               | variant-pat [ "(" [ pattern-arg { item-sep pattern-arg } ] ")" ]  (* variant / destructure *)
               | ident                                  (* binding *)
variant-pat  ::= path | type-expr "." ident             (* a variant pattern is a `path` (bare `Red` / `Color::Red`) OR the constructor `.`-spelling `Color.Red` (Control Flow §5.2): a pattern may mirror how the variant is built *)
               | type-expr "." "(" expr ")"             (* comptime variant pattern T.(v): the variant of T named by the comptime Variant/str `v`; resolves to T.<v.name>. Inside a `comptime for` over `typeinfo(T).variants` — the enum analog of a.(f) (CT-9; Comptime §5.5). T.(v)(p) binds the whole payload as one value (a tuple when multi-component). *)
pattern-arg  ::= pattern
               | ident "=" pattern                      (* field destructure: Point(x = px, …) *)
range-pattern ::= literal ".." literal | literal "..=" literal   (* scalar ranges (Control Flow §5.4) *)
```

A `match` **arm list** MAY contain a `comptime for x in <comptime collection> { <arm> … }` that **unrolls** to one arm
per iteration (the arm-position analog of statement-position `comptime for`, §8.3; CT-9) — the form that makes a `match`
over a **generic enum** exhaustive in a structural derive (Comptime §5.5).

#### 3.6 Functions, parameters, ABI (Functions §9; SYN-6/SYN-7)

```ebnf
abi-attr     ::= "@abi" "(" expr ")"                   (* calling convention on a function: an `Abi` value, or the `naked` / `entry` sentinel (Functions §6; ABI appendix §3.1) *)
fn-value     ::= fn-sig block                          (* a function value = signature + body *)
fn-sig       ::= "fn" "(" [ param-list ] ")" [ "->" type-expr ] [ when-clause ]  (* -> T = single anonymous projected output (Functions §3.1) *)
(* the function-value TYPE (a `type-atom`, §3.3; Functions §1.5, FN-10): bodyless signature in TYPE position,
   parameter types with optional directions, NO names, NO block. `fn(` in a type position is always `fn-type`
   (`fn` is a keyword, never a generic instantiation `ident(...)`). *)
fn-type      ::= "fn" "(" [ fn-type-param { item-sep fn-type-param } ] ")" [ "->" type-expr ]
fn-type-param ::= [ "in" | "out" | "in" "out" ] type-expr    (* direction (default in) + type — no name *)
(* the type-erased dyn closure TYPE (a `type-atom`, §3.3; Functions §1.6, FN-11): `dyn` over a fn-type =
   a {code, env} fat pair. Constructed by the prelude `dyn_over(ptr(mut store))` (a builtin, no grammar);
   called `d(args)` via the ordinary postfix call (§3.4). *)
dyn-type     ::= "dyn" fn-type
param-list   ::= variadic-param
               | param { item-sep param } [ item-sep variadic-param ]   (* at most one variadic, last; may be the sole parameter (Functions §7) *)
param        ::= in-param | place-param
in-param     ::= [ "in" ] [ "comptime" ] ident ":" type-expr [ "=" expr ]  (* default allowed (Functions §5.1) *)
place-param  ::= ( "out" | "in" "out" ) ident ":" type-expr                (* named out / in-place; no default; no comptime *)
variadic-param ::= [ "in" ] ident ":" "..." [ type-expr ]    (* trailing-rest (Functions §7): `...T` = homogeneous slice `[T]`; `...` (no type) = comptime tuple, or — under `@abi(c)` — a C variadic *)

(* a generic type is an ordinary fn-value returning `type`: Vec := fn(T : type) -> type { … } (SYN-6) — no separate production *)

return-stmt  ::= "return" [ expr ]                     (* expr only with a single anonymous projected output `-> T` (Functions §3.4) *)
```

A `T : type` parameter is comptime by nature — no `comptime` marker (Comptime §1.3). The result uses one
output-place model and is declared **either** as a single **anonymous projected output** `-> T` after the
parameter list, **or** as one or more **named** `out r : T` parameters — **never both** (`->` excludes
named `out`-results; it composes with `in` / `in out`). With `->`, the parameter list is **uniform** —
every entry is `name : type`. There is **no** anonymous `out T` form.

#### 3.7 Control flow (Control Flow §11; CF-1)

```ebnf
if-expr       ::= "if" expr block [ "else" ( if-expr | block ) ]
match-expr    ::= "match" expr "{" match-arm { sep match-arm } "}"
match-arm     ::= pattern { "|" pattern } "=>" ( expr | block )   (* OR-pattern: alternatives share one body (Control Flow §5.4) *)
loop-expr     ::= "loop" block                          (* + break value → loop is an expression *)
while-stmt    ::= "while" expr block
for-stmt      ::= "for" ident "in" expr block
labeled-loop-expr ::= label-attr loop-expr              (* value-bearing structured label; break/continue target (CF §7.1; CF §11); CF-5 *)
labeled-loop-stmt ::= label-attr ( while-stmt | for-stmt )  (* statement-only structured label; break/continue target (CF §7.1; CF §11); CF-5 *)
labeled-block ::= label-attr block                      (* break target only (CF §7.1; CF §11); CF-5 *)
labeled-point ::= label-attr postfix-expr               (* postfix-expr must be an instruction/intrinsic call — no `instruction` production (§3.10/CF §11); CF-5 *)
break-stmt    ::= "break" [ struct-label ] [ expr ]     (* struct-label resolved semantically (CF §7.1) *)
continue-stmt ::= "continue" [ struct-label ]           (* struct-label must be a loop (CF §7.1) *)
struct-label  ::= ident                                 (* must resolve to an in-scope structured label *)

defer-stmt    ::= "defer" ( expr | block )              (* LIFO; runs on all normal exits (Memory §5.8) *)
unchecked-region ::= "unchecked" ( expr | block )       (* verification mode; sibling of comptime{} (CG-6/CG-7) *)
alloc-with-region ::= "alloc" "::" "with" "(" expr ")" block  (* ambient-allocator scope: the place arg is the ambient allocator within `block` (MEM-5; Memory §5.2.1) *)
```

`unchecked ( expr | block )` is the **verification mode** (CG-7): within the scope the
**checked-guard family is dropped** — checkable operations take their hardware-defined
behavior (arithmetic wraps, indexing skips bounds, narrowing truncates, float→int and
div0/shift/alignment faults pass through; never UB, I11) — **and** the genuinely-raw
operations (pointer arithmetic, int↔ptr, raw transfer, raw-union reads whose member is not
definitely active, `uninit` reads, raw `asm`) become writeable; as an expression it yields
the inner value. `@` never wraps an
expression (CG-6).

Conditions are **unparenthesized**; bodies are always braced. `if` and `match` are
**expressions** (CF-2). A labeled `loop` remains value-bearing; labeled `while`/`for`
are statement-only, like their unlabeled forms. A leading identifier on `break`/`continue` is a `struct-label`
**only** if it resolves to an in-scope structured label; otherwise it is part of `expr`
(CF §7.1).

#### 3.8 Comptime and generics (Comptime §10; CT-8)

```ebnf
(* form notes — not distinct productions; each derives from a general rule:
     comptime binding  `comptime PI : f64 = 3.14`  = binding (§3.2) with the `comptime` modifier
     comptime param    `comptime N : usize`          = in-param (§3.6) with the `comptime` modifier
     type-param        `T : type`                  = in-param (§3.6) — comptime by nature *)

comptime-block   ::= "comptime" block
comptime-if      ::= "comptime" "if" expr block [ "else" ( comptime-if | block ) ]
comptime-match   ::= "comptime" match-expr                 (* multi-way comptime branch selection: comptime-selects the first matching arm over a comptime scrutinee, emits only it; arm bindings are comptime values (Comptime §8.2) *)
comptime-for     ::= "comptime" "for" ident "in" expr block

when-clause      ::= "when" expr                        (* declaration guard: between signature and body *)
```

Generic instantiation is **ordinary call syntax** (`Vec(u8)`, `max(a, b)`), not angle
brackets. Built-ins (`typeinfo`, `resolves`, `compiles`, `bitcast`, the pointer/layout
word-functions `ptr`/`deref`/`size`/`align`, `forget`, `panic`, …) are **prelude
identifiers**, not keywords and not `@`-marked
(Stdlib §3; catalog in Stdlib appendix §4; OP-1/SYN-7); they therefore have **no special
grammar** — a call to
one is an ordinary `ident "(" … ")"` derived through `postfix-expr` (§3.4). The named
forms in §3.9 (`ptr`/`deref`, `forget`) and §3.10 (`asm`, instructions) are *instances* of
that same shape, shown there only to document each one's meaning and operands, not to
add syntax.

#### 3.9 Memory (Memory §7; MEM-7)

The memory builtins add **no productions** — each is a call to a prelude identifier
derived through `postfix-expr` (§3.4). The following are **form schemas** (operand
shapes and meaning), not grammar rules:

```text
ptr( place )            — takes an address; ≡ place.ptr()
deref( expr )           — load, or a place when assigned; ≡ expr.deref()
forget( expr )          — discharge linearity without release
```

These pointer ops are data-only (Memory §7). A **code** address (a function or a
code-point label) is the entity's name in value position — its pointer-width address —
**not** a `ptr`/`deref` form (CF-5/MEM-7).

#### 3.10 Assembly correspondence (Assembly §11; CG-2)

Likewise, instructions and the raw escape add **no productions**: an instruction is a
call to a per-arch intrinsic, and `asm` is a prelude-identifier call while `at` is on the
arch surface (qualified `<arch>::at(…)` outside an arch gate) — all
derived through `postfix-expr` (§3.4). Arch access is just a `path` (§3.3). The
following are **form schemas**, not grammar rules:

```text
instr-name( arg-list )            — instruction, destination-first: movq(rax, 60)
expr . instr-name( arg-list )     — instruction, UFCS form: rax.movq(60)
at( mem-field, … )                    — addressing operand (arch surface; <arch>::at(…) when out of an arch gate); mem-field = base|index|scale|disp|offset = expr
expr . ident( arg-list )          — operand decorator (SIMD lanes/masks, reloc/TLS): v0.lanes(f32, 4); sym.plt()
asm( string, expr, … )            — raw GAS escape; requires an unchecked grant
```

Instructions are per-arch **intrinsics** in call form (destination-first) or UFCS; the
memory operand is the named-argument arch-surface builtin `at(…)` (no `[ ]` addressing grammar);
the raw GAS escape `asm(…)` uses positional `{i}` substitution validated only by `as`.

#### 3.11 Attributes (SYN-7)

```ebnf
attribute    ::= "@" attr-name [ "(" [ arg-list ] ")" ]  (* @name | @name(args) *)
attr-name    ::= ident | "abi"                          (* `abi` stays reserved as a bare word (§3.1); it is admitted only here, after `@`, as the fixed attribute name `@abi` (Functions §6) *)
```

An attribute `@name(args)` is **a comptime function applied to the construct it prefixes**
(a binding, field, type, function, file, or code construct; TYP-7/SYN-7/CT-10): `@name X` ≡
`X.name(args)`. A **configuration** attribute composes the **closed core lever set** (CT-10) — the 6
layout/representation levers (`repr`/`align`/`packed`/`offset`/`endian`/`niche`).
A **contract** attribute (`@require`) is a **comptime-function attribute** defining the
explicit checked constructor gate of Type System §8.1, **not** a lever or an `assert`/
`panic` call.
(Branding is **not** an attribute: it is the ordinary builtin call `brand(T)` — a
type-constructor, like `bitcast` — Type System §5.4 / Declarations §6.)
**Storage, linkage, ABI, and control** attributes (`@reg`/`@static`/`@extern`/`@export`/
`@abi`/`@limits`/`@label`) are the other fixed attribute categories (TYP-7/SYN-7). Either way,
prelude attributes (`@align`/`@niche`/…) and library ones (`@nonzero`/…) are
**indistinguishable** in form. `@`
is compile-time and decorates a *construct* — it **never wraps an expression** (CG-6);
runtime operation effects are ordinary builtin calls (`atomic::load`, §3.4; CC-2), not `@`.
The v1 **prelude-provided** effectors (all prefix; library ones extend the open set):

- **storage / lifetime:** `@reg` / `@reg(rax)` / `@stack` / `@static` /
  `@section("…")` / `@scoped` / `@alloc(allocator)`;
- **layout:** `@repr(T)` (pin an enum tag's underlying type, Type System §8) /
  `@align(N)` / `@packed` / `@offset(N)` / `@endian(big|little)` / `@niche(producer)`;
- **ABI / contract / conversion:** `@abi(value)` / `@limits(…)` / `@require(pred)`
  (validity-construction gate, Type System §8.1) / `@convert` (conversion-constructor for
  `T(v)`, Type System §4.6 / TYP-6);
- **linkage / symbol:** `@extern` / `@extern("name")` (import an external symbol, no
  body) and `@export` / `@export("name")` (emit a symbol; auto `__`-name or exact);
- **control / code:** `@label(name)` — names a code point (a `jmp` target; the name in
  value position is a raw code address) **or** a loop/block (a `break`/`continue`
  target — not a value, no address) (CF-5).

Built-ins are **not** `@`-marked; `@` means exactly one thing (an attribute), never an
intrinsic (SYN-7).

```ebnf
extern-decl  ::= ident ":=" "@extern" [ "(" string ")" ] { attribute } fn-sig
               (* printf := @extern @abi(c) fn(in fmt : str, args : ...) -> i32 — body-less (Modules §7) *)
(* @export is an ordinary attribute (§3.11); form: "@export" [ "(" string ")" ] — force symbol / exact name (Modules §6) *)
label-attr   ::= "@label" "(" ident ")"                  (* precedes the instruction / loop / block it names *)
```

---

### 4. Operator precedence and associativity

From **tightest** (binds first) to **loosest**. Within a level, associativity is as
shown. Comparisons are **non-associative** (`a < b < c` is ill-formed; use explicit
grouping). Parentheses always override (SYN-5).

| Level | Operators / forms                                  | Assoc.          |
|------:|----------------------------------------------------|-----------------|
| 1     | postfix: call `()`, index `[]`, field `.`, UFCS `.f()`, try `?` | left |
| 2     | unary prefix: `-`, `~`, `not`                      | right           |
| 3     | `*`  `/`  `%`                                       | left            |
| 4     | `+`  `-`                                            | left            |
| 5     | `&`                                                 | left            |
| 6     | `^`                                                 | left            |
| 7     | `\|`                                                 | left            |
| 8     | comparisons `==` `!=` `<` `<=` `>` `>=`             | non-associative |
| 9     | `and`                                               | left            |
| 10    | `or`                                                | left            |
| 11    | range `..`  `..=`                                   | non-associative |

Notes:
- Bit shifts and rotations are **operations in call/UFCS form**, not glyph operators,
  so they do not appear in the table (OP-2); write them explicitly when mixing with
  arithmetic to make intent and cost visible.
- `not` is the unary logical operator; `and`/`or` are the binary short-circuit
  operators (keywords; OP-2). The bitwise family `& | ^ ~` is greedy (OP-2).
- Assignment `place = expr` is a **statement**, not an expression (SYN-5); it has no
  precedence level and does not nest inside expressions.

---

### 5. Conformance

A conforming implementation MUST:

1. **lex** Alatyr per §2 — UTF-8 source; **case-sensitive, underscore-significant**
   identifiers compared by code points as written (§2.2; SYN-1); the
   keyword set of §2.3 (with `async`/`await` reserved, `in`/`out` contextual); the
   literal forms of §2.4 (with `_` as a non-significant separator); line-only `#`/`##`
   comments with adjacent-line merging and doc-capture (§2.5);
2. apply the **separator and line-continuation** rules of §2.6 deterministically —
   newline-or-`;` between statements/`match` arms, comma-or-newline between sequence
   elements, free newlines inside `( … )` / `[ … ]`, and the incomplete-line
   continuation rule (SYN-4);
3. **parse** the grammar of §3, including the optional file-level `@limits(...)`
   attribute, and the two distinct separator contexts (`sep` vs
   `item-sep`), accepting exactly the programs these productions admit and rejecting
   others with a Parse-stage diagnostic (Tooling §5); since `module-body` admits only
   `module-item` (a `declaration` or the top-level-only `@test`; §3.1), an executable
   statement at **module scope** is rejected at the **Parse stage** (§3.1); and compute
   a **block's value** as its final element only
   when that element is an
   expression, with trailing separators collapsed and no unit type (§3.1; Control Flow
   §3);
4. resolve **operator precedence and associativity** per §4, treating comparisons and
   ranges as non-associative and assignment as a statement;
5. keep this chapter and every feature chapter's *Syntax* section **in agreement**: a
   construct admitted here MUST be admitted by its feature chapter and conversely;
   where they differ, the feature chapter's prose governs and the divergence is a
   defect.

Where a production here is a *reference* to a feature chapter (e.g. the precise
binding rules of `out` parameters, or the semantic resolution of a `struct-label`),
the feature chapter is normative for the meaning; this chapter is normative for the
**form**.
