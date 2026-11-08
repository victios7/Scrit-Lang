# Scrit — Language Reference & Agent Engineering Guide

**Purpose:** normative reference for AI agents that generate, edit, debug, optimize, or export Scrit projects in the Scrit IDE.

Scrit is a Python-like, indentation-based language that compiles into **Scratch 3 block graphs**. It is not Python: the syntax is Python-inspired, but runtime behavior, libraries, data storage, control flow, and function lowering are defined by this compiler.

| Item | Contract |
|---|---|
| Source extension | `.scrit` |
| One sprite target | `.sprite3` |
| Two or more sprite targets | `.sb3` |
| Runtime target | Scratch 3 / TurboWarp-compatible project format |
| Function implementation | Native Scratch custom procedures (`procedures_definition`, `procedures_prototype`, `procedures_call`) |
| Procedure execution mode | `warp: "true"` (run without screen refresh) |
| Optimizer IR | Scratch blocks → SIR → optimization passes → Scratch blocks |

This document distinguishes **implemented behavior**, **safe authoring rules**, and **known limitations**. Agents must not invent APIs or assume ordinary Python semantics where this guide says otherwise.

---

## 0. Agent contract — read before generating code

1. Generate valid Scrit, not Python, JavaScript, or raw Scratch JSON.
2. Use the built-in short API names shown here. Scratch opcodes belong to the compiler output, not user source.
3. Every sprite source must begin with a valid `properties:` section. A library source must not contain one.
4. Treat every `def` as a real custom procedure. **Do not inline it, duplicate its body at call sites, or rewrite it as a macro.**
5. Use `macro` only when compile-time source expansion is specifically wanted.
6. Use `return` only inside a `def`. A return stores the result and stops the remainder of that invocation through a generated guard; it is not a native Scratch opcode.
7. Function arguments are read-only. Do not assign to an argument name.
8. Scrit variables declared inside a function body are still Scratch target variables. They are not automatically private stack locals. Use unique names for scratch temporaries and avoid recursive/re-entrant functions that mutate shared working variables.
9. Keep reporters and commands distinct: a reporter is used as a value; a command is used as a statement.
10. List indexes are **1-based**. Pointer heap positions and LBA absolute block numbers are also 1-based; LBA extent offsets start at zero.
11. Do not split assignment lines at `=` while inside quoted strings. The compiler tokenizer handles quoted strings; agent-side source rewriting must do the same.
12. Preserve side effects. Never remove or reorder drawing, sound, sensing, list writes, pointer writes, LBA writes, broadcasts, waits, or procedure calls merely because their result is unused.
13. Compile the current contents of the IDE file system. Do not use a stale source snapshot or assume a template is the user's active program.
14. An optimization is valid only if it preserves source behavior and procedure/control-flow structure. If unsure, prefer `-O0` over an unsafe transformation.
15. Report compiler diagnostics by their **E-code**, file, and line. Do not silently repair a semantic error by changing the user's intent.
16. Do not claim that a `.sprite3`/`.sb3` has been runtime-tested merely because its JSON or ZIP was generated successfully.

### Safe default workflow

For a new or edited project:

1. Inspect all sprite and library sources in the project.
2. Check `properties:`, imports, duplicate function/macro names, function arity, and library API spellings.
3. Compile at **O0** first (`optimize -O0`, then `compile`). Resolve all errors before optimizing.
4. Test function behavior explicitly, especially early returns, nested calls, argument handling, and functions used in expressions.
5. Run one optimization level at a time (`-O1`, then `-O2`, then `-O3`). Compare behavior and diagnostics after each level.
6. Export `.sprite3` for one sprite source or `.sb3` for multiple sprite sources.
7. For export failures, capture the exact error, optimizer level, file list, and block counts. Do not keep increasing optimization when export is already failing.

---

## 1. Language design and compilation model

### 1.1 Core language

The core language is available without importing a library:

- Variables and assignments.
- Numeric and text literals.
- Arithmetic, comparison, and Boolean expressions.
- Strings and lists.
- `if`, `else`, `forever`, `repeat`, `repeat until`, `while`, and `for ... in range(...)`.
- Scratch event hats (`when ...:`).
- `def` custom procedures and `return`.
- `macro` compile-time expansion.
- `str(...)`, `number(...)`, `len(...)`.
- Type annotations for supported variable types.
- `@local`, `@cloud`, and `$` indirection.
- Pointer (`ptr`) and logical-block disk (`lba`) operations through their libraries.

### 1.2 Built-in libraries

Scratch functionality is exposed through `@import`. The currently registered library families are:

`motion`, `looks`, `sound`, `sensing`, `pen`, `music`, `ptr`, `stack`, `lba`, `math`, `events`, `control`, and `lists`.

`str(expr)` is a core intrinsic; it does not need an import. The compiler also recognizes string helpers in expression form, but agents should prefer the documented core intrinsic and avoid treating `str` as a required library.

### 1.3 Compilation stages

The effective pipeline is:

1. **Project scan:** classify each file as sprite or library.
2. **Source preprocessing:** expand explicit `macro` declarations; collect and remove top-level `def` bodies from the main statement stream.
3. **Parse/code generation:** create Scratch blocks, variables, lists, procedure metadata, events, and control flow.
4. **Link/import handling:** resolve built-in aliases, project-file imports, and remote cache entries.
5. **Optimization (optional):** blocks → SIR → selected passes → blocks.
6. **Graph normalization:** remove dangling links and break malformed `next`/`parent` cycles defensively.
7. **Export:** write the `.sprite3` or `.sb3` archive and SVG costume assets.

### 1.4 Compiler invariants

- A sprite's `properties:` metadata belongs in the Scratch target fields, not in executable blocks.
- A `def` declaration is represented by a custom procedure definition, even when it has no call sites.
- A procedure call is a side effect for optimization purposes.
- Custom procedure mutation metadata must match between definition and call blocks.
- Scratch inputs use only SB3 tags `1`, `2`, or `3` (see §14).
- All links are block IDs, not nested block objects.

---

## 2. Multi-file project model

### 2.1 File classification

| Kind | Detection | Meaning | Exported as target? |
|---|---|---|---|
| Sprite | Contains a top-level `properties:` header | One Scratch sprite | Yes |
| Library | No `properties:` header | Shared source imported by a sprite | No |

Export selection:

- Exactly **one** sprite source → `.sprite3`.
- **Two or more** sprite sources → `.sb3` containing Stage plus one target per sprite source.
- Library sources are not exported as independent sprites.

The Explorer supports creating, opening, renaming, and deleting files. The active tab is not necessarily the only sprite in the project; a full export scans the project file system.

### 2.2 Import syntax

```scrit
@import motion as move
@import looks as look
@import sensing as sense
@import "util.scrit" as util
@import "https://example.com/lib.scrit" as remote
```

| Import form | Behavior |
|---|---|
| `@import motion as move` | Resolve a registered built-in library. |
| `@import "util.scrit" as util` | Resolve a project file from the IDE file system. Missing file → E013. |
| `@import "https://host/file.scrit" as remote` | Fetch/cache remote source at compile time; it is not added to Explorer. |
| `include "https://host/file.scrit"` in console | Download the source into the project file system as a library. |

A library must not contain `properties:`. The compiler reports E014 if it does. The console `include` workflow strips a `properties:` section when importing a sprite-like file as a library; agents should still generate a clean library file instead of depending on that cleanup.

### 2.3 Imported file functions — important limitation

The current imported-file path is not the same as a main-file native `def`:

- The importer scans library `def` declarations for a `return expression`.
- `alias.function(args)` in an expression substitutes the arguments into that captured expression.
- The importer does **not** compile an imported library function as a Scratch custom procedure with its full statement body.
- Side effects, local statement sequences, early returns, and complex control flow in imported library `def`s are therefore not a supported reusable-library contract.

**Agent rule:** keep imported library functions pure and expression-only, with one clear `return expression`. Put multi-statement behavior in a `def` in the sprite source, or explicitly copy the library's supported source into the sprite only when the user asks for that behavior.

Example library:

```scrit
# math_utils.scrit — no properties:
def square(x):
  return x * x

def clamp01(x):
  if x < 0:
    return 0
  return x
```

