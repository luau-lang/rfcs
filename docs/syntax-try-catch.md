# try catch

## Summary

Adds a `try` construct at the statement and expression level. `try` constructs allow you to evaluate code and intercept errors that occur during execution with a `catch` statement. `try` expressions also permit a catchless evaluation of their expression: when `catch` is omitted, `try` expressions that result in an error evaluate to `nil`.

```lua
-- Expression form: try <expr> [catch <name> then <expr>]
try a.b.c.d                         -- nil if anything in the chain errors
try a.b.c.d catch err then f(err)   -- any fallback you write to handle the error
try maybeErr()                      -- any error is ignored and execution continues

-- Block form: try <block> catch <name> then <block> end
try
   local config = loadConfig()
   applyConfig(config)
catch err then
   warn(`couldn't apply config: {err}`)
end
```

## Motivation

Safe navigation operators ([RFC \#142](https://github.com/luau-lang/luau/pull/142)) and optional chaining like `?.` have been proposed and discussed to short-circuit chains on `nil`, but it was removed via [\#501](https://github.com/luau-lang/luau/pull/501) because of incompatibilities with the Roblox Instance hierarchy. Namely `__index` on an Instance raises an error for a missing child instead of returning `nil`. 

For example, given `vehicle.Turret?.Frame.CFrame`, the error is raised by `vehicle.Turret` before `?.` sees a value, so the chain will throw if `Turret` is absent.

We want a concise and intuitive way to allow users to write “get this value, or fall back if the path doesn’t resolve” that works for both tables and Instances without changing `__index` semantics. 

### Today

Direct indexing is readable and expects the child to exist, but Roblox Instances error if any link is missing:

```lua
local humanoid = workspace.NPC.Humanoid -- errors if the NPC or Humanoid is absent
```

`FindFirstChild` returns `nil`, but chaining it needs nested `if`s or repeated `and` checks, making the code harder to read if used repeatedly, and requires re-evaluating earlier links:

```lua
local result = instance.Parent
if result then
    result = result:FindFirstChildWhichIsA("Humanoid")
    if result then
        result:TakeDamage(100)
    end
end

-- or
instance.Parent
    and instance.Parent:FindFirstChildWhichIsA("Humanoid")
    and instance.Parent:FindFirstChildWhichIsA("Humanoid"):TakeDamage(100)
```

Mixed chains also error easily, since any dotted access involving an Instance can still throw:

```lua
local label = gui and gui:FindFirstChild("HUD") and gui.HUD.Health.Label -- gui.HUD.Health may throw
```

`pcall` currently handles these cases, but it needs a closure, returns two values, and can hide intent:

```lua
local ok, d = pcall(function() return a.b.c.d end) 
if not ok then d = nil end
```

### With `try`

```lua
local label = try player.PlayerGui.HUD.Health.Label
local health = try character.Humanoid.Health catch err then 0
if label and health > 0 then
    try character.Humanoid:TakeDamage(health - 10)
    label.Text = tostring(health - 10)
end
```

It also composes with [conditional bindings](https://github.com/luau-lang/rfcs/blob/if_local/docs/syntax-conditional-bindings.md):

```lua
if local x = try foo() then
    print(x)
else
    print("x is nil")
end

while local y = try foo() then
   print(y)
end
```

### Block form
The block form can replace and simplify the `pcall` pattern as well:

```lua
try
    loadData()
catch err then
    if typeof(err) == "table" and err.kind == "NotFound" then
        useDefaults()
    else
        error(err)
    end
end
```

## Design

### Syntax

There are two forms: In both, `catch` always binds the error.

```
-- Expression form
try <expr>
try <expr> catch <name> then <expr>

-- Block form
try
   <block>
catch <name> then
   <block>
end
```

More formally, 

```
exp  ::= ... | 'try' exp [ 'catch' Name 'then' exp ]
stat ::= ... | 'try' block 'catch' Name 'then' block 'end'
```

### Semantics

* `try` catches the same errors `pcall` does

```lua
local a = try error("error!")       -- nil: raised by error()
local b = try a.field               -- nil: indexing a nil value
local c = try workspace.NoSuchChild -- nil: __index on an Instance errors
```

* `try x` is the same as `try x catch e then nil`

```lua
local a = try foo()
local b = try foo() catch e then nil -- same as above
```

* In the block form, locals from the `try` block aren’t visible in the `catch` block. `return` , `break`, and `continue` work as in a `do` block, and assignments to globals and outer locals work

```lua
local result
try
   local data = load()
   result = process(data)        -- assigning to an outer local works
catch err then 
   result = 0                    -- `data` is not in scope here
end

for _, item in items do
   try do
      process(item)
   catch err then 
      break                      -- exits loop
   end
end
```

* The catch expression or block is only evaluated if an error is caught in the try expression or block, and errors raised inside it propagate normally

```lua
local d = try load() catch err then fallback()     -- fallback() only runs if load() errors
local e = try load() catch err then error(`{err}`) -- stringify and propagate err
```

* If an `error()` call is caught by `catch`, its argument is bound to the name which comes after it, so tables and other values can be rethrown with `error(err)` and is visible only in the `catch` part.

```lua
try 
   error({ kind = "NotFound" })
