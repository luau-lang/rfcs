# Conditional bindings (`if local` and `while local`)

## Summary

Allow a single `local` or `const` binding in the condition of `if` or `elseif` statements and expressions, and of `while` loops. The branch is taken (or the loop body runs) when the initializer is truthy, and the variable is scoped to the `then` block (or loop body). Additionally, `const` is supported wherever `local` is.

```lua
-- statement
if local x = expr() then
    -- x is truthy here
end

-- expression
local y = if local x = expr() then x else "default"

-- loop
while local line = reader:nextLine() do
    -- line is truthy here, and re-bound each iteration
end
```

`const` works the same way, with the additional guarantee that the binding cannot be reassigned within the block:

```lua
if const x = expr() then
    -- x is truthy here and cannot be reassigned
end

while const item = queue:pop() do
    -- item is truthy here and cannot be reassigned
end
```

## Motivation

A common pattern is to call something that might return nil, bind the result, and then check it:

```lua
-- users: {[number]: User}, where User = {email: string?}
local user = users[id]
if user then
    local email = user.email
    if email then
        sendEmail(email)
    end
end
```

Each nil-checked access takes two lines, and the variable stays in the surrounding scope where it could be nil and shouldn't be used. This shows up anywhere an API returns `T?`, like in table lookups, string matching, and Roblox code like `Player.Character`, or `FindFirstChild`.

With `if local`:

```lua
if local user = users[id] then
    if local email = user.email then
        sendEmail(email)
    end
end
```

The variable is only in scope where it's guaranteed to be non-nil. This example still requires nesting for the two-step lookup above, but each level of nesting now corresponds to a single safe binding, and chaining multiple bindings into one condition is left for future work (see [Possible Future Work](#possible-future-work)).

Loops that consume values until their source is exhausted (i.e. iterators, readers, queues) have a similar problem. Today, they either repeat the call before the loop and at the end of the body:

```lua
local line = reader:nextLine()
while line do
    process(line)
    line = reader:nextLine()
end
```

or use an unconditional loop with a manual `break`:

```lua
while true do
    local line = reader:nextLine()
    if not line then
        break
    end
    process(line)
end
```

But with `while local`:

```lua
while local line = reader:nextLine() do
    process(line)
end
```

## Design

### Syntax

When the parser sees `local` or `const` immediately after `if`, `elseif`, or `while`, it parses:

```lua
('local' | 'const') Name [':' Type] '=' exp
```

