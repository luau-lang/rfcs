# try catch

## Summary

Adds `try` / `catch` construct at the statement and expression level. `try` constructs allow you to evaluate code and intercept errors that occur during execution with a `catch`.

```lua
-- Expression form: try <expr> catch <name> then <expr>
local config = try loadConfig() catch err then DEFAULT_CONFIG
local config = try loadConfig() catch err then f(err)

-- Block form: try <block> catch <name> then <block> end
try
   local config = loadConfig()
   applyConfig(config)
catch err then
   warn(`couldn't apply config: {err}`)
end
```

## Motivation

`pcall` and `xpcall` are currently the standard way to catch errors in Luau. They work, but can only protect a function call, so any code that isn’t already a single call has to be wrapped in a closure. That closure can’t `return`, `break`, or `continue` on the surrounding code, results have to be passed back out through return values or outer locals, and it easily becomes hard for the reader and the type checker to follow what happens inside the call. We want a concise and intuitive way to handle errors that reads like the rest of the code.

`pcall` is also difficult to type check. `pcall` returns multiple values: a boolean which indicates whether the operation succeeded, plus either the return values of the protected function or the error. This means that the type of one variable depends on the value of another, which Luau can’t express.

### Today

The protected code goes in a closure, so `break`, `continue`, and `return` inside it can't reach the surrounding loop or function. Results are passed out through outer locals, which the type checker can't connect to `ok`:

```lua
for _, url in urls do
    local weather: Weather? = nil
    local ok, err = pcall(function()
        weather = parseWeather(fetch(url)) -- can't `continue` the loop from in here
    end)
    if not ok then
        print(`request failed: {err}`)
        continue
    end
    print(weather.temp) -- type error: `weather` could be nil
end
```

### With `try`

```lua
for _, url in urls do
    local weather: Weather? = nil
    try
        weather = parseWeather(fetch(url))
    catch err then
        print(`request failed: {err}`)
        continue
    end
    print(weather.temp) -- ok: `weather` was assigned
end
```

For a single call with a fallback, the expression form can further simplify the code and avoid the extra local and second return value:

```lua
local weather = try parseWeather(body) catch err then DEFAULT_WEATHER
```

Errors can still be inspected and rethrown, similar to `pcall`:

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

There are two forms: In both, `catch` is required and always binds the error.

```
-- Expression form
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
exp  ::= ... | 'try' exp 'catch' Name 'then' exp
stat ::= ... | 'try' block 'catch' Name 'then' block 'end'
```

### Semantics

* `try` catches the same errors `pcall` does

```lua
local a = try error("error!") catch e then 0 -- a is 0, error() raised an error

local t = nil
local b = try t.field catch e then 0         -- b is 0, indexing nil raised an error
```

* Like `pcall`, code inside `try` can also yield  
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
   try
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

* If an `error()` call is caught by `catch`, its argument is bound to the name which comes after it, so tables and other values can be rethrown with `error(err`. The name is visible only in the `catch` part. The bound value is the same one `pcall` would return, so string errors include the same position information (e.g. `script:12: message`).

```lua
try 
   error({ kind = "NotFound" })
catch err then 
   print(err.kind)  -- NotFound
   error(err)       -- rethrows unchanged
end
```

### Type system

* `try x catch e then f` has type `T | U` where `T` is the type of `x` and `U` is the type of `f`  
* The bound error has type `unknown`. Any operation inside the `try`, including calls into other functions, metamethods, and host functions, can raise an error of any type, so it can’t be narrowed in general. It could be refined with `typeof` like any other `unknown` value.  
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

* After a try catch block, the type of a local is the union of its type at the end of the `try` block and at the end of the `catch` block. Since the `catch` block may run before an assignment in the `try` block, a local that’s only assigned in the `try` block is optional afterward. If the `catch` block returns, breaks, or rethrows, only the `try` block contributes.

```lua
local config
try
    config = loadConfig()
catch err then
    warn(err)
    return
end
-- config: Config
```

* `try` doesn't change how the code inside it is type checked. Errors that would be reported outside a `try` are still reported inside one, such as a misspelled member (`h.Helth`) or indexing a value that could be `nil`, even though the error would be caught at runtime.

### Parsing and compatibility

* `try` and `catch` are contextual keywords, like `continue` and `const`, so existing identifiers with those names still work as previously expected  
* In the expression form, `try` followed by `(`, `{`, `"`, `'`, `` ` ``, or `-` is potentially ambiguous with function calls and subtraction. Since `catch` is required, we can resolve this by checking whether `catch <name> then` follows the operand, which never happens in existing code. The parser checkpoints at `try` and, if no `catch` follows, rewinds and parses a function call or subtraction as before. This preserves backwards compatibility, and lets the operand start with any token:

```
local some = try { result = someThingThatCanFail(), requestId = nextId() } catch _ then table.clone(default)
local hash = try `{getHashPortionThatCanFail()}:{nextId()}` catch _ then defaultHash
```

* In the block form, a statement starting with `try` is only valid today as a function call or assignment. If `try` is followed by `(`, `[`, `.`, `:`, `{`, a string, `=`, `,`, or a compound assignment operator, we parse it as before. In all other cases, we parse a `try` block. The cost is that the first statement in a `try` block can't start with `(`, which can be worked around by wrapping it in `do ... end`

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

`health` is `nil` whether `takeDamage` failed for an expected reason or `Helth` is a typo (a bug). The typechecker and lint can help in typed code (i.e. strict mode), and the block form can allow users to inspect the error and rethrow based on specific errors, which is the same tradeoff `pcall` has today.

* `try` is a valid identifier today. It is a contextual keyword to avoid breaking existing code, so `try (x)`, `try { ... }`, `try “s”`, and `try -x` keep their current meaning.

## Alternatives

* Do nothing and keep `pcall` as the existing solution.

## Prior Art 

* Javascript, Java, C\#, Python, PHP all have `try` / `catch` as statements with a bounded error, and usually `finally`  
* Kotlin and Scala have `try` as an expression: `val x = try { parse(s) } catch (e: Exception) { null }`  
* Swift has a `do { try f() } catch { ... }` block that binds the error to an implicit `error`, and `try?` to turn an error into `nil` (`let x = try? foo()`)