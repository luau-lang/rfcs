# Interpolated string function calls

## Summary

Allow calling functions with interpolated string literals without parentheses. The call is desugared into a function call with two arguments: the original template string (with interpolation expressions intact) and a table of the evaluated values. Two new core library functions are also introduced: one for extracting the stringified interpolation expressions from a template, and one for rendering a template with values. These reuse the same code as normal string interpolation. Together, this enables domain-specific language patterns like structured logging, SQL escaping, and HTML templating.

## Motivation

Luau currently supports function calls without parentheses for string literals and table literals:

```luau
print "hello"        -- equivalent to print("hello")
print { 1, 2, 3 }    -- equivalent to print({1, 2, 3})
```

When string interpolation was introduced, this calling style was explicitly prohibited for interpolated strings:

```luau
local name = "world"
print `Hello {name}`  -- currently a parse error
```

The [string interpolation RFC](https://github.com/luau-lang/rfcs/blob/master/docs/syntax-string-interpolation.md) noted this restriction was "likely temporary while we work through string interpolation DSLs."

This proposal lifts that restriction with semantics that decompose the interpolated string into its constituent parts, passing them to the called function. This enables the function to process the template and values however it sees fit, and to recover expression metadata via core library functions.

### Structured logging

The primary motivating use case is structured logging. Production log consumption systems have two key requirements:

1. **Human-readable template strings** for aggregation. Logs need to be grouped by message pattern so that operators can see, for example, "200 instances/minute of '{foo} occurred after {bar}'" rather than an opaque format string or fingerprint.

2. **Key-value structured data** for search and filtering. Given a template like `{user.name} logged in from {ip}`, the consuming system needs both the expression names (`user.name`, `ip`) and their runtime values (`"Alice"`, `"10.0.0.1"`) to enable queries like "show me all logs where user.name = 'Alice'".

This proposal satisfies both requirements. The template string is passed directly as the first argument, and the core library function `string.interpparse` allows consumers to extract expression names and associate them with values:

```luau
local a = 1
local function double(x: number)
  return x * 2
end

-- Desugars to: log:Info("The double of {a} is {double(a)}", {a, double(a)})
log:Info `The double of {a} is {double(a)}`
```

The template `"The double of {a} is {double(a)}"` serves directly as an aggregation key, and a consumer can extract the expression names `"a"` and `"double(a)"` via `string.interpparse`.

These template strings are substantially more useful than format strings with opaque `%*` placeholders. Compare these two log call sites:

```luau
log `{query} returned {result} in {duration}ms`
log `{endpoint} returned {status} in {duration}ms`
```

With template strings, these aggregate separately, so operators can distinguish database queries from HTTP requests. With `%*` format strings, both produce `"%* returned %* in %*ms"` and would merge into a single bucket. Call stack metadata (e.g. fingerprints) could be used to disambiguate, but the resulting aggregation keys are opaque and require a source code lookup to interpret.

Template strings also make aggregated log patterns self-documenting. An operator seeing a pattern like `"%*: %* %*:%* -> %*:%* proto=%* bytes=%*"` in a dashboard cannot map the first four `%*` to meaning without a source code lookup. The template `"{rule_id}: {action} {src_ip}:{src_port} -> {dst_ip}:{dst_port} proto={proto} bytes={bytes}"` is immediately interpretable.

Without this feature, developers must either manually construct all arguments (tedious and error-prone) or use a logging library that implements its own template parsing at runtime (duplicating language functionality).

### SQL escaping

Interpolated string calls enable safe, ergonomic SQL query construction where the function can automatically escape interpolated values:

```luau
local sqlite = require("luau_sqlite")

local function takeUserInput(db: sqlite.DB, user: string, comment: string)
  -- Auto-escape inputs to guard against SQL injection
  db:Exec `INSERT INTO user_inputs (user, comment) VALUES ({user}, {comment})`
  -- Desugars to: db:Exec("INSERT INTO user_inputs (user, comment) VALUES ({user}, {comment})", {user, comment})
end
```

### HTML templating

Similarly, HTML renderers can automatically escape interpolated values to prevent XSS attacks:

```luau
local tmpl = require("luau_html_renderer")

local function renderPage(r: tmpl.Renderer, userinput: string)
  -- Automatic XSS protection through escaping
  return tmpl.HTML `The user asked about {userinput}`
  -- Desugars to: tmpl.HTML("The user asked about {userinput}", {userinput})
end
```

## Design

### Grammar

The grammar for function calls is extended to allow an interpolated string as the argument:

```
functioncall ::= prefixexp args
args ::= '(' [explist] ')' | tableconstructor | LiteralString | stringinterp
```

### Semantics

When a function is called with an interpolated string literal in this style, the compiler desugars the call into a function call with two arguments:

1. **Template string**: The original template with interpolation expressions intact, e.g. `"The double of {a} is {double(a)}"`
2. **Values table**: A sequential table of the evaluated interpolation values, e.g. `{a, double(a)}`

The template string is determined at compile time. The values table is evaluated at runtime.

For method calls (`:` syntax), `self` is passed first as usual, followed by these two arguments.

If the interpolated string contains no interpolation expressions, the call passes a single string argument (the literal string), identical to a regular string literal call.

### Core library functions

Two new functions are added to the `string` library to support consumers of interpolated string calls. Both reuse the same code as the compiler's normal string interpolation implementation.

#### `string.interpparse`

```luau
string.interpparse(template: string) -> {string}
```

Parses an interpolation template and returns a sequential table of the stringified interpolation expressions.

```luau
string.interpparse("Hello {name}")          -- returns {"name"}
string.interpparse("{a} + {b} = {a + b}")   -- returns {"a", "b", "a + b"}
string.interpparse("No interpolations")     -- returns {}
```

#### `string.interp`

```luau
string.interp(template: string, values: {any}) -> string
```

Renders an interpolation template by replacing each `{expression}` with the `tostring` of the corresponding value from the values table, matching the behavior of normal string interpolation. The values table may be either sequential (positional matching) or associative with string keys (matching by expression name):

```luau
-- Sequential: values matched positionally to expressions
string.interp("Hello {name}", {"Alice"})          -- returns "Hello Alice"
string.interp("{a} + {b} = {a + b}", {1, 2, 3})   -- returns "1 + 2 = 3"

-- Associative: values matched by expression name
string.interp("Hello {name}", {name = "Alice"})   -- returns "Hello Alice"
```

Supporting both sequential and associative tables enables a single function to serve as both a parentheses-free interpolated string call target and a conventional function accepting a template with a named context object:

```luau
function log_info(self, template, values)
    local exprs = string.interpparse(template)
    local rendered = string.interp(template, values)
    -- exprs[i] is the expression name, values[i] is the runtime value
    -- rendered is the fully formatted string
end

-- Paren-free: compiler fills in values positionally from local variables
log:Info `Hello {name}`
-- Desugars to: log:Info("Hello {name}", {"Alice"})

-- Parenthesized: caller provides a named context object
log:Info("Hello {name}", {name = "Alice"})
```

Note that calling `string.interpparse` at runtime means the template is parsed twice: once by the compiler when desugaring the interpolated string call (to identify expressions and evaluate them), and a second time by the library function. This is a minor cost, as template strings are typically short. The implementation of `string.interpparse` can also cache results internally, since template strings are compile-time constants and strings are interned in Luau, making repeated calls with the same template effectively free.

### Examples

```luau
local id = 42
local user = {name = "Alice"}

-- Simple identifier
log:Info `Processing item {id}`
-- Desugars to: log:Info("Processing item {id}", {42})

-- Member expression
log:Info `User {user.name} logged in`
-- Desugars to: log:Info("User {user.name} logged in", {"Alice"})

-- Multiple expressions
log:Info `{user.name} is processing item {id}`
-- Desugars to: log:Info("{user.name} is processing item {id}", {"Alice", 42})
```

### No expression restrictions

This design uses positional tables, so **any expression valid in a regular interpolated string is also valid here**, including function and method calls:

```luau
-- Function calls are allowed
log:Info `Result is {compute()}`
-- Desugars to: log:Info("Result is {compute()}", {compute()})

-- Method calls are allowed
log:Info `Name is {user:getName()}`
-- Desugars to: log:Info("Name is {user:getName()}", {user:getName()})

-- Repeated expressions are fine (each is a separate positional entry)
log:Info `{increment()} and then {increment()}`
-- Desugars to: log:Info("{increment()} and then {increment()}", {increment(), increment()})
```

Since values are stored positionally rather than keyed by expression text, there is no ambiguity when the same expression appears multiple times or when expressions have side effects.

### Behavior with variadic functions

Functions that accept variadic arguments will receive the two desugared arguments. For example, `print` concatenates its arguments with spaces:

```luau
local name = "Alice"
print `Hello {name}`
-- Desugars to: print("Hello {name}", {"Alice"})
-- Output: Hello {name} table: 0x...
```

This is likely not the desired output. Developers wanting simple string interpolation should use parentheses:

```luau
print(`Hello {name}`)  -- Output: Hello Alice
```

The parentheses-free form is intended for functions specifically designed to receive the expanded arguments.

### Interaction with existing syntax

This feature does not change the behavior of:

- Regular string literals: `print "hello"` continues to pass a single string argument
- Table literals: `print {1, 2, 3}` continues to pass a single table argument
- Parenthesized calls: `print(`hello {name}`)` continues to pass a single formatted string

The new behavior only applies to calls without parentheses using interpolated strings.

### Passing additional context via currying

Some use cases (e.g. structured logging) benefit from passing additional context alongside the interpolated string. Rather than introducing special syntax for an extra argument, this can be achieved through currying (a function that returns a function):

```luau
-- log accepts the desugared interpolated string arguments and returns
-- a function that accepts additional context
function log(template, vals)
  return function(context)
    local rendered = string.interp(template, vals)
    emit(rendered, template, vals, context)
  end
end

-- No parentheses anywhere -- log is called with the desugared
-- interpolated string and returns a function, which is then called
-- with the context table
log `Hello {name}` {userId = 12345, region = "us-east"}

-- Desugars to: log("Hello {name}", {"Alice"})({userId = 12345, region = "us-east"})
```

This approach requires no special grammar support. It is a natural composition of a parentheses-free interpolated string call (which returns a function) and a parentheses-free table literal call on that returned function.

### Bridging to existing functions with a `Message` type

A `Message` wrapper type can bridge the gap between this calling convention and existing functions like `print` or `warn` that expect string arguments:

```luau
local Message = {}
Message.__index = Message

function Message.new(template, vals)
  return setmetatable({
    template = template,
    values = vals,
  }, Message)
end

function Message:__tostring()
  return string.interp(self.template, self.values)
end
```

Because `Message` defines `__tostring`, existing functions that convert arguments to strings will produce the expected formatted output:

```luau
-- Message.new receives the desugared arguments and returns a Message object
-- print() calls tostring on it, producing "Hello Alice"
print(Message.new `Hello {name}`)

-- warn works the same way
warn(Message.new `User {user.name} failed to log in`)
```

Structured logging APIs can also inspect the Message's fields directly:

```luau
function structured_log(msg: Message)
  local exprs = string.interpparse(msg.template)
  emit(tostring(msg), msg.template, msg.values, exprs)
end
```

While not part of this proposal, this pattern also opens the door for existing functions to be updated to special-case a single `Message` argument, extracting structured data for telemetry while preserving their current behavior for all other callers.

## Drawbacks

### Increased complexity in the grammar

Adding interpolated strings as a parentheses-free call argument adds complexity to the parser. However, the grammar change is minimal and unambiguous.

### Learning curve

Developers need to understand that interpolated string calls without parentheses behave differently from parenthesized calls. This could be confusing:

```luau
log:Info `Hello {name}`        -- passes 2 arguments (template, values)
log:Info(`Hello {name}`)       -- passes 1 argument (formatted string)
```

This distinction is intentional and valuable for DSL use cases, but documentation will need to clearly explain the difference.

### Surprising behavior with existing functions

Existing functions that are not designed for this calling convention will receive unexpected arguments if called without parentheses. For example, `print` and `warn` accept variadic arguments and would print the template string and values table rather than producing a formatted message.

Functions designed for parentheses-free interpolated string calls would need to be written (or updated, if possible) to accept the two-argument format. When an existing function cannot be updated in a non-breaking way (for example, because it accepts variadic arguments), a pattern such as the `Message` wrapper type described in the Design section can be used instead to bridge the gap.

### Double parsing of template strings

When a consumer calls `string.interpparse`, the template string is parsed twice: once by the compiler when desugaring the call, and once at runtime by the library function. While this cost is minor for typical template strings, it is inherently redundant. The [three-argument desugaring with byte offsets](#three-argument-desugaring-with-byte-offsets) alternative avoids this by providing expression location data at compile time.

## Alternatives

### Three-argument desugaring with byte offsets

Instead of two arguments and core library functions, the compiler could desugar the call into three arguments: the template string, the values table, and a table of byte-offset/length pairs locating each interpolation expression within the template:

```luau
log:Info `The double of {a} is {double(a)}`
-- Would desugar to: log:Info("The double of {a} is {double(a)}", {a, double(a)}, {{15, 3}, {22, 11}})
```

The third argument is a sequential table of `{offset, length}` pairs, where each pair gives the 1-based byte offset and length of the corresponding `{expression}` span (including braces) within the template string. Both the template string and offsets table are determined at compile time; only the values table is evaluated at runtime.

A consumer would extract expression text using the offsets:

```luau
-- Extract {expr} including braces
local span = string.sub(template, offsets[i][1], offsets[i][1] + offsets[i][2] - 1)
-- Extract just the expression name (strip braces)
local expr = string.sub(template, offsets[i][1] + 1, offsets[i][1] + offsets[i][2] - 2)
```

This approach avoids the double-parsing drawback since all expression metadata is provided at compile time. However, it passes a more complex calling convention (three arguments instead of two) and requires every consumer to manually perform byte arithmetic to extract expression names. Additionally, reconstructing the formatted string requires the consumer to implement its own replacement loop using the byte offsets rather than calling a single library function.

### Interpolation metadata object syntax

Rather than overloading the parentheses-free calling convention, a new prefix syntax could produce an interpolation metadata object that is used with normal parenthesized function calls:

```luau
local name = "Alice"
local msg = @`Hello {name}`
-- msg is an object like: { template = "Hello {name}", values = {"Alice"}, expressions = {"name"} }

log:Info(@`The double of {a} is {double(a)}`)
db:Exec(@`SELECT * FROM users WHERE id = {userId}`)
```

The `@` prefix before an interpolated string would evaluate to a structured object containing the template, values, and parsed expression names. This object could be passed to any function using normal parenthesized call syntax, stored in variables, placed in data structures, etc.

This approach avoids the learning-curve issue of parenthesized vs. non-parenthesized calls having different semantics for interpolated strings. It also makes the intent explicit at the call site. However, it introduces new syntax (`@` prefix) for a feature that is also achievable through the parentheses-free calling convention, and it loses the concise DSL aesthetic of `log `message`` in favor of `log(@`message`)`.

### Manual format string with curried values

As noted by reviewers, a similar pattern is already achievable without new syntax using currying with a format string and a values table:

```luau
local function Log(fmt: string)
  return function(args: { any })
    local message = string.format(fmt, table.unpack(args))
    print("Log:", message, "Format:", fmt)
  end
end

local user = "Bottersnike"
Log "Hello, %*" { user }
```

While this works for simple cases, it has significant limitations:

1. The developer must manually write the format string separately from the values, losing the ergonomic benefits of interpolated string syntax and introducing a risk of the two getting out of sync
2. The format string uses `%*` placeholders rather than the original expression names, so it cannot serve as a human-readable aggregation key (e.g., `"Hello, %*"` vs. `"Hello, {user}"`)
3. There is no way to recover expression names for structured key-value logging

This RFC automates the decomposition that developers would otherwise have to do by hand, while preserving the original template and expression metadata.

### Expressions list instead of byte offsets

Instead of core library functions, the compiler could pass expression source texts directly as a third argument:

```luau
log:Info `Hello {name}`
-- Would pass: log:Info("Hello {name}", {"Alice"}, {"name"})
```

This avoids the double-parsing drawback of the core library function approach and is slightly more convenient for consumers who only need expression names. However, it requires the language specification to define normalization rules for source text (whitespace trimming, quote handling for literals, etc.) and adds a third argument to the calling convention. The core library function approach handles normalization in the library implementation without burdening the calling convention.

### Tagged interpolated strings (JavaScript-style)

JavaScript's tagged template literals pass an array of string parts and the interpolated values as separate arguments:

```javascript
tag`Hello ${name}, you have ${count} messages`;
// Calls: tag(["Hello ", ", you have ", " messages"], name, count)
```

A Luau adaptation would pass two tables, string parts and values:

```luau
tag `Hello {name}, you have {count} messages`
-- Would call: tag({"Hello ", ", you have ", " messages"}, {name, count})
```

This was considered and the proposed design shares the same spirit: decomposing the interpolated string into parts that don't require the consumer to parse Luau expressions. The proposed design differs in preserving the original template string (useful as an aggregation key) and providing library functions for flexible extraction of expression metadata.

### Interpolation table with named keys

An earlier version of this RFC proposed passing a table mapping expression names to their values:

```luau
log:Info `Hello {name}`
-- Would pass: log:Info("Hello Alice", "Hello {name}", {name = "Alice"})
```

This was rejected because:

1. It requires consumers to parse the `{...}` syntax in the template string to reconstruct the formatted output
2. Complex expressions like `{a + b}` or `{user.name}` create awkward or ambiguous table keys
3. Function and method calls must be restricted because repeated calls (e.g. `{increment()}` appearing twice) can produce different values for the same key
4. Metamethods on property access can cause the same issues as function calls, as noted by reviewers

The positional values table avoids all of these problems.

### Extra context argument in grammar

An earlier version proposed allowing an optional trailing table literal after the interpolated string:

```luau
log:Info `Hello {name}`, {userId = 12345}
```

This was rejected because it adds ambiguity to the grammar and is overly specific to the logging use case. The currying pattern described in the Design section provides equivalent functionality without grammar changes.

## References

- [String interpolation RFC](https://github.com/luau-lang/rfcs/blob/master/docs/syntax-string-interpolation.md) - The original RFC that introduced interpolated strings and noted the temporary restriction
