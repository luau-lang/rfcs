# Interpolated string function calls

## Summary

Allow calling functions with interpolated string literals without parentheses. The call is desugared into a function call with three arguments: the original template string (with interpolation expressions intact), a table of the evaluated values, and a table of byte-offset/length pairs locating each interpolation expression within the template. This enables domain-specific language patterns like structured logging, SQL escaping, and HTML templating.

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

This proposal lifts that restriction with semantics that decompose the interpolated string into its constituent parts, passing them to the called function. This enables the function to process the template, values, and expression metadata however it sees fit, without needing to implement its own string parser.

### Structured logging

The primary motivating use case is structured logging. Production log consumption systems have two key requirements:

1. **Human-readable template strings** for aggregation. Logs need to be grouped by message pattern so that operators can see, for example, "200 instances/minute of '{foo} occurred after {bar}'" rather than an opaque format string or fingerprint.

2. **Key-value structured data** for search and filtering. Given a template like `{user.name} logged in from {ip}`, the consuming system needs both the expression names (`user.name`, `ip`) and their runtime values (`"Alice"`, `"10.0.0.1"`) to enable queries like "show me all logs where user.name = 'Alice'".

This proposal satisfies both requirements. The template string is passed directly as the first argument, and the byte offsets allow consumers to extract expression names and associate them with values:

```luau
local a = 1
local function double(x: number)
  return x * 2
end

-- Desugars to: log:Info("The double of {a} is {double(a)}", {a, double(a)}, {{15, 3}, {22, 11}})
log:Info `The double of {a} is {double(a)}`
```

The template `"The double of {a} is {double(a)}"` serves directly as an aggregation key, and a consumer can extract the expression names `"a"` and `"double(a)"` via the byte offsets.

Without this feature, developers must either manually construct all arguments (tedious and error-prone) or use a logging library that implements its own template parsing at runtime (duplicating language functionality).

### SQL escaping

Interpolated string calls enable safe, ergonomic SQL query construction where the function can automatically escape interpolated values:

```luau
local sqlite = require("luau_sqlite")

local function takeUserInput(db: sqlite.DB, user: string, comment: string)
  -- Auto-escape inputs to guard against SQL injection
  db:Exec `INSERT INTO user_inputs (user, comment) VALUES ({user}, {comment})`
  -- Desugars to: db:Exec("INSERT INTO user_inputs (user, comment) VALUES ({user}, {comment})", {user, comment}, {{47, 6}, {55, 9}})
end
```

### HTML templating

Similarly, HTML renderers can automatically escape interpolated values to prevent XSS attacks:

```luau
local tmpl = require("luau_html_renderer")

local function renderPage(r: tmpl.Renderer, userinput: string)
  -- Automatic XSS protection through escaping
  return tmpl.HTML `The user asked about {userinput}`
  -- Desugars to: tmpl.HTML("The user asked about {userinput}", {userinput}, {{22, 11}})
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

When a function is called with an interpolated string literal in this style, the compiler desugars the call into a function call with three arguments:

1. **Template string**: The original template with interpolation expressions intact, e.g. `"The double of {a} is {double(a)}"`
2. **Values table**: A sequential table of the evaluated interpolation values, e.g. `{a, double(a)}`
3. **Offsets table**: A sequential table of `{offset, length}` pairs, where each pair gives the 1-based byte offset and length of the corresponding `{expression}` span (including braces) within the template string

The template string and offsets table are determined at compile time. The values table is evaluated at runtime.

A consumer can extract the expression text for the i-th interpolation using the offsets:

```luau
-- Extract {expr} including braces
local span = string.sub(template, offsets[i][1], offsets[i][1] + offsets[i][2] - 1)
-- Extract just the expression name (strip braces)
local expr = string.sub(template, offsets[i][1] + 1, offsets[i][1] + offsets[i][2] - 2)
```

For method calls (`:` syntax), `self` is passed first as usual, followed by these three arguments.

If the interpolated string contains no interpolation expressions, the call passes a single string argument (the literal string), identical to a regular string literal call.

### Examples

```luau
local id = 42
local user = {name = "Alice"}

-- Simple identifier
log:Info `Processing item {id}`
-- Desugars to: log:Info("Processing item {id}", {42}, {{17, 4}})

-- Member expression
log:Info `User {user.name} logged in`
-- Desugars to: log:Info("User {user.name} logged in", {"Alice"}, {{6, 11}})

-- Multiple expressions
log:Info `{user.name} is processing item {id}`
-- Desugars to: log:Info("{user.name} is processing item {id}", {"Alice", 42}, {{1, 11}, {33, 4}})
```

### No expression restrictions

This design uses positional tables, so **any expression valid in a regular interpolated string is also valid here**, including function and method calls:

```luau
-- Function calls are allowed
log:Info `Result is {compute()}`
-- Desugars to: log:Info("Result is {compute()}", {compute()}, {{11, 11}})