The first function is the safe import pattern. The second contains multiple return/control statements; do not assume the imported-file expression loader implements it faithfully. Keep reusable imported functions to one return expression.

### 2.4 Linker and unused functions

The IDE has a source scan intended to identify imported function call sites and a link-time DCE hook. Do not assume it performs a full inter-file call-graph analysis. Main-file `def`s are emitted as custom procedures; do not delete a function merely because a simple textual search did not find its name (indirect or nested calls may be missed by a simplistic scan).

---

## 3. `properties:` — sprite metadata

Every sprite file must have `properties:` at the beginning of the file (no indentation before the header).

### 3.1 Required and optional fields

| Field | Required | Type | Scratch target mapping |
|---|---:|---|---|
| `name` | Yes | non-empty string | Sprite name |
| `size` | Yes | number | Size percentage |
| `position` | Yes | number | Layer order |
| `rotation` | Yes | number | Direction |
| `pos_x` | Yes | number | X position |
| `pos_y` | Yes | number | Y position |
| `show` | Yes | Boolean | Visibility |
| `costume` | No | known theme name | Generated SVG costume key; default `cpu` |
| `strict` | No | Boolean | Enables strict new-variable checks; default false |

### 3.2 Canonical template

```scrit
properties:
  name = "Main"
  size = 100
  position = 1
  rotation = 90
  pos_x = 0
  pos_y = 0
  show = true
  costume = cpu
  strict = false

@import motion as move
@import looks as look

when flag_clicked:
  look.say("Hello from Scrit")
```

### 3.3 Validation rules

- Missing `properties:` → E001.
- Missing required key → E002.
- Invalid field value or unknown costume theme → E021.
- Do not add type annotations to property keys.
- Property rows are not translated into `data_setvariableto` blocks.
- `name` must be a non-empty string; numeric fields must parse as finite numbers; `show` must be `true` or `false`.

### 3.4 Costume themes

The current built-in SVG theme keys are:

`cpu`, `power`, `fpga`, `ram`, `gpu`, `soc`, `bus`, `adc`, `rom`, `net`, `clk`, `io`, `dsp`.

If omitted, the theme defaults to `cpu`. During multi-sprite export, a used theme may be varied/reassigned so sprites do not all receive the same exact theme asset. The theme is generated SVG artwork, not an uploaded bitmap.

---

## 4. Lexical rules, names, and expressions

### 4.1 Indentation and comments

- Suites are indentation-based and end when indentation returns to the parent level.
- A `#` starts a comment outside a quoted string.
- Blank lines are allowed.
- Strings can use single or double quotes.
- Escaped `\n`, `\r`, and `\t` are recognized by the tokenizer; escaped quote/backslash characters are also consumed as escaped characters.
- Do not split tokens or assignments inside strings.

### 4.2 Names

Use simple identifiers matching `[A-Za-z_][A-Za-z0-9_]*`. Avoid punctuation, spaces, and names that collide with imported aliases, built-ins, or procedure names.

### 4.3 Literals

```scrit
count = 12
ratio = 0.25
message = "hello"
ready = true
not_ready = false
items = []
```

Boolean literals compile to Scratch numeric truth values (`true` → 1, `false` → 0).

### 4.4 Operators

| Family | Operators |
|---|---|
| Arithmetic | `+`, `-`, `*`, `/`, `%`, `**` |
| Comparison | `==`, `!=`, `>`, `<`, `>=`, `<=` |
| Boolean | `and`, `or`, `not`, `&&`, `||`, `!` |
| Assignment shorthand | `+=`, `-=`, `*=`, `/=`, `%=` |

Precedence, from lower to higher, is Boolean OR, Boolean AND, comparisons, addition/subtraction, multiplication/division/modulo, exponentiation, then unary operators/primary expressions.

`**` is lowered to Scratch math reporters using logarithm/exponent operations; it is not a dedicated power block. Avoid relying on it for bases outside the valid logarithm domain (zero or negative base) unless the target runtime behavior has been tested.

The compiler maps `>=` and `<=` through inverted less-than/greater-than comparisons, and `!=` through inverted equality.

### 4.5 Truth values

Scratch Boolean reporters are used when available. In general numeric conditions are treated as true when equal to `1` by Scrit's condition wrapper. Prefer explicit comparisons (`x > 0`, `ready == true`) rather than relying on arbitrary nonzero truthiness.

### 4.6 Parentheses and argument splitting

Use parentheses to make nested expressions unambiguous. Commas inside nested `(...)` or `[...]` do not separate outer call arguments.

```scrit
result = combine(a, scale(b, 2), (c + d) * 3)
```

### 4.7 Unsupported assumptions

Do not assume Python features not listed here: f-strings, dictionaries, sets, comprehensions, exceptions, generators, classes, decorators, keyword arguments, default parameters, `break`, `continue`, `elif`, or arbitrary Python modules. The current compiler's event/control grammar is intentionally smaller than Python's.

---

## 5. Variables, types, and storage

### 5.1 Assignment and compound assignment

```scrit
x = 1
x = x + 1
x += 1
x -= 2
x *= 3
x /= 2
x %= 5
```

Compound assignment is lowered to a normal variable assignment whose expression uses the old variable value.

### 5.2 Type annotations

```scrit
count: number = 0
message: string = "ready"
ptr_value: ptr = ptr.null()
```

The source parser directly recognizes declaration annotations for `number`, `string`, and `ptr`. Pointer and LBA operations also set internal type metadata as they allocate/initialize handles.

The diagnostic catalogue mentions `lba` and `lba_ext`, but the direct typed-assignment pattern is narrower than that catalogue hint. Prefer `e = lba.alloc(n)` to create an extent handle rather than manually annotating it.

### 5.3 Strict mode

Set `strict = true` in `properties:` to require an annotation for newly introduced variables (E020). Existing/inferred typed names can be reassigned without repeating the type.

```scrit
properties:
  name = "StrictDemo"
  size = 100
  position = 1
  rotation = 90
  pos_x = 0
  pos_y = 0
  show = true
  strict = true

counter: number = 0
label: string = "init"
```

Type mismatch can produce E011. Invalid type names can produce E035.

### 5.4 `@local` and `@cloud`

```scrit
@local cache: number = 0
@cloud score: number = 0
```

These declaration prefixes are recorded as variable-kind metadata and shown in the IDE memory tracker. They do not introduce lexical function scope. The current exporter writes ordinary Scratch target variable entries; do **not** promise that `@cloud` becomes a synchronized Scratch cloud variable, because the current serialized variable shape does not establish that cloud behavior.

`@local` / `@cloud` are declaration prefixes, not assignment modifiers. Invalid or repeated use can produce E012.

### 5.5 Function variables are not automatic locals

Main-file `def` arguments are Scratch procedure arguments. A variable assigned inside a `def` body, however, is emitted as an ordinary Scratch variable for that sprite. It persists between calls and can be shared by concurrent event scripts. Agents must:

- Prefer procedure arguments for input values.
- Use clearly prefixed names for function-owned working state (for example `__decode_index`).
- Avoid assuming re-entrant safety for mutable working variables.
- Avoid recursion that depends on per-call local variables unless the program explicitly saves/restores those variables itself.

### 5.6 `$` indirection

```scrit
name = "score"
$name = 10       # assign to the variable whose name is stored in name
value = $name    # read the variable whose name is stored in name
```

Behavior:

- If the referenced name variable has a compile-time literal string matching an identifier, the compiler can resolve it statically.
- Otherwise, runtime lookup uses two Scratch lists: `__sk_keys` and `__sk_vals`.
- A runtime read performs a key lookup and retrieves the corresponding value.
- A runtime write updates the key/value map and may write through to statically known variables.
- When any source in the project uses a runtime `$` read, ordinary assignments in the project synchronize the runtime map so reads see current values. This can increase generated block count.
- `$$` is forbidden.
- `$` indirection is forbidden in library files.

**Optimization guidance:** do not introduce runtime `$` reads merely to simulate ordinary variables. Static names are smaller and easier to optimize. Use `$` only when the program genuinely needs a variable selected by a runtime string.