catch err then 
   print(err.kind)  -- NotFound
   error(err)       -- rethrows unchanged
end
```

### Type system

* `try x` has type `T | nil` where `T` is the type of `x`  
* `try x catch e then f` has type `T | U` where `U` is the type of `f`  
* Inside a try catch block, any expression or statement that could raise an exception is treated as a control flow edge from that point to the catch block, for example:

```lua
local function getinventory(): { [string]: Item }
    -- ...
end

local checkitems(_: { [string]: Item })
    -- ...
end

local inventory = nil
try
    inventory = getinventory()
    checkitems(inventory) -- inventory: { [string]: Item }
catch e then
end

-- inventory: { [string]: Item }?, we might have errored during `getinventory()`
```

* The operand is type-checked as usual, so a misspelled member on a typed Instance (`h.Helth`) can still be reported in strict mode  
* The bound error has type `unknown`

### Parsing and compatibility

* `try` and `catch` are contextual keywords, like `continue` and `const`, so existing identifiers with those names still work as previously expected  
* If `try` is parsed as a contextual keyword, the catch-less `try` form is potentially ambiguous with function calls and subtraction. To resolve these ambiguities, we make the decision that if the token following `try` is `(`, `{`, `”`, `’`, or `-` we will parse a function call or subtraction. In all other cases, we parse the relevant `try` node. This preserves backwards compatibility. The cost is that a `try` operand can’t start with `(`, `{`, single quotes, double quotes, or `-`, and adding parenthesis does not help (`try (-x)` is still a call). A workaround could be to write a function that constructs the value and call that with `try`.

```lua
-- Error: This parses as a function call.
local a = try {something_risky(), something_else()}

function create_table()
    return {something_risky(), something_else()}
end

-- Ok
local a = try create_table()
```

## Drawbacks

* `try` catches every error, so it will catch all errors and can’t tell an expected failure from a mistake

```lua
  local function takeDamage()
      local h = workspace.NPC.Humanoid
      h.Helth -= 10 -- typo: should be Health
      return h.Health
  end

  local health = try takeDamage()
```

`health` is `nil` whether the humanoid is missing (expected) or `Helth` is a typo (a bug). The typechecker and lint can help in typed code (i.e. strict mode), and the block form can allow users to inspect the error and rethrow based on specific errors. 

* `try` is a valid identifier today. It is a contextual keyword to avoid breaking existing code, so `try (x)`, `try { ... }`, `try “s”`, and `try -x` keep their current meaning.

## Alternatives

* Do nothing and keep `pcall` and `FindFirstChild` as the existing solutions  
* `?.` that suppresses indexing `nil`  
  * This is a smaller change than this proposal, but it doesn’t help with Instances, the original problem in [\#501](https://github.com/luau-lang/luau/pull/501)  
* `?.` that suppresses all errors  
  * This would work for Instances, but a one-character operator that silently suppresses every error is easy to overuse, not as intuitive as `try` / `catch`, and hard to debug. If we choose to catch all errors, the syntax should explicitly say so.  
* `?.` combined with a `__safeindex` metamethod or a third argument to `__index`  
  * This only works if every access in the chain uses `?.` (`foo.bar?.baz` will still throw if `foo.bar` does). Additionally, this opens the door to `__safecall`, `__safenewindex`, etc. (see [\#501](https://github.com/luau-lang/luau/pull/501) for more details)  
* Requiring `catch` on every `try`  
  * This requires the user to state the failure behavior in every case, which while good coding practice, can increase verbosity and reduce readability:

```lua
local x = if try foo() catch e then nil then try foo() catch e then nil else nil
-- instead of
local x = if try foo() then try bar() else nil
```

* Requiring `else` on the bare `try` form  
  * This eliminates the ambiguity of parsing `try` as a function call or subtraction, since `try <expr> else <expr>` is not valid syntax today. However, it becomes quickly unreadable if nested in an if expression.

```lua
if c then try(x) else foo() end         -- valid today, `else` belongs to the `if`
if local x = try foo() else nil then ... end    -- hard to read in a condition
local x = if try foo() else false then 1 else 2
```

* Adding a glyph such as `?` to the bare `try` form  
  * While this also eliminates parsing ambiguity, it diverges from Luau’s established design pattern of using keywords by introducing a new glyph in value contexts.

```lua
local d = try? a.b.c.d
```

## Prior Art 

* Javascript, Java, C\#, Python, PHP all have `try` / `catch` as statements with a bounded error, and usually `finally`  
* Kotlin and Scala have `try` as an expression: `val x = try { parse(s) } catch (e: Exception) { null }`  
* Swift uses `try?` To turn an error into `nil` (`let x = try? foo()`) which is our catchless form  
  * A `do { try f() } catch { ... }` block binds the error to an implicit `error`  
* Javascript/Typescript have optional chaining with the `?.` operator as well as `??` for nullish coalescing