-- Method calls are allowed
log:Info `Name is {user:getName()}`
-- Desugars to: log:Info("Name is {user:getName()}", {user:getName()}, {{9, 17}})

-- Repeated expressions are fine (each is a separate positional entry)
log:Info `{increment()} and then {increment()}`
-- Desugars to: log:Info("{increment()} and then {increment()}", {increment(), increment()}, {{1, 13}, {25, 13}})
```

Since values are stored positionally rather than keyed by expression text, there is no ambiguity when the same expression appears multiple times or when expressions have side effects.

### Behavior with variadic functions

Functions that accept variadic arguments will receive the three desugared arguments. For example, `print` concatenates its arguments with spaces:

```luau
local name = "Alice"
print `Hello {name}`
-- Desugars to: print("Hello {name}", {"Alice"}, {{7, 6}})
-- Output: Hello {name} table: 0x... table: 0x...
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
function log(template, vals, offsets)
  return function(context)
    -- The template (e.g. "Hello {name}") is already a human-readable aggregation key.
    -- Build the formatted message by replacing {expr} spans with values.
    local message = template
    for i = #offsets, 1, -1 do
      local pos, len = offsets[i][1], offsets[i][2]
      message = string.sub(message, 1, pos - 1) .. tostring(vals[i]) .. string.sub(message, pos + len)
    end
    emit(message, template, vals, context)
  end
end

-- No parentheses anywhere -- log is called with the desugared
-- interpolated string and returns a function, which is then called
-- with the context table
log `Hello {name}` {userId = 12345, region = "us-east"}

-- Desugars to: log("Hello {name}", {"Alice"}, {{7, 6}})({userId = 12345, region = "us-east"})
```

This approach requires no special grammar support. It is a natural composition of a parentheses-free interpolated string call (which returns a function) and a parentheses-free table literal call on that returned function.

### Bridging to existing functions with a `Message` type

A `Message` wrapper type can bridge the gap between this calling convention and existing functions like `print` or `warn` that expect string arguments:

```luau
local Message = {}
Message.__index = Message

function Message.new(template, vals, offsets)
  return setmetatable({
    template = template,
    values = vals,
    offsets = offsets,
  }, Message)
end

function Message:__tostring()
  local result = self.template
  for i = #self.offsets, 1, -1 do
    local pos, len = self.offsets[i][1], self.offsets[i][2]
    result = string.sub(result, 1, pos - 1) .. tostring(self.values[i]) .. string.sub(result, pos + len)
  end
  return result
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
  emit(tostring(msg), msg.template, msg.values, msg.offsets)
end
```

While not part of this proposal, this pattern also opens the door for existing functions to be updated to special-case a single `Message` argument, extracting structured data for telemetry while preserving their current behavior for all other callers.

## Drawbacks

### Increased complexity in the grammar

Adding interpolated strings as a parentheses-free call argument adds complexity to the parser. However, the grammar change is minimal and unambiguous.

### Learning curve

Developers need to understand that interpolated string calls without parentheses behave differently from parenthesized calls. This could be confusing:

```luau
log:Info `Hello {name}`        -- passes 3 arguments (template, values, offsets)
log:Info(`Hello {name}`)       -- passes 1 argument (formatted string)
```

This distinction is intentional and valuable for DSL use cases, but documentation will need to clearly explain the difference.

### Surprising behavior with existing functions

Existing functions that are not designed for this calling convention will receive unexpected arguments if called without parentheses. For example, `print` and `warn` accept variadic arguments and would print the template string, values table, and offsets table alongside each other rather than producing a formatted message.

Functions designed for parentheses-free interpolated string calls would need to be written (or updated, if possible) to accept the three-argument format. When an existing function cannot be updated in a non-breaking way (for example, because it accepts variadic arguments), a pattern such as the `Message` wrapper type described in the Design section can be used instead to bridge the gap.

## Alternatives

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

Instead of byte offsets, the third argument could be a sequential table of expression source texts:

```luau
log:Info `Hello {name}`
-- Would pass: log:Info("Hello %*", {"Alice"}, {"name"})
```

This is slightly more convenient for consumers who only need expression names, but requires the language specification to define normalization rules for source text (whitespace trimming, quote handling for literals, etc.). The byte-offset approach avoids this complexity by letting consumers extract whatever they need directly from the original template string.

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

This was considered and the proposed design shares the same spirit: decomposing the interpolated string into parts that don't require the consumer to parse Luau expressions. The proposed design differs in preserving the original template string (useful as an aggregation key) and providing byte offsets for flexible extraction of expression metadata.

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