---

## 6. Strings and lists

### 6.1 Core string helpers

```scrit
text_value = "Scrit"
text_length = len(text_value)
copy = str(text_value)
number_value = number(42)
```

The current core expression lowering treats `str(expr)` and `number(expr)` as pass-through reporters rather than full general-purpose parsing/formatting conversions. Scratch may display numeric/text values flexibly, but agents must not assume `number("42")` is a strict parser or `str(42)` performs locale-independent conversion.

`len(expr)` supports string length; when the argument is a known list name, it emits list length.

### 6.2 String library helpers in expressions

The compiler recognizes these expression helpers:

| Form | Meaning |
|---|---|
| `str.length(value)` / `str.lenght(value)` | String length |
| `str.join(a, b)` | Concatenate text |
| `str.letter(index, text)` | Character at 1-based index |
| `str.contains(text, part)` | Text contains substring |

Prefer `len(text)` for ordinary length checks. The `str` helpers are compiler-supported expressions; do not rely on them as statement commands.

### 6.3 Lists

```scrit
scores = []
scores.append(10)
scores.append(25)
scores[1] = 15
first = scores[1]
count = len(scores)
scores.delete(2)
scores.clear()
```

Direct list operations supported by statement lowering:

- `list.append(value)`
- `list.delete(index)`
- `list.clear()` / `list.delete_all()`
- `list.insert(index, value)`
- `list.replace(index, value)`
- `list[index] = value`

The expression path also recognizes list length/size and, in relevant alias-based list forms, `get`/`item`, `contains`, and `index_of`.

Indexes are **1-based**, matching Scratch lists. Do not use zero-based Python indexing.

### 6.4 `lists` library

```scrit
@import lists as inventory

inventory.append("key")
inventory.insert(1, "first")
inventory.replace(1, "updated")
found = inventory.contains("key")
position = inventory.index_of("key")
count = inventory.length()
inventory.delete(1)
inventory.clear()
```

The alias is also the underlying Scratch list name. Use one alias consistently for each logical list.

---

## 7. Macros — compile-time source expansion

Macros are not Scratch procedures. They are expanded before parsing and are appropriate only for simple textual/expression substitutions.

### 7.1 Declaration syntax

The compiler recognizes a colon-indented macro declaration:

```scrit
macro double(x):
  return x * 2
```

It does **not** use the older `macro double(x) = (x * 2)` spelling shown in some earlier notes. Agents must use the colon form.

### 7.2 Expression use

```scrit
macro double(x):
  return x * 2

when flag_clicked:
  result = double(21)
```

An expression macro should have one clear `return expression`. That return expression is substituted into a call site, including supported expression contexts.

### 7.3 Statement use

A macro may expand a bare statement call into its body:

```scrit
macro announce():
  look.say("Starting")
  pen.down()

when flag_clicked:
  announce()
```

Keep statement macros shallow and simple. Nested calls and complex control flow are not a substitute for a native `def`.

### 7.4 Macro restrictions

- Duplicate names in one file → E023.
- Macros are source expansion, not runtime procedures.
- Macro expansion is iterative and bounded; recursive/cyclic macros are unsafe. Avoid them completely. E033 exists in the diagnostic catalogue, but agents must not rely on cycle detection as a substitute for writing an acyclic macro.
- Argument replacement is textual. Use distinct, simple parameter names and parenthesize expression arguments.
- Do not use a macro to implement a reusable stateful routine when `def` is appropriate.

---

## 8. Functions (`def`) — native Scratch custom procedures

This section overrides any old documentation that describes `def` as macro-style inlining. In the current compiler, a top-level `def` is compiled as a **native Scratch custom block**.

### 8.1 Basic syntax

```scrit
def add(a, b):
  return a + b

when flag_clicked:
  total = add(20, 22)
  look.say(total)
```

### 8.2 Scratch block representation

For each main-file `def`, the compiler creates:

- `procedures_definition` — top-level custom-block definition.
- `procedures_prototype` — the custom block's prototype and input declarations.
- `argument_reporter_string_number` — argument reporters used by the body and prototype.
- `procedures_call` — a call site.

Definition/call mutation metadata contains a `proccode`, JSON `argumentids`, JSON `argumentnames`, JSON `argumentdefaults`, and `warp: "true"`.

The `warp` flag activates Scratch's run-without-screen-refresh behavior for the custom procedure. It does not mean the code is compiled to native machine code.

### 8.3 Function calls

A procedure can be called as a statement:

```scrit
reset_machine()
```

A return-valued procedure can be assigned:

```scrit
result = add(4, 5)
```

Calls can also appear inside supported expressions and nested argument lists:

```scrit
def inc(value):
  return value + 1

def add(a, b):
  return a + b

when flag_clicked:
  answer = add(inc(2), add(3, 4))
  if add(inc(1), 2) == 4:
    look.say(answer)
```

### 8.4 Why call results need temporary variables

Scratch custom procedures are command blocks; they do not return a value through a reporter socket. Scrit bridges this mismatch:

1. Save the caller's return/done state in per-function frame lists.
2. Reset the callee's hidden result and completion variables.
3. Emit a real `procedures_call` block.
4. Copy the function result into a temporary variable when the source call is used as an expression.
5. Restore the caller's saved state from the frame lists.

Nested procedure calls in expressions are lowered into procedure-call statements plus generated temporary variables. They are **not** inlined macro text.

### 8.5 `return` behavior

```scrit
def find_value(n):
  if n < 0:
    return -1
  if n == 0:
    return 0
  return n * 2
```

Scratch custom procedures do not have a native return instruction. Scrit therefore generates hidden per-function state:

- A hidden result variable (`__scrit_ret_<function>`).
- A hidden completion flag (`__scrit_done_<function>`).
- Per-function frame lists that save/restore the result and completion flag across nested calls.

When `return expr` executes, the compiler:

1. Evaluates `expr` (or uses zero for a bare `return`).
2. Assigns the result to the hidden return variable.
3. Sets the hidden completion flag to `1`.
4. Uses generated guards so later statements in the same invocation do not execute.
5. Makes function loops stop when the completion flag is raised.

The next statements after a return remain in the block graph but are guarded; they do not execute for that invocation once the return flag is set.

A function that reaches the end without an explicit return has the default result `0` because the call setup initializes its return variable to zero.

### 8.6 Nested and recursive calls

The call lowering saves/restores hidden return and completion state on per-function lists, so a nested call does not overwrite its caller's hidden return state. This frame mechanism is specifically for the function result/return flag; it does **not** make all user variables local or protect arbitrary mutable state.

**Agent rule:** recursion may be expressed as a function call, but only use it when termination is guaranteed and all mutable per-call state is explicit. Prefer iterative loops for large or unbounded recursion. Do not assume function-body assignment variables are isolated per invocation.

### 8.7 Function arguments and scope

- Parameters are exposed through Scratch argument reporters.
- Parameters are read-only. Assignment to a parameter produces E024.
- Wrong number of arguments produces E030.
- Duplicate function name in one file produces E022.
- Nested `def` declarations are not supported; they produce E022.
- Function overloading is not supported. Never define two functions with the same name and different parameter counts.
- Function definitions are collected before the main script is compiled, allowing forward calls to other declared functions in the same sprite file.

### 8.8 Return statements in branches and loops

The compiler prepares function bodies by wrapping statements with a completion-flag guard. The transform is indentation-aware and preserves `if`/`else` structure. Function loops also include completion checks.

Use clear structured returns:

```scrit
def safe_divide(a, b):
  if b == 0:
    return 0
  return a / b
```

Do not put `return` at sprite top level or inside a library expecting native procedure behavior; `return` outside a main-file `def` is E004.

### 8.9 Main-file `def` versus imported library `def`

| Context | Current behavior |
|---|---|
| `def` in a sprite source | Native Scratch custom procedure |
| `def` in an imported `.scrit` library | Imported expression helper only when a `return expression` can be extracted; not a full procedure body |
| `macro` | Compile-time source expansion |

This distinction is essential. Do not document, generate, or test them as interchangeable.

---

## 9. Control flow

All control suites use a trailing colon and indented body.

### 9.1 `if` / `else`

```scrit
if score >= 10:
  look.say("Pass")
else:
  look.say("Try again")
```

An `if` without a condition produces E015. Empty control bodies are suspicious and may produce E026.

### 9.2 `forever`

```scrit
forever:
  move.steps(1)
```

In a function, `forever` is lowered to a repeat-until form that can stop when the function's return flag is set. At top level it maps to Scratch `control_forever`.

### 9.3 `repeat`

```scrit
repeat 10:
  move.steps(5)
```

The repeat count may be an expression or a variable:

```scrit
repeat amount:
  counter += 1
```

### 9.4 `repeat until`

```scrit
repeat until ready == true:
  wait(0.1)
```

### 9.5 `while`

```scrit
while counter < 10:
  counter += 1
```

`while` is lowered into Scratch repeat-until with an inverted condition. A function's completion flag is included so early return can stop the loop.

### 9.6 `for ... in range(...)`

```scrit
for i in range(10):
  look.say(i)

for j in range(2, 8):
  look.say(j)

for k in range(0, 10, 2):
  look.say(k)
```

The parser accepts one to three `range` arguments:

- `range(stop)`
- `range(start, stop)`
- `range(start, stop, step)`

The compiler lowers this to an initialized counter plus a Scratch repeat block and an increment. Prefer positive integer steps and test boundary behavior when exact Python `range` semantics matter. Do not assume complete parity for negative steps, fractional arguments, or unusual bounds.

### 9.7 Wait, broadcast, and stop

```scrit
wait(1)
broadcast("ready")
```

The compiler also recognizes some direct global commands (`say`, `say_for`, `think`, `ask`, `wait_until`, `stop`). For consistent, readable source, prefer library-qualified calls where documented (`look.say`, `sense.ask`, or `events.broadcast`).

---

## 10. Events

Supported event forms in the current compiler include:

```scrit
when flag_clicked:
  look.say("Green flag")

when key_pressed("space"):
  move.steps(10)

when sprite_clicked:
  look.say("Clicked")

when clone_started:
  look.say("Clone ready")

when receive("start"):
  look.show()

when message_received("stop"):
  look.hide()

when backdrop_switches("level2"):
  look.say("Level 2")

when backdrop_changes("level3"):
  look.say("Level 3")

when loudness > 20:
  look.say("Sound detected")
```

`when clicked:` is also accepted as an alias for `when sprite_clicked:`.

Invalid event header → E028. Use quoted event names for key and broadcast/backdrop names.

---

## 11. Built-in libraries — current API reference

Import each library with an alias, then use that alias consistently. Methods are commands unless specifically listed as reporters or documented as expression helpers.

### 11.1 `motion`

```scrit
@import motion as move

move.steps(10)
move.right(15)
move.left(15)
move.goto(0, 0)
move.set_x(10)
move.set_y(20)
move.change_x(5)
move.change_y(-5)
move.point(90)
move.bounce()
move.glide_xy(1, 100, 50)
move.glide_to(1, "_mouse_")
move.goto_mouse()
move.goto_random()
move.point_towards("_mouse_")
move.rotation_style()

x = move.x
y = move.y
direction = move.direction
```

**Methods:** `steps`, `right`, `left`, `goto`, `set_x`, `set_y`, `change_x`, `change_y`, `point`, `bounce`, `glide_xy`, `glide_to`, `goto_mouse`, `goto_random`, `point_towards`, `rotation_style`.

**Reporters:** `x`, `y`, `direction` (no parentheses).

### 11.2 `looks`

```scrit
@import looks as look

look.say("Hello")
look.say_for("Hello", 2)
look.think("Hmm")
look.think_for("Hmm", 2)
look.show()
look.hide()
look.set_size(100)
look.change_size(10)
look.next_costume()
look.costume("costume1")
look.clear_effects()
look.set_effect(50)
look.change_effect(10)
look.next_backdrop()
look.backdrop("backdrop1")
look.front()
look.back()
look.forward_layers(1)
look.backward_layers(1)
size = look.size
```

**Methods:** `say`, `say_for`, `think`, `think_for`, `show`, `hide`, `set_size`, `change_size`, `next_costume`, `costume`, `clear_effects`, `set_effect`, `change_effect`, `next_backdrop`, `backdrop`, `front`, `back`, `forward_layers`, `backward_layers`.

**Reporter:** `size`.

### 11.3 `sound`

```scrit
@import sound as audio

audio.play("Meow")
audio.play_until("Meow")
audio.stop()
audio.set_volume(80)
audio.change_volume(-10)
audio.set_effect(20)
audio.change_effect(5)
audio.clear_effects()
audio.stop_all()
volume = audio.volume
```

**Methods:** `play`, `play_until`, `stop`, `set_volume`, `change_volume`, `set_effect`, `change_effect`, `clear_effects`, `stop_all`.

**Reporter:** `volume`.

Sound menu values are Scratch sound menu names. Ensure the selected sprite actually has the requested sound asset; Scrit does not automatically create arbitrary sound files from a string.

### 11.4 `sensing`

```scrit
@import sensing as sense

sense.ask("What is your name?")
name = sense.answer
time = sense.timer
mx = sense.mouse_x
my = sense.mouse_y
pressed = sense.mouse_down
loud = sense.loudness
# `key` y `touching` están expuestos en la API sensing; para eventos de teclado,
# prefiere `when key_pressed("space"):`. Comprueba en autocompletado la forma
# exacta de los reporters booleanos en la versión del compilador que uses.
sense.reset_timer()
sense.set_drag_mode()
```

**Core reporters:** `answer`, `timer`, `mouse_x`, `mouse_y`, `mouse_down`, `loudness`.

**Additional reporter expressions supported:** `touching_color(color)`, `color_touching(color1, color2)`, `distance(target)`, `current()`, `days_since_2000()`, `username()`.

**Methods:** `ask`, `reset_timer`, `key`, `touching`, `set_drag_mode`.

Reporters are values without parentheses (`sense.answer`). Methods require parentheses (`sense.ask(...)`). Some sensing members have reporter and method-style opcode metadata; prefer the exact form shown in autocomplete and use results as expressions.

### 11.5 `pen`

```scrit
@import pen as pen

pen.clear()
pen.stamp()
pen.down()
pen.up()
pen.color(90)
pen.change_color(10)
pen.set_size(3)
pen.change_size(1)
```

**Methods:** `clear`, `stamp`, `down`, `up`, `color`, `change_color`, `set_size`, `change_size`.

Pen requires the Scratch `pen` extension. It is added to the export extension list when used.

### 11.6 `music`

```scrit
@import music as music

music.note(60, 0.5)
music.drum(1, 0.25)
music.rest(0.5)
music.set_tempo(120)
music.change_tempo(10)
tempo = music.tempo
```

**Methods:** `note`, `drum`, `rest`, `set_tempo`, `change_tempo`.

**Reporter:** `tempo`.

Music requires the Scratch `music` extension. The sprite/project must have compatible extension support in the target runtime.

### 11.7 `math`

```scrit
@import math as math

pi_value = math.pi
r = math.random(1, 10)
a = math.abs(-5)
b = math.floor(2.8)
c = math.ceil(2.1)
d = math.sqrt(9)
e = math.sin(90)
f = math.cos(0)
g = math.tan(45)
h = math.asin(1)
i = math.acos(1)
j = math.atan(1)
k = math.ln(2)
l = math.log(100)
m = math.e(2)
n = math.pow10(2)
o = math.round(2.5)
p = math.round_down(2.8)
q = math.round_up(2.1)
```

**Constant reporter:** `math.pi` (without parentheses).

**Expression functions:** `random(min, max)`, `abs`, `floor`, `ceil`, `ceiling`, `sqrt`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `ln`, `log`, `e`, `pow10`, `round`, `round_down`, `round_up`.

These map to Scratch operator/math reporters. Check the target Scratch math operator semantics and angle units when writing trigonometric code; do not assume a full Python `math` module.

### 11.8 `events`

```scrit
@import events as events

events.broadcast("ready")
events.broadcast_wait("load_done")
events.clone("_myself_")
events.delete_clone()
events.stop()
```