The initializer is the entire condition: nothing else appears between it and `then` (or `do`, for `while`), and one name is bound; multiple names and multiple initializers are not supported yet (see [Possible Future Work](#possible-future-work)).

`local` cannot start an expression, so `if local` and `while local` are unambiguous. `const` is a contextual keyword and is also a valid variable name, so `if const then` and `while const do` are legal today (testing a variable named `const` for truthiness). This is resolved with a single token of lookahead: `if const Name` and `while const Name` are not currently legal syntax, so when the parser sees `const` followed by a name, it parses a binding declaration.

### Semantics

#### `if` and `elseif`

The initializer is evaluated. If the result is truthy (not `nil`, not `false`), the branch is taken and the binding is available inside the `then`\-block. Otherwise execution falls through to `elseif` or `else`. The variable doesn't exist outside its block.

#### `while`

At the start of each iteration, the initializer is evaluated. If the result is truthy, a fresh binding is created for that iteration and the body runs with it in scope. Otherwise, the loop exits. The variable doesn't exist after the loop.

Since each iteration gets its own binding, closures created in the body capture that iteration's value, matching the existing behavior of locals declared inside a loop body:

```lua
local callbacks = {}
while local item = queue:pop() do
    table.insert(callbacks, function()
        print(item) -- each closure sees its own item
    end)
end
```

`break` exits the loop as usual. `continue` skips the rest of the body and proceeds to the next iteration, which re-evaluates the initializer.

#### `const`

A `const` binding follows the same reassignment restriction it has as a statement: the binding itself can't be reassigned inside the block it guards, though a mutable value it refers to (e.g. a table) can still be mutated.

A `local` binding may be reassigned inside the block like any other local. For `while local`, reassigning the variable in the body affects only the current iteration; it doesn't influence whether the loop continues, because the initializer is re-evaluated at the start of the next iteration.

### Scoping

Each `if`/`elseif` branch introduces its own independent binding:

```lua
-- admins and guests: {[number]: {name: string}}
if local admin = admins[id] then
    print(admin.name) -- admin is in scope
elseif local guest = guests[id] then
    print(guest.name) -- guest is in scope, admin is not
else
    -- neither in scope
end
-- neither in scope
```

A binding from one branch is never visible while evaluating a later branch's own condition, since that condition only runs if the earlier one was falsy.

A `while local` binding is visible in the loop body only. It isn't visible after the loop, and an iteration's initializer can't see the previous iteration's binding.

If a variable with the same name exists in an enclosing scope, the new binding shadows it for the duration of the block, the same as an ordinary `local x = ...`. The outer `x` is unaffected and reappears after the block ends, and the initializer itself still refers to the outer `x`, so `while local x = x.next do` does not advance through a linked list; it reads the same outer `x` every iteration.

### Type narrowing

The type checker narrows the binding using the same truthiness refinement it already applies to `if x then` — removing `nil` and `false` from the type:

```lua
-- users[id] returns User?
if local user = users[id] then
    -- user: User (not User?)
    user.lastSeen = os.time()
end

-- reader:nextLine() returns string?
while local line = reader:nextLine() do
    -- line: string (not string?)
    print(line:upper())
end
```

#### Explicit type annotations

Since conditional bindings are syntactic sugar for a `local` declaration followed by a truthiness test (see [Desugaring](#desugaring)), an annotation is checked exactly as it would be for that `local`, so

```lua
if local x: T = y then
    -- ...
end
```

behaves like

```lua
local x: T = y
if x then
    -- ...
end
```

Additionally, like a `local` declaration, the annotation is optional. When the initializer returns an optional or `any` type, we annotate with the optional type (or omit it) and let the branch refine it:

```lua
-- expr(): number?
if local x: number? = expr() then
    -- declared type is number?, but x is refined to number here
end
```

The value being bound still has to match the annotation's type, so an incompatible annotation is a type error:

```lua
local function f(v: number?)
    if local x: string = v then -- error: number? is not assignable to string
        print(x)
    end
end
```

The same rules apply to `while local`.

### Expressions

`if local` also works in `if` expressions (Luau's ternary form). The binding is visible in the `then` expression but not the `else` branch:

```lua
local display = if local user = users[id] then user.name else "anonymous"
```

The `else` branch is required, as with all `if` expressions. `elseif local` is supported as well:

```lua
local label = if local admin = admins[id] then admin.title
    elseif local guest = guests[id] then guest.name
    else "unknown"
```

The same scoping rules apply: each binding is only visible in its own `then` expression, and bindings from earlier branches are not visible in later ones.

`while` has no expression form, so there is no expression counterpart to `while local`.

### Desugaring

#### `if` statements

`if local x = E then B end` is equivalent to:

```lua
do
    local x = E
    if x then
        B
    end
end
```

and likewise `if const x = E then B end` desugars to a `do` block with a `const x = E` statement in place of `local`.

#### `if` expressions

`if local x = E then A else B` is equivalent to:

```lua
if E then
    local x = E  -- bound and in scope only for A
    A
else
    B
end
```

In practice the compiler evaluates `E` once, binds the result to `x`, tests truthiness, and produces either `A` (where `x` is in scope) or `B` (where it is not).

#### `while` loops

`while local x = E do B end` is equivalent to:

```lua
while true do
    local x = E
    if not x then
        break
    end
    B
end
```

and likewise `while const x = E do B end` uses `const x = E` in place of `local`. A `continue` inside `B` jumps to the end of the loop body and re-evaluates `E`, which matches this desugaring.

No new bytecode instructions or evaluation rules are needed for any of these forms. They are syntactic sugar with tighter scoping.

## Drawbacks

**No compound conditions yet.** There is no way to write `if local x = expr() and x > 0 then`, where the branch is taken when x is greater than zero. The binding *is* the condition. See Alternatives for why this is out of scope for now rather than an open design question.

## Alternatives

### Do nothing

The two-line pattern works, but it is verbose enough that developers often skip the nil-check entirely, leading to "attempt to index nil" errors on instance access. For loops, the existing workarounds either duplicate the producing call or require a `while true` loop with a manual `break`.

`repeat ... until cond` lets `cond` see locals declared in the loop body, which is a partial workaround today, but it still needs an explicit check inside the body:

```lua
repeat
    local line = reader:nextLine()
    if line then
        process(line)
    end
until not line
```

### Compound conditions

A previous [RFC (\#110)](https://github.com/luau-lang/rfcs/pull/110) proposed multi-binding stacked declarations and an `in` keyword for additional conditions. Discussion on that RFC did not have a consensus on the syntax for chaining a boolean test after the binding: every candidate delimiter is either an existing token being given new meaning (`;`, which the language has otherwise avoided assigning semantics to) or a new reserved keyword (`where`). We think chaining can be added later, given that truthy checks will be the vast majority of usage.

## Possible Future Work

These can build on top without breaking changes, and are intentionally not resolved here:

### Compound conditions

Allow an additional boolean expression after the binding. The exact syntax is unresolved, but candidates raised so far include `;`, `where`, or `and` with special scoping rules:

```lua
-- Possible syntaxes (not proposed here):
if local x = getValue(); x > 0 then ...
if local x = getValue() where x > 0 then ...
```

This is the most-requested extension but requires either giving `;` semantic meaning or reserving a new keyword, both of which have significant design implications and are better suited to a dedicated RFC.

Other languages have a variety of approaches, further motivating this should have its own dedicated design:

* C++17 and Go separate the initializer from the condition with `;`  
  * `if (auto x = expr(); x > 0) { ... }`  
  * `if v, ok := m[k]; ok { ... }`  
* Swift chains a binding with additional boolean tests using `,`  
  * `if let x = expr(), x > 0 { ... }`  
* Rust allows `&&` to chain `let` bindings with boolean tests  
  * `if let Some(x) = expr() && x > 0 { ... }`

### Multiple bindings per condition

Stacked declarations where all must be truthy for the branch to be taken:

```lua
if
    local user = users[id]
    local email = user.email
then
    -- both in scope
end
```

Each binding would be visible to subsequent bindings in the stack, short-circuiting on the first falsy one.

### `if not local` statements

A negated form that takes the branch when the initializer is *falsy* was raised as a fit for early-return guards:

```lua
if not local x = expr() then
    return -- expr() was falsy
end
-- would x be usable (and refined) here?
```

To be useful the binding would need to be in scope *after* the block, not inside the `then`\-block where it is falsy — the opposite of the scoping every other form here uses. That needs its own scoping and refinement story, so it is left for a separate proposal.

### Multi-return binding

```lua
if local ok, result = pcall(riskyFunction) then
    use(result)
end
```

This raises questions about which value is tested for truthiness (the first? all of them?) and is left for a separate proposal.

### Destructuring patterns

Binding into a shape rather than a single name, e.g. `if local Point {x, y} = p then ... end`, was raised in discussion as a natural extension once the language has more general pattern-matching support, but it's a large addition on its own and out of scope here.
