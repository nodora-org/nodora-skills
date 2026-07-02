---
name: nodora
description: Write, compile, and evaluate Nodora rulesets. Nodora is a declarative rule engine. Use this skill when authoring `.ruleset` files, debugging Nodora compile or evaluation errors, or running the `nodora` CLI.
---

# Nodora

Nodora is a small declarative language for **business rules**. A ruleset is
compiled once and then evaluated against a JSON `input` object. Evaluation produces
two things: an **outputs** map (what is true about the input) and an ordered
list of **emitted signals** (what should happen next).

Mental model: a rule is a pure function `input -> { outputs, emitted_signals }`.
There are no loops, no mutation, no I/O, and no side effects inside expressions.
Everything is type-checked at compile time.

## Anatomy of a file

A source file (conventionally `*.ruleset`) is a flat list of top-level
declarations. Three kinds exist: **signals**, **constants**, and **rules**.
Line comments start with `//`. Whitespace is insignificant.

```text
// a signal: an external event a rule can emit
signal BlockAccount(user_id)

// a constant: a compile-time value shared by all rules below it
const MIN_AGE = 18

// a rule: named block evaluated against `input`
rule Approve {
    is_adult = input.age >= MIN_AGE          // local binding
    emit BlockAccount(input.user_id) when !is_adult
    out approved = is_adult                   // exposed output
}
```

**Top-down rule:** a declaration may only reference names declared _above_ it.
Forward references are a compile-time error. Order your file: signals/consts
first, then the rules that use them.

## Declarations

### `signal`

Declares an external event with a fixed positional parameter list. Parameters
document arity only (not types). A signal may be declared once. Signals are
never called directly, they are queued by `emit`.

```text
signal NotifyUser(user_id, message)
signal Audit()
```

### `const`

A top-level named value computed at compile time. Its expression must be a
**constant expression**: literals, operators, conditionals, arrays/objects,
field/index access, references to constants declared above, and calls to
**pure** built-ins. It may **not** read `input`, call an impure function whose
result varies at runtime (like `time::now()`), or hold a lambda or
`match` expression.

```text
const MAX_SCORE = 100
const LIMIT     = MAX_SCORE * 2          // refers to an earlier const
const REGIONS   = ["us", "eu"]
const ABS5      = math::abs(-5)          // pure call is allowed

const BAD = input.score                  // ERROR: reads input
```

Constants are folded into each use at compile time and cost nothing at runtime;
unused constants are dropped.

### `rule`

A named block of statements run top-to-bottom. Each rule has its own scope and
receives the implicit `input` object. Statements are only **assignments** and
**emits**.

```text
rule Name {
    // statements
}
```

## Statements

### Assignment (local binding)

`name = expr` introduces an immutable local, visible to later statements.
Each name is bound **exactly once**: reassignment is a compile-time error.

```text
total = input.qty * input.price
flag  = total > 1000
```

### Output

`out name = expr` is an assignment that is _also_ placed in the result
`outputs` map under that name. Output values that evaluate to `undefined` are
omitted from the result. A rule with no outputs is valid (useful for signals).

```text
out approved = is_adult
```

`out name = expr` already binds `name`, so write the value straight into the
output instead of binding a local and re-exposing it under the same name (that
re-binds the name and is a compile error). This is the same single-assignment
rule as `=`; an `out` is just a `=` that is also exported, and any later
statement can read the name it binds.

```text
out total = expr        // correct

total = expr
out total = total       // WRONG: re-binds `total`
```

### Emit

`emit Signal(args...)` queues a signal. The optional `when <bool>` clause gates
it; if the condition is `false` or `undefined`, nothing is emitted. Argument
count must match the signal's declaration. Emissions are dispatched after the
rule finishes, in source order.

```text
emit BlockAccount(input.user_id) when !eligible
emit Audit(input.user_id)                          // always fires
```

## The `input` object and `undefined`

`input` is a reserved, read-only object of type `object` provided at evaluation
time. Read fields with dot or bracket access:

```text
name = input.user.name
role = input["user"]["role"]
tag0 = input.tags[0]
```

Reading a missing field yields the special value `undefined`. `undefined`
**propagates** through almost everything (arithmetic, comparison, field access,
most built-ins) instead of raising an error, and `undefined` outputs are
omitted. So partial inputs are safe. Test explicitly with `is_defined(x)`.
(Exception: `&&`/`||` use Kleene logic — see Operators.)

## Types

Nodora is statically typed with **no implicit coercion**.

