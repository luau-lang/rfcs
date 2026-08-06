# `if local` statements and expressions

## Summary

Allow a `local` or `const` binding in the condition of `if` or `elseif` statements or expressions. The variable is scoped to the `then`\-block, and the branch is taken when the initializer is truthy. `const` is included alongside `local` because it is already a contextual keyword valid everywhere `local` is valid.

```lua
-- statement
if local x = expr() then
    -- x is truthy here
end

-- expression
local y = if local x = expr() then x else "default"
```

`const` works the same way, with the additional guarantee that the binding cannot be reassigned within the block:

```lua
if const x = expr() then
    -- x is truthy here and cannot be reassigned
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

Every nil-checked access takes two lines, and the variable leaks into the surrounding scope where it may be nil and shouldn't be used. This shows up anywhere an API returns `T?`, like in table lookups, string matching, and Roblox code like `Player.Character`, or `FindFirstChild`

With `if local`:

```lua
if local user = users[id] then
    if local email = user.email then
        sendEmail(email)
    end
end
```

The variable's scope is exactly where it's guaranteed to be non-nil. Note that this still requires nesting for the two-step lookup above; a single binding per `if` is the minimal useful case, and chaining multiple bindings into one condition is left for future work (see below).

## Design

### Syntax

When the parser sees `local` or `const` immediately after `if` or `elseif`, it parses:

```lua
('local' | 'const') Name [':' Type] '=' exp
```

The initializer is the entire condition: nothing else appears between it and `then`. `local` cannot start an expression, so `if local` is unambiguous. `const` is a contextual keyword and is also a valid variable name, so `if const then` is legal today (testing a variable named `const` for truthiness). This is resolved with a single token of lookahead: `if const Name` is not currently legal syntax, so when the parser sees `const` followed by a name, it parses a binding declaration.

### Semantics

The initializer is evaluated. If the result is truthy (not `nil`, not `false`), the branch is taken and the binding is available inside the `then`\-block. Otherwise execution falls through to `elseif` or `else`. The variable does not exist outside its block. A `const` binding follows the same reassignment restriction it has as a statement: the binding itself can't be reassigned inside the `then`\-block, though a mutable value it refers to (e.g. a table) can still be mutated.

### Scoping

Each branch introduces its own independent binding:

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

If a variable of the same name already exists in an enclosing scope, `if local x = ...` (or `if const x = ...`) shadows it for the duration of the `then`\-block, exactly as an ordinary `local x = ...` or `const x = ...` statement would. The outer `x` is unaffected and reappears after the block ends.

### Type narrowing

The type checker narrows the binding using the same truthiness refinement it already applies to `if x then` — removing `nil` and `false` from the type:

```lua
-- users[id] returns User?
if local user = users[id] then
    -- user: User (not User?)
    user.lastSeen = os.time()
end
```

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

### Desugaring

#### Statements

`if local x = E then B end` is equivalent to:

```lua
do
    local x = E
    if x then
        B
    end
end
```

and likewise `if const x = E then B end` desugars to a `do` block with a `const x = E` statement in place of `local`. No new bytecode instructions or evaluation rules are needed for either form. It is syntactic sugar with tighter scoping.

#### Expressions

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

## Drawbacks

**No compound conditions yet.** There is no way to write `if local x = expr() and x > 0 then`, where the branch is taken when x is greater than zero. The binding *is* the condition. See Alternatives for why this is out of scope for now rather than an open design question.

## Alternatives

### Do nothing

The two-line pattern works, but it is verbose enough that developers often skip the nil-check entirely, leading to "attempt to index nil" errors on instance access.

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

### `while local` loops

```lua
while local line = reader:nextLine() do
    process(line)
end
```

The binding is re-evaluated each iteration and the loop terminates when the expression produces a falsy value. `while` is a different syntactic construct from `if` with different loop-scoping semantics, so it is left for a separate proposal.

Alternatively, `repeat ... until cond` already lets `cond` see locals declared in the loop body, so the following is a workaround today:

```lua
repeat
	local line = reader:nextLine()
	if line then
		process(line)
	end
until not line
```

### Multi-return binding

```lua
if local ok, result = pcall(riskyFunction) then
    use(result)
end
```

This raises questions about which value is tested for truthiness (the first? all of them?) and is left for a separate proposal.

### Destructuring patterns

Binding into a shape rather than a single name, e.g. `if local Point {x, y} = p then ... end`, was raised in discussion as a natural extension once the language has more general pattern-matching support, but it's a large addition on its own and out of scope here.