**Methods:** `broadcast`, `broadcast_wait`, `clone`, `delete_clone`, `stop`.

Use `when receive("message"):` (or `when message_received("message"):`) to handle broadcasts.

### 11.9 `control`

```scrit
@import control as control

control.wait(1)
control.stop()
```

The general control structures (`if`, `forever`, `repeat`, `repeat until`, `while`) are written as Scrit syntax, not invoked as ordinary library commands. The imported namespace exposes only the registered direct methods; use Scrit control syntax for suites.

### 11.10 `lists`

See §6.4. The alias names the Scratch list.

### 11.11 `ptr`

See §12. The pointer implementation is a Scratch-list-backed heap.

### 11.12 `stack`

See §13. The stack implementation is a Scratch-list-backed LIFO structure.

### 11.13 `lba`

See §14. The LBA implementation models an extendable logical block store backed by Scratch lists.

---

## 12. `ptr` — list-backed heap and pointer handles

`ptr` models a simple heap using a Scratch list. It is not a native memory address and does not expose arbitrary process memory.

### 12.1 Data model

| Artifact | Role |
|---|---|
| `heap_ptr` | Scratch list containing heap cells |
| Free marker | String containing one space (`" "`), not list deletion |
| `P_start` | First allocated cell for pointer `P` |
| `P_end` | Last allocated cell for pointer `P` |
| `P_cur` | Current pointer cursor |
| `P` | Public pointer value/cursor used by dereference syntax |

The allocator reuses available holes where possible and grows the backing list when required. Freeing a pointer writes space markers into its allocated cells; it does not delete list elements, so positions remain stable.

### 12.2 Allocation, read, write, and movement

```scrit
@import ptr as ptr

p = ptr.allocator(4)
*p = 100
value = *p
p = ptr.move(2)
p = ptr.step(p)
ptr.write(p, 200)
value2 = ptr.peek(p)
size = ptr.sizeof(p)
heap_length = ptr.len()
ptr.free(p)
ptr.free_heap()
```

Available special operations include:

| Call / syntax | Meaning |
|---|---|
| `p = ptr.allocator(n)` | Allocate `n` heap cells and initialize pointer metadata |
| `*p = value` | Write to the current cell |
| `value = *p` | Read current cell |
| `ptr.write(p, value)` | Explicit write to pointer `p` |
| `ptr.peek(p)` | Reporter-style read for a known pointer handle |
| `p = ptr.move(offset)` | Set current cursor offset |
| `p = ptr.step(p)` | Advance pointer (the parser supports the pointer-specific step form) |
| `ptr.free(p)` | Mark allocation cells as free spaces |
| `ptr.free_heap()` | Clear heap list and free all tracked pointers |
| `p = ptr.null()` | Null pointer value (`0`) |
| `ptr.is_null(p)` | Null comparison expression |
| `ptr.sizeof(p)` | Allocation size from `start/end` metadata |
| `ptr.len()` | Heap list length |

### 12.3 Pointer safety diagnostics

| Code | Meaning |
|---|---|
| E006 | Use after free |
| E007 | Double free |
| E008 | Pointer not allocated / unbound |
| E009 | Pointer cursor out of bounds |
| E010 | Allocation was not freed at end of static tracking |
| E011 | Incompatible pointer type |
| E018 | Duplicate allocation/symbol condition |

The static pointer tracker catches many direct errors but does not prove all runtime paths safe. Use structured allocation/use/free lifetimes. Avoid aliasing multiple pointer names to the same allocation unless the code explicitly manages that aliasing.

### 12.4 Index rules

Pointer heap cells are 1-based. The allocation metadata `start`, `end`, and `cur` use the same 1-based positions. Offsets and movement must remain inside the allocated interval.

### 12.5 Optimizer interaction

The `mem` pass may remove unused pointer metadata or provably empty heap operations. It must not remove a heap read/write that affects observable behavior. Agents must preserve allocation, dereference, and free ordering.

---

## 13. `stack` — dynamic LIFO list

`stack` uses Scratch list storage (`stack_data`) and a stack pointer variable (`sp`) in the generated runtime representation.

### 13.1 API

```scrit
@import stack as stack

stack.clear()
stack.push(10)
stack.push(20)
last = stack.peek()
value = stack.pop()
size = stack.size()
empty = stack.is_empty()
stack.dup()
stack.swap()
stack.grow(100)
```

| Form | Meaning |
|---|---|
| `stack.push(value)` | Push a value; runtime list grows as needed |
| `value = stack.pop()` | Read and remove the top item |
| `stack.pop()` | Remove the top item and discard it |
| `value = stack.peek()` | Read top item without removing it |
| `stack.clear()` | Delete all stack list items |
| `size = stack.size()` | Stack/list length |
| `empty = stack.is_empty()` | Boolean empty test |
| `stack.dup()` | Duplicate the top item |
| `stack.swap()` | Swap the top two items |
| `stack.grow(n)` | Ensure runtime stack list length is at least `n` by appending zero values |

### 13.2 Safety

- Push before pop/peek.
- Static underflow estimate may produce E017.
- Runtime input-dependent underflow still needs explicit guards in user code.
- `stack.grow(n)` is capacity preparation, not a guarantee that all subsequent operations are valid.

---

## 14. `lba` — Logical Block Addressing store

`lba` models a logical block device using Scratch lists and variables. It is an in-project simulation, not a physical disk, file system, or operating-system block device.

### 14.1 Storage artifacts

| Artifact | Role |
|---|---|
| `lba_disk` | Block data list; absolute LBA indexes are 1-based |
| `lba_map` | Allocation metadata (`0` free, `1` used) |
| `lba_capacity` | Current number of logical blocks |
| `lba_block_size` | Block-size metadata (default initialized by format; commonly set to 512) |
| `lba_errno` | Runtime status code |
| `lba_sym_keys` / `lba_sym_vals` | Symbol-to-address lookup tables |

### 14.2 Types and extent layout

| Type | Meaning |
|---|---|
| `lba` | Absolute logical block address (number) |
| `lba_ext` | Extent handle represented by variables for base LBA, length, cursor offset, and live flag |

For an extent `e`, the runtime metadata uses:

- `e_lba`: absolute first block (1-based).
- `e_len`: extent length.
- `e_off`: cursor offset (starts at zero).
- `e_live`: whether the handle is currently valid.

The absolute block read/written by an extent cursor is `e_lba + e_off`. Consequently, `e_off` is zero-based even though absolute LBA values are one-based.

### 14.3 API

```scrit
@import lba as lba

lba.format(256)
lba.grow(512)
capacity = lba.capacity()
block_size = lba.block_size()
lba.set_block_size(512)

lba.write_at(10, 99)
value = lba.read_at(10)

e = lba.alloc(8)
lba.write(e, 42)
value2 = lba.read(e)
e = lba.seek(e, 3)
e = lba.advance(e)

lba.symbol("boot", e)
address = lba.lookup("boot")
extent_base = lba.lba_of(e)
extent_length = lba.len_of(e)
status = lba.errno()
lba.free(e)
```

**Methods / expression helpers:**

- `format(n)`, `free_disk()` — reset/format backing data.
- `grow(n)` — ensure capacity is at least the requested size.
- `capacity()`, `block_size()`, `set_block_size(n)`, `errno()`.
- `alloc(n)` / `allocator(n)` — allocate an extent.
- `free(e)` — free an extent.
- `read_at(address)`, `write_at(address, value)` — absolute block access.
- `read(e)`, `write(e, value)` — extent cursor access.
- `seek(e, offset)`, `advance(e)` / `step(e)`.
- `lba_of(e)`, `len_of(e)`, `symbol(name, address)`, `lookup(name)`.
- `null()`, `is_null(e)`.

Some helpers are expression forms and some are statement forms. Use assignment for value-producing calls (for example `value = lba.read_at(10)`) and a standalone statement for writes/frees.

### 14.4 Allocation behavior — do not assume first-fit

The current extent allocator appends/grows the disk from the previous capacity and marks the new range as used in `lba_map`. Do **not** describe it as a first-fit allocator that reuses arbitrary free extents. Freeing marks the allocation map entries free and invalidates the extent metadata; the current allocator's growth strategy remains append-oriented.