| Type     | Literals                                               |
| -------- | ------------------------------------------------------ |
| `string` | `"hi"`, `""` (escapes `\n \t \r \" \\`)                |
| `number` | `0`, `42`, `3.14` (64-bit float; no separate int type) |
| `bool`   | `true`, `false`                                        |
| `null`   | `null` (a defined value; distinct from `undefined`)    |
| `object` | `{ a: 1, "b": "x" }`                                   |
| `array`  | `[1, 2, 3]`                                            |

`any` is the top type (built-in arg slots). Unions appear in signatures as
`A|B`. `array<T>` is the element-typed array form used in signatures.

`null` is an explicit empty (a JSON `null` input reads as `null`; a missing
field reads as `undefined`). A `null` output is serialized; an `undefined`
output is omitted. Compare against `null` with `==`/`!=` regardless of the
other operand's type.

## Operators

```text
arithmetic   + - * / %        (numbers; + also concatenates string+string)
comparison   < <= > >=        (numbers -> bool)
equality     == !=            (same-type, structural for arrays/objects)
logical      && || !          (bools -> bool)
membership   x in array       (-> bool; right side must be an array)
access       a.b   a["b"]   a[i]   f(...)   ns::f(...)
```

Key constraints (each is a compile error if violated):

- `+` requires both `number` or both `string`. No mixing.
- `- * / %`, `< <= > >=` require `number`. `/` and `% 0` yield `undefined`.
- `&& || !` require `bool`. `&&`/`||` are three-valued (Kleene): `false && x`
  is `false` and `true || x` is `true` even when `x` is `undefined`; otherwise
  an `undefined` operand gives `undefined`.
- `==`/`!=` require operands of the same type (except against `null`, which is
  comparable to any type); compare structurally.
- `in` requires an array on the right.

Precedence (high→low): access/call → unary `! -` → `* / %` → `+ -` →
`< <= > >=` → `in` → `== !=` → `&&` → `||` → conditional. Use parentheses to
disambiguate.

### Conditionals

Two equivalent ternary forms; the condition must be `bool`, both branches
type-checked to a common type:

```text
limit = if input.plan == "free" then 100 else 1000
limit = input.plan == "free" ? 100 : 1000
```

### `match`

Compares a scrutinee against patterns; first matching arm wins. Must be
**exhaustive** (provide `_` or a bare identifier catch-all). Arms are
`pattern => expr`, comma-separated, trailing comma allowed.

```text
out limit = match input.plan {
    "free" => 100,
    "pro"  => 1000,
    _      => 500,            // catch-all required for exhaustiveness
}
```

Patterns: number/string/bool/`null` literal (equality), identifier (binds the
value), `_` (wildcard). A `null` pattern matches any scrutinee type. An arm may
add a `when <bool>` guard; guarded arms do **not** count toward exhaustiveness.

`_` covers every **defined** value but not `undefined`: an undefined scrutinee
makes the whole `match` `undefined` (the catch-all is not reached). Make the
scrutinee defined if you need to handle the missing case.

```text
out tier = match input.requests {
    n when n > 1000 => "critical",
    n when n > 100  => "elevated",
    _               => "normal",
}
```

### Lambdas

Written `|params| body`. Passed to higher-order built-ins. Lambdas are values
but cannot be an output or come from `input`.

```text
has_pos = some(input.numbers, |x| x > 0)
groups  = arrays::group_by(input.items, |it| it.category)
```

## Built-in functions

Call **core** functions by bare name, **namespaced** functions with `ns::name`.
Compact catalog below. To inspect the full signature of a function — argument
names, types, which are optional, and the return type — list its namespace with
`nodora registry <ns>` — this prints every function in that namespace with its
full signature and per-argument docs (e.g. `nodora registry math` lists all the
`math` functions and their signatures). `nodora registry` with no argument lists
every namespace.

- **core:** `int`, `len`, `is_defined`, `is_empty`, `is_number`, `is_string`,
  `is_bool`, `is_array`, `is_object`, `sprintf`, `some`, `every`, `min`, `max`,
  `sum`, `product`, `fallback`
- **arrays:** `concat`, `filter`, `find_index`, `flatten`, `group_by`, `map`,
  `reduce`, `reverse`, `slice`, `unique`, `zip`
- **strings:** `concat`, `contains`, `starts_with`, `ends_with`, `lower`, `upper`,
  `trim`, `trim_start`, `trim_end`, `split`, `substring`, `replace`, `is_alpha`,
  `is_alnum`, `min_length`, `max_length`
- **math:** `abs`, `ceil`, `clamp`, `floor`, `min`, `max`, `pow`, `round`, `sqrt`
- **objects:** `filter`, `get`, `is_subset`, `keys`, `remove`, `union`, `values`
- **json:** `is_valid`, `marshal`, `unmarshal`
- **conv:** `atoi`, `atof`, `to_string`
- **time:** `now` (only impure built-in — ms since epoch, UTC),
  `add`, `sub`, `add_units`, `sub_units`, `start_of`, `end_of`,
  `parse_rfc3339`, `format_rfc3339`
- **crypto:** `sha1`, `sha256`, `hmac_sha256`
- **base64 / base64url:** `encode`, `decode`, `is_valid`
- **semver:** `is_valid`, `canonical`, `compare`
- **uuid:** `is_valid`
- **glob:** `matches`

There is no implicit conversion: use `conv::atoi`/`conv::atof` to parse strings,
`conv::to_string` / `sprintf` / `json::marshal` to produce strings.

## Rules to follow when writing a ruleset (checklist)

1. Declare signals/consts before the rules that use them; never forward-reference.
2. Bind each local once; never reassign a name within a rule.
3. Keep operand types consistent — no string/number mixing; comparisons need
   numbers; logic needs bools.
4. Make every `match` exhaustive (`_` or bare-identifier arm).
5. `emit` argument count must equal the signal's declared parameter count.
6. Constants must be constant expressions (no `input`, no impure calls, no
   lambda/`match`).
7. Prefer letting `undefined` propagate over defensive guards; outputs that are
   `undefined` simply vanish from the result.
8. `out` exposes a value; plain `=` keeps it local.

## Complete example

```text
signal BlockAccount(user_id)
signal NotifyUser(user_id, message)

const ALLOWED = ["us", "ca"]

rule AccountApproval {
    is_adult        = input.age >= 18
    allowed_country = input.country in ALLOWED
    eligible        = is_adult && allowed_country

    emit BlockAccount(input.user_id) when !eligible
    emit NotifyUser(input.user_id, "Welcome!") when eligible && !input.email_verified

    out approved = eligible
    out tier = match input.requests {
        n when n > 1000 => "critical",
        n when n > 100  => "elevated",
        _               => "normal",
    }
}
```

## CLI

Install with the following command:

- Linux / macOS: `curl -fsSL https://nodora.org/install.sh | bash`
- Windows: `irm https://nodora.org/install.ps1 | iex`

Verified against nodora `v0.2.1` (check yours with `nodora version`). 
The commands and flags may differ on other releases — run `nodora help` to confirm.

### compile

```sh
nodora compile -f rule.ruleset [-o out.json]
```

Parses, type-checks, and compiles the ruleset (defaults to the source path with
a `.json` extension). Prints positioned errors and exits non-zero on failure.

### eval / run

```sh
nodora eval -f rule.json [-r RuleName] (-i input.json | --stdin) [--debug] [-e "Signal=cmd {1}"]
```

- `-r/--rule`: rule name (optional if the file has exactly one rule)
- `-i/--input-file` or `--stdin`: the JSON input object
- `--debug`: print slot/op/emission traces
- `-e/--exec "Signal=cmd"`: run a shell command per emission; `{1}`, `{2}`, …
  are replaced by the signal's arguments

```sh
echo '{"user_id":"u42","age":17,"country":"us","requests":50,"email_verified":true}' \
  | nodora eval -f AccountApproval.json --stdin -e "BlockAccount=echo blocked {1}"
```

Result shape:

```json
{
  "outputs": { "approved": false, "tier": "normal" },
  "emitted_signals": [{ "name": "BlockAccount", "args": ["u42"] }]
}
```

### registry / version

```sh
nodora registry              # list namespaces
nodora registry math         # describe a namespace's functions
nodora version
```

## Common errors and how to fix them

- **`undefined symbol 'X'`** — referenced before declaration, or a rule-local
  used in another rule / a constant. Move the declaration above the use; remember
  each rule has its own scope.
- **`operator '+' cannot be applied to ...`** — mixed string/number, or comparing
  across types. Convert explicitly (`conv::atoi`, `sprintf`).
- **`match expression is not exhaustive`** — add a `_` or bare-identifier arm.
- **`const '...' must be a constant expression`** — the const reads `input`, calls
  an impure function, or holds a lambda/`match`. Move that logic into a rule.
- **`symbol '...' already declared`** — duplicate top-level name or a re-bound
  local. Rename.
- **`signal '...' expects N arguments`** — fix the `emit` arity to match the
  `signal` declaration.