### 14.5 Bounds and runtime errors

| Condition | Runtime behavior |
|---|---|
| Absolute LBA outside `1..capacity` | Operation is skipped and `lba_errno` becomes `36` (E036) |
| Extent is dead or cursor offset is outside the extent | Operation is skipped and `lba_errno` becomes `37` (E037) |
| Free an already freed extent | Static/runtime double-free handling; E007 / errno 37 |
| Use a non-extent where an extent is required | E038 may be reported |

Always validate addresses and handle `lba.errno()` after operations that may fail. Do not infer that a read succeeded merely because an output variable has a value; on failure, the old/default value may remain.

---

## 15. Diagnostics — complete agent-oriented catalogue

Diagnostics contain a code, severity, message, hint, line, and file. When reporting an issue, quote the exact E-code and the source symbol/line.

| Code | Severity | Meaning / agent action |
|---|---|---|
| E001 | error | Sprite source lacks `properties:`. Add the required header. |
| E002 | error | Required property is missing. Add the named field. |
| E003 | error | Unknown built-in library. Check spelling/registered library list. |
| E004 | error | Unknown name or invalid `$` use. Check definitions/import alias/indirection. |
| E005 | error | Unknown library member/method. Check alias and supported API. |
| E006 | error | Pointer use after free. Stop dereferencing/moving it. |
| E007 | error | Double-free. Remove repeated free or create a new allocation. |
| E008 | error | Pointer was never allocated. Allocate before use. |
| E009 | error | Pointer bounds violation. Check allocation range and cursor. |
| E010 | warn | Pointer allocation not statically freed. Add `ptr.free` or `ptr.free_heap` if appropriate. |
| E011 | error | Type mismatch. Correct the annotation/value or operation. |
| E012 | error | Invalid `@local` / `@cloud` declaration usage. Apply only to declarations. |
| E013 | error | Imported project file is missing. Check exact filename/path. |
| E014 | error | A library contains `properties:`. Move sprite metadata to sprite source. |
| E015 | error | `if` lacks a condition. Add an expression. |
| E016 | error | Invalid SB3 input shape or internal compiler error. Preserve the source and report the diagnostic/stack. |
| E017 | warn | Possible stack underflow. Push first or guard pop/peek. |
| E018 | error | Duplicate symbol/allocation. Rename or remove duplicate declaration/allocation. |
| E019 | info | Reporter used as a statement. Assign/use the reporter as a value. |
| E020 | error | Strict mode variable lacks a type annotation. Add a supported annotation. |
| E021 | error | Invalid sprite property. Check name, numeric fields, visibility, and costume key. |
| E022 | error | Duplicate function or unsupported nested function. Rename/remove duplicate; do not overload. |
| E023 | error | Duplicate macro. Rename/remove one declaration. |
| E024 | error | Invalid assignment target, including assignment to a procedure argument. |
| E025 | error | Malformed call or invalid procedure parameters. Check identifier syntax/parentheses. |
| E026 | error | Empty control body is suspicious. Add statements or remove the structure. |
| E027 | error | Unclosed string or parenthesis. Balance quotes and parentheses. |
| E028 | error | Invalid event header. Use one of the documented event forms. |
| E029 | error | Reporter cannot be a command statement. Use it in an expression/assignment. |
| E030 | error | Wrong procedure argument count. Match the declared arity. |
| E031 | warn | Unreachable code after a terminating construct. Recheck intent and placement. |
| E032 | error | Optimizer internal error. Retry at O0, preserve logs, and report the failing pass. |
| E033 | error | Recursive macro diagnostic code. Avoid recursive/cyclic macro expansion; do not rely on complete cycle detection. |
| E034 | warn | Shadowed name. Rename the inner/outer symbol to make intent explicit. |
| E035 | error | Invalid type name. Use supported declaration types and compiler-inferred pointer/LBA handles. |
| E036 | error | LBA absolute address out of bounds. Grow/format disk or use a valid address. |
| E037 | error | LBA extent invalid/freed/out of range. Allocate a live extent and keep its cursor in bounds. |
| E038 | error | Expected an LBA extent. Pass a handle created by `lba.alloc`. |

### 15.1 Diagnostic discipline for agents

- Never suppress or delete diagnostics to make a build look clean.
- Resolve errors before interpreting optimizer results.
- A warning may still indicate an actual runtime bug; explain whether it is safe to leave.
- Do not claim an E-code is emitted on every possible path just because it is in the catalogue; some checks are static approximations.
- If E016/E032 appears, include the exact file, line, optimization level, and the first internal error message.

---

## 16. Scratch block graph and SB3 data contract

### 16.1 Input tuples

Every Scratch input array must begin with one of these tags:

| Tag | Shape | Meaning |
|---:|---|---|
| `1` | `[1, shadowId]` | Shadow block only |
| `2` | `[2, blockId]` | Real block input |
| `3` | `[3, blockId, shadowId]` | Real block with shadow fallback |

No other leading tag is valid in the emitted SB3 input shape. Invalid shapes can produce E016 or broken project blocks.

### 16.2 Block links

- `next` is the next command in the same stack.
- `parent` points to the containing block or script owner.
- `inputs` reference reporters, shadow blocks, or substacks.
- `topLevel` marks top-level hats and procedure definitions.
- A block ID must resolve to an entry in the block map.
- A `next` chain must not contain cycles.
- Parent chains must not be cyclic.

### 16.3 Procedure blocks

A generated custom procedure must preserve all of the following:

- `procedures_definition` is top-level.
- Its `custom_block` input references the corresponding prototype.
- `procedures_prototype` contains the same mutation metadata used at call sites.
- Each parameter has a stable argument ID and a prototype shadow reporter.
- `procedures_call` carries the matching mutation and all parameter inputs.
- The mutation retains `warp: "true"` when instant execution is intended.
- Procedure body command chains remain attached to the definition's `next` chain.

Do not rename or discard argument IDs independently in the definition and call blocks. Do not treat a `procedures_call` as a pure reporter or remove it because it has no direct variable output.

### 16.4 Graph normalization

Before export, the IDE runs an iterative graph-normalization pass that:

- Removes missing `next` references.
- Breaks malformed `next` cycles.
- Removes or repairs dangling input references where possible.
- Breaks malformed parent cycles.

This is defensive repair, not a substitute for correct code generation. It cannot infer the intended semantics of a malformed block graph. Agents must still preserve valid parent/next/input relationships.

---

## 17. Compilation, optimization, and performance

### 17.1 Commands and optimization level

```text
compile
optimize -O0
optimize -O1
optimize -O2
optimize -O3
optimize -Bandit
optimize -O3 -Bandit
bandit
optlevel
```

The default optimization level is O0. `optimize` changes the IDE's current optimization level; later `compile`/`export` operations use the configured level. Use `optimize -O0` to return to the unoptimized baseline.

### 17.2 Levels

The current optimizer is SIR-based. Higher levels enable more passes/analysis, but they do not guarantee a smaller project or faster Scratch runtime for every source program.

| Level | General intent |
|---|---|
| O0 | Basic reachability DCE; reference build for correctness |
| O1 | Basic folding, algebraic simplification, affine/constant loop work, and selected safe passes |
| O2 | Adds stronger global value numbering and runtime 1D affine loop optimization paths |
| O3 | Stronger equality-saturation budget and second optimization wave (`sat2`) |
| `-Bandit` | UCB1-style pass selection; implies O3 in the console command handler |

The exact pass selection may depend on the `-Bandit` selection state and the current implementation. Treat O0 as the semantic baseline.

### 17.3 Intermediate representation

The optimizer converts non-shadow Scratch blocks into SIR nodes containing:

- Block ID and opcode.
- `next` and `parent` links.
- Input and field maps.
- Procedure mutation metadata, where present.
- Liveness state.
- Analysis fields for SSA versions, uses/defs, pulse IDs, equality classes, and folded constants.

After the passes, live SIR nodes are reconstructed into Scratch blocks, with original shadow blocks restored where referenced.

### 17.4 Pass catalogue

| Pass | Current role |
|---|---|
| `dce` | Reachability-based dead-code elimination from hats/top-level entries |
| `egraph` | Equality saturation and constant/algebraic folding |
| `smt` | Lightweight constant equality propagation |
| `strength` | Strength reduction and removal of no-op arithmetic such as adding zero where safe |
| `gvn` | Global value numbering / common-expression reuse for selected pure reporters |
| `pulse` | Side-effect/pulse analysis and constant-branch pruning |
| `linear` | Simplify straight-line affine assignments and constant changes |
| `loops` | Collapse eligible pure `repeat` loops into affine closed forms |
| `inline` | Collapse selected one-use temporary assignments; this is not source-level `def` inlining |
| `mem` | Remove selected unused pointer metadata or empty heap operations |
| `unroll` | Simplify `repeat 1` and eliminate `wait 0` |
| `empty` | Remove empty control structures |
| `sat2` | Second saturation/cleanup wave, enabled at O3 |

### 17.5 Procedure safety rules for the optimizer

- `procedures_call` is treated as side-effecting.
- The `inline` pass walks procedure-definition bodies and excludes those nodes from ordinary temporary collapsing.
- Procedure arguments, call mutation metadata, hidden return state, and frame-list operations are semantically significant.
- Do not inline a `def` body into every caller as an optimization shortcut.
- Do not remove a call solely because its result is unused; the function may mutate variables, lists, pointers, LBA storage, draw, sense, play sound, or broadcast.
- Do not move side-effecting operations across function calls or control boundaries.

These safeguards are part of the current optimizer design, not a formal proof that every pass is bug-free. Compare output behavior against O0 when changing optimizer code.

### 17.6 Loop-to-equation optimization

Eligible pure data loops may be modeled as an affine transform:

\[
  x \leftarrow A x + b
\]

For constant iteration count \(N\), the compiler may emit the closed form:

\[
  x_N = A^N x_0 + \left(\sum_{i=0}^{N-1} A^i\right)b
\]

The current implementation restricts the affine model to a small number of variables (up to three in the matrix path) and only collapses bodies it can model as pure affine data updates.

Side-effecting bodies such as movement, pen drawing, asking the user, broadcasting, procedure calls, list mutations, or other non-affine operations must not be treated as affine data loops. Keep them as loops.

Runtime-count affine loops have a specialized 1D path at higher optimization levels. Agents must not rewrite a loop into a mathematical closed form unless the transformation is valid for the actual value domain and side effects are absent.

### 17.7 `-Bandit`

`-Bandit` uses a multi-armed-bandit/UCB1 selection strategy to select optimization passes using an approximate reward derived from blocks removed and pass time. The current implementation always keeps core passes including `dce`, `egraph`, `gvn`, `inline`, and `mem`; other passes may be selected according to the bandit state.

`bandit` prints pass-arm statistics. Because pass selection is adaptive, do not expect two different project histories to select exactly the same optional passes.

### 17.8 Performance guidance for source generators

- Prefer static variable names over runtime `$` indirection.
- Prefer one reusable `def` over repeated copied code.
- Avoid generating thousands of nearly identical reporter/control blocks where a loop or procedure is sufficient.
- Use lists for indexed collections rather than generating a separate variable for every element.
- Avoid excessive nested `if` structures where a simple lookup or clear data structure works.
- Do not optimize side-effect-heavy loops into algebra.
- Do not create enormous compile-time macro expansions.
- Keep expression nesting moderate; deep nested reporter trees are more difficult for editors/runtimes to process.
- Use O0 compilation as a baseline and compare block counts/timing at each optimized level.

---

## 18. IDE console and project operations

### 18.1 Console commands

```text
help
ls
cat <file>
rm <file>
new <file> [lib]
open <file>
include "https://host/path/file.scrit"
compile
export
optimize -O0|-O1|-O2|-O3|-Bandit
bandit
optlevel
cache
clear
cls
```

| Command | Purpose |
|---|---|
| `help` | Show command summary |
| `ls` / `dir` | List project files; files without properties are displayed as libraries |
| `cat <file>` | Print source (console output is truncated to a bounded amount) |
| `rm <file>` / `del <file>` | Delete a project file (the IDE keeps at least one file) |
| `new <file> [lib]` | Create a sprite or library source |
| `open <file>` | Activate a file tab |
| `include "url"` | Fetch remote source into the project file system as a library |
| `compile` | Compile using current optimization setting; default O0 |
| `export` | Compile and write `.sprite3` or `.sb3` |
| `optimize -O<n>` | Select optimization level and recompile |
| `optimize -Bandit` | Enable bandit selection and O3 |
| `bandit` | Show adaptive pass statistics |
| `optlevel` | Show current optimization level and bandit state |
| `cache` | Show remote-import cache |
| `clear` / `cls` | Clear console output |

### 18.2 UI areas

| Area | Purpose |
|---|---|
| Explorer | Project files and active file |
| Editor | Scrit source text |
| Problems | Diagnostics with code, message, hint, line, and file |
| Console | Build/export commands and output |
| Blocks | Preview of generated block opcodes |
| Memoria | Pointer states and variable-kind tags |
| Compilador | Timestamped parse/link/opt/export trace |
| Footer | Current status, timing, and cursor position |

### 18.3 Build/export troubleshooting

If the IDE or target runtime shows an error such as `RangeError: Maximum call stack size exceeded`:

1. Record the exact error and whether it happens on compile, optimize, export, or project load.
2. Set O0 (`optimize -O0`) and compile again.
3. Compare the number of generated blocks per sprite before and after optimization.
4. Check for runtime `$` reads, large dynamic map writes, huge macro expansion, deeply nested reporter expressions, and large generated control graphs.
5. Check whether the failing path is `.sprite3` or multi-target `.sb3` export.
6. Inspect procedure definitions/calls and ensure no malformed `next`, `parent`, or input references were introduced by an optimization.
7. Do not silently delete blocks or truncate source as a workaround. Preserve a minimal reproduction and the first failing optimizer/export phase.

The exporter includes iterative graph cleanup for malformed links, but it cannot repair arbitrary semantic corruption or every possible recursion/stack overflow in all host UI components.

---

## 19. Agent generation patterns

### 19.1 Minimal sprite

```scrit
properties:
  name = "Hello"
  size = 100
  position = 1
  rotation = 90
  pos_x = 0
  pos_y = 0
  show = true
  costume = cpu
  strict = false

@import looks as look

when flag_clicked:
  look.say("Hello, Scratch!")
```

### 19.2 Reusable native procedures

```scrit
properties:
  name = "Procedures"
  size = 100
  position = 1
  rotation = 90
  pos_x = 0
  pos_y = 0
  show = true
  costume = dsp
  strict = false

@import looks as look

when flag_clicked:
  total = sum_values(10, 20)
  look.say(total)

def sum_values(a, b):
  return a + b
```

Expected compile shape: one custom procedure definition and a call from the green-flag script; the procedure is not expanded into the caller.

### 19.3 Early return

```scrit
def safe_ratio(a, b):
  if b == 0:
    return 0
  result = a / b
  return result

when flag_clicked:
  value = safe_ratio(10, 2)
  look.say(value)
```

Expected behavior: if `b == 0`, the return flag prevents `result = ...` and the later return from running in that invocation.

### 19.4 Nested procedure calls

```scrit
def increment(value):
  return value + 1

def multiply(a, b):
  return a * b

when flag_clicked:
  result = multiply(increment(2), increment(3))
  look.say(result)
```

Expected shape: actual procedure calls plus hidden temporary variables, not inline macro substitutions.

### 19.5 Lists and loops

```scrit
values = []
values.append(3)
values.append(6)
values.append(9)

sum = 0
for i in range(1, len(values) + 1):
  sum += values[i]
```

Remember that the list is 1-based. When exact range bounds matter, test the generated repeat count.

### 19.6 Runtime-selected variable

```scrit
properties:
  name = "Dynamic"
  size = 100
  position = 1
  rotation = 90
  pos_x = 0
  pos_y = 0
  show = true
  strict = false

name = "score"
score = 0
$name = 12
current = $name
```

The canonical source syntax for dynamic write is `$name = value` where `name` is the variable holding the destination key. Do not insert whitespace between `$` and the identifier. Use runtime indirection only when the key is genuinely dynamic.

### 19.7 LBA store

```scrit
@import lba as lba

when flag_clicked:
  lba.format(64)
  lba.set_block_size(512)
  lba.write_at(1, 73)
  magic = lba.read_at(1)
  if magic == 73:
    look.say("Disk OK")
```

### 19.8 Pointer lifecycle

```scrit
@import ptr as ptr

when flag_clicked:
  p = ptr.allocator(4)
  *p = 42
  value = *p
  ptr.free(p)
```

### 19.9 Library expression helper

`math_helpers.scrit` (library; no `properties:`):

```scrit
def square(x):
  return x * x
```

Sprite file:

```scrit
@import "math_helpers.scrit" as util

when flag_clicked:
  result = util.square(9)
```

Keep library functions expression-only. Use main-file `def` for real custom procedures with multiple statements.

---

## 20. Agent checklist before returning a project

### Source structure

- [ ] Every sprite file starts with valid `properties:`.
- [ ] Required property keys exist and use correct types.
- [ ] Costume name is one of the registered themes.
- [ ] Library files contain no `properties:`.
- [ ] Imports use real built-in names or exact project filenames.

### Syntax and API

- [ ] `look.say(...)` is used instead of Python `print`.
- [ ] Reporters are used as values; methods are used as statements.
- [ ] Core `str`, `number`, and `len` are not mistaken for Python conversions.
- [ ] List indexes are 1-based.
- [ ] `ptr` and `lba` operations use their exact documented signatures.
- [ ] No unsupported Python constructs are generated.

### Procedures and macros

- [ ] Every reusable runtime routine is a top-level `def`, not a macro.
- [ ] Every macro is declared with `macro name(...):` and used only for compile-time substitution.
- [ ] No duplicate `def` names (E022); no duplicate macro names (E023).
- [ ] No nested `def` declarations.
- [ ] Call argument count matches function arity (E030).
- [ ] Function arguments are not assigned (E024).
- [ ] `return` occurs only inside a main-file `def`.
- [ ] Functions used as values are tested in nested expressions/conditions.
- [ ] Function working variables are treated as shared Scratch variables, not automatic locals.
- [ ] Imported library functions contain one pure `return expression`.

### Runtime correctness

- [ ] Every `if` has a condition.
- [ ] Every control block has a meaningful body.
- [ ] Early return has been checked against later statements in the same function.
- [ ] Loops have valid termination conditions or intentional forever behavior.
- [ ] Side effects have not been removed/reordered.
- [ ] Pointer allocation/read/write/free order is valid.
- [ ] LBA addresses and extent offsets remain within bounds.
- [ ] Stack pop/peek operations are guarded where underflow is possible.

### Build, optimize, export

- [ ] O0 compilation succeeds first.
- [ ] No unresolved E-code diagnostics remain.
- [ ] O1/O2/O3 builds preserve O0 behavior for representative tests.
- [ ] Custom procedure definitions, calls, mutation metadata, and `warp: "true"` remain intact.
- [ ] Export target is correct: one sprite → `.sprite3`; multiple sprites → `.sb3`.
- [ ] The generated archive opens in the intended Scratch/TurboWarp runtime when actual runtime verification is available.
- [ ] The final response distinguishes compile success, archive generation, and runtime testing.

---

## 21. Informal grammar (implemented subset)

This grammar is an aid, not a complete formal parser specification. Indentation is represented by `suite`.

```text
program          := properties? imports? top_level_item*
properties       := "properties:" newline property_line+
property_line    := indent identifier "=" literal
imports          := import_line*
import_line      := "@import" (identifier | string) "as" identifier

macro_def        := "macro" identifier "(" parameters? ")" ":" suite
def_def          := "def" identifier "(" parameters? ")" ":" suite
parameters       := identifier ("," identifier)*

top_level_item   := statement | event | def_def | macro_def
statement        := assignment
                  | function_call
                  | library_call
                  | wait_call
                  | broadcast_call
                  | pointer_statement
                  | stack_statement
                  | lba_statement
                  | control
                  | return_statement

assignment       := variable_target assignment_operator expression
variable_target  := identifier [":" type]
                  | "@local" identifier [":" type]
                  | "@cloud" identifier [":" type]
                  | "*" identifier
                  | "$" identifier
assignment_operator := "=" | "+=" | "-=" | "*=" | "/=" | "%="

event            := "when flag_clicked:" suite
                  | "when key_pressed(" string "):" suite
                  | "when sprite_clicked:" suite
                  | "when clone_started:" suite
                  | "when receive(" string "):" suite
                  | "when message_received(" string "):" suite
                  | "when backdrop_switches(" string "):" suite
                  | "when backdrop_changes(" string "):" suite
                  | "when loudness >" expression ":" suite

control          := if_control
                  | "forever:" suite
                  | "repeat" expression ":" suite
                  | "repeat until" expression ":" suite
                  | "while" expression ":" suite
                  | "for" identifier "in range(" range_args "):" suite
if_control       := "if" expression ":" suite ["else:" suite]
range_args       := expression | expression "," expression
                  | expression "," expression "," expression

return_statement := "return" [expression]
function_call    := identifier "(" arguments? ")"
library_call     := identifier "." identifier "(" arguments? ")"
reporter         := identifier "." identifier
expression       := literal | identifier | reporter | list_access
                  | call | unary | binary | "(" expression ")"
list_access      := identifier "[" expression "]"
unary            := ("-" | "+" | "!" | "not" | "$" | "*") expression
binary           := expression operator expression
literal          := number | string | "true" | "false" | "[]"
type             := "number" | "string" | "ptr"
```

---

## 22. Implementation notes and explicit non-guarantees

This section prevents an agent from overstating the language implementation.

1. Scrit compiles to Scratch block graphs; `warp: "true"` custom procedures are Scratch VM procedures, not native machine code.
2. Scratch custom procedures are command blocks. Scrit simulates return values using hidden variables, completion flags, and per-function frame lists.
3. The procedure frame mechanism protects hidden return/done state; it does not create private storage for every source variable in a function body.
4. Imported library `def`s currently expose extracted return expressions; they are not equivalent to full main-file custom procedures.
5. `@cloud` is tracked as a kind in IDE metadata, but the current export shape does not prove that a real Scratch cloud variable is created.
6. `str` and `number` are not full Python type-conversion functions in the current expression compiler.
7. `range` lowering is designed around Scratch repeat counts; exact Python behavior for negative/fractional steps should not be assumed without tests.
8. Runtime `$` indirection creates list-backed map logic and may significantly increase block count.
9. The optimizer is a collection of compiler passes, not a formal equivalence prover. Always compare optimized behavior against O0 when semantics are non-trivial.
10. LBA allocation currently grows the disk at the end; do not claim it reuses arbitrary freed extents using first-fit.
11. The graph normalizer repairs some malformed links but cannot restore lost semantic information.
12. Successful ZIP/JSON generation is not the same as opening and executing the exported project in Scratch Desktop or TurboWarp.

---

## 23. Reference summary

- **Language:** Scrit
- **Style:** Python-like indentation and calls; Scratch-oriented runtime
- **Targets:** Scratch 3 / TurboWarp `.sprite3` and `.sb3`
- **Functions:** native Scratch custom procedures; `warp: "true"`; result/return state emulated with variables and frame lists
- **Macros:** explicit `macro` declarations expanded at compile time
- **Pointer backend:** Scratch list `heap_ptr`; free cells marked with a single space; 1-based heap indexes
- **Stack backend:** Scratch list `stack_data` plus stack pointer state
- **LBA backend:** `lba_disk`, `lba_map`, capacity/block-size/errno metadata; append-oriented allocation
- **Optimizer:** SIR, SSA approximation, e-graph, SMT-lite, pulses, strength reduction, GVN, affine loop passes, inline-temp cleanup, memory cleanup, UCB1-style `-Bandit`
- **Diagnostics:** E001–E038
- **IDE:** multi-file Explorer, editor, Problems, Console, Blocks, Memoria, Compilador trace, SVG costume themes
