# Interpolated string function calls

## Summary

Allow calling functions with interpolated string literals without parentheses. The call is desugared into a function call with three arguments: a format string with `%s` placeholders, a table of the evaluated interpolation values, and a table of the original expression source texts. This enables domain-specific language patterns like structured logging, SQL escaping, and HTML templating.

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

The primary motivating use case is structured logging, where it is valuable to capture both the formatted message template and its interpolation values for machine processing:

```luau
local a = 1
local function double(x: number)
  return x * 2
end

-- Desugars to `log:Info("The double of %s is %s", {a, double(a)}, {"a", "double(a)"})`
-- Note that the second argument evalutes to {1, 2} at runtime, but not at the desugaring step.
log:Info `The double of {a} is {double(a)}`
```

The template string enables aggregating logs by message pattern (e.g. grouping all "The double of %s is %s" messages), while the values and expression names provide searchable structured data for platforms like Splunk, Datadog, or Elasticsearch.

Without this feature, developers must either manually construct all arguments (tedious and error-prone) or use a logging library that implements its own template parsing at runtime (duplicating language functionality).

### SQL escaping

Interpolated string calls enable safe, ergonomic SQL query construction where the function can automatically escape interpolated values:

```luau
local sqlite = require("luau_sqlite")

local function takeUserInput(db: sqlite.DB, user: string, comment: string)
  -- Auto-escape inputs to guard against SQL injection
  db:Exec `INSERT INTO user_inputs (user, comment) VALUES ({user}, {comment})`
  -- Desugars to: db:Exec("INSERT INTO user_inputs (user, comment) VALUES (%s, %s)", {user, comment}, {"user", "comment"})
end
```

### HTML templating

Similarly, HTML renderers can automatically escape interpolated values to prevent XSS attacks:

```luau
local tmpl = require("luau_html_renderer")

local function renderPage(r: tmpl.Renderer, userinput: string)
  -- Automatic XSS protection through escaping
  return tmpl.HTML `The user asked about {userinput}`
  -- Desugars to: tmpl.HTML("The user asked about %s", {userinput}, {"userinput"})
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

1. **Format string**: The template with each `{expression}` replaced by `%s`, e.g. `"The double of %s is %s"`
2. **Values table**: A sequential table of the interpolation values, e.g. `{a, double(a)}`
3. **Expressions table**: A sequential table of the original expression source texts, e.g. `{"a", "double(a)"}`

The format string and expressions table are determined at compile time. The values table is evaluated at runtime.

For method calls (`:` syntax), `self` is passed first as usual, followed by these three arguments.

If the interpolated string contains no interpolation expressions, the call passes a single string argument (the literal string), identical to a regular string literal call.

### Examples

```luau
local id = 42
local user = {name = "Alice"}

-- Simple identifier
log:Info `Processing item {id}`
-- Desugars to: log:Info("Processing item %s", {42}, {"id"})

-- Member expression
log:Info `User {user.name} logged in`
-- Desugars to: log:Info("User %s logged in", {"Alice"}, {"user.name"})

-- Multiple expressions
log:Info `{user.name} is processing item {id}`
-- Desugars to: log:Info("%s is processing item %s", {"Alice", 42}, {"user.name", "id"})
```

### No expression restrictions

Unlike some alternative designs that use named keys for interpolated values, this design uses positional tables. This means **any expression valid in a regular interpolated string is also valid here**, including function and method calls:

```luau
-- Function calls are allowed
log:Info `Result is {compute()}`
-- Desugars to: log:Info("Result is %s", {compute()}, {"compute()"})

-- Method calls are allowed
log:Info `Name is {user:getName()}`
-- Desugars to: log:Info("Name is %s", {user:getName()}, {"user:getName()"})

-- Repeated expressions are fine (each is a separate positional entry)
log:Info `{increment()} and then {increment()}`
-- Desugars to: log:Info("%s and then %s", {increment(), increment()}, {"increment()", "increment()"})
```

Since values are stored positionally rather than keyed by expression text, there is no ambiguity when the same expression appears multiple times or when expressions have side effects.

### Behavior with variadic functions

Functions that accept variadic arguments will receive the three desugared arguments. For example, Roblox's `print` concatenates its arguments with spaces:

```luau
local name = "Alice"
print `Hello {name}`
-- Desugars to: print("Hello %s", {"Alice"}, {"name"})
-- Output: Hello %s table: 0x... table: 0x...
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
function log(fmt, vals, exprs)
  return function(context)
    -- Reconstruct the formatted message
    local message = string.format(fmt, table.unpack(vals))
    -- Reconstruct a human-readable template using the expression names
    local wrapped = table.create(#exprs)
    for i, expr in exprs do
      wrapped[i] = "{" .. expr .. "}"
    end
    local template = string.format(fmt, table.unpack(wrapped))
    -- template is now "Hello {name}" -- the original template string.
    emit(message, template, vals, exprs, context)
  end
end

-- No parentheses anywhere -- log is called with the desugared
-- interpolated string and returns a function, which is then called
-- with the context table
log `Hello {name}` {userId = 12345, region = "us-east"}

-- Desugars to: log("Hello %s", {"Alice"}, {"name"})({userId = 12345, region = "us-east"})
```

This approach requires no special grammar support. It is a natural composition of a parentheses-free interpolated string call (which returns a function) and a parentheses-free table literal call on that returned function.

### Bridging to existing functions with a `Message` type

A `Message` wrapper type can bridge the gap between this calling convention and existing functions like `print` or `warn` that expect string arguments:

```luau
local Message = {}
Message.__index = Message

function Message.new(fmt, vals, exprs)
  return setmetatable({
    fmt = fmt,
    values = vals,
    expressions = exprs,
  }, Message)
end

function Message:__tostring()
  return string.format(self.fmt, table.unpack(self.values))
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
  emit(tostring(msg), msg.fmt, msg.values, msg.expressions)
end
```

While not part of this proposal, this pattern also opens the door for existing functions to be updated to special-case a single `Message` argument, extracting structured data for telemetry while preserving their current behavior for all other callers.

## Drawbacks

### Increased complexity in the grammar

Adding interpolated strings as a parentheses-free call argument adds complexity to the parser. However, the grammar change is minimal and unambiguous.

### Learning curve

Developers need to understand that interpolated string calls without parentheses behave differently from parenthesized calls. This could be confusing:

```luau
log:Info `Hello {name}`        -- passes 3 arguments (format, values, expressions)
log:Info(`Hello {name}`)       -- passes 1 argument (formatted string)
```

This distinction is intentional and valuable for DSL use cases, but documentation will need to clearly explain the difference.

### Surprising behavior with existing functions

Existing functions that are not designed for this calling convention will receive unexpected arguments if called without parentheses. For example, in Roblox, `print` and `warn` accept variadic arguments and would print the format string, values table, and expressions table alongside each other rather than producing a formatted message.

Functions designed for parentheses-free interpolated string calls would need to be written (or updated) to accept the three-argument format. Developers should continue using parenthesized calls for existing functions, or use a pattern such as the `Message` wrapper type described in the Design section to bridge the gap.

## Alternatives

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

This was considered and the proposed design shares the same spirit: decomposing the interpolated string into parts that don't require the consumer to parse Luau expressions. The proposed design differs in using a format string with `%s` placeholders instead of a string parts array, and in providing an additional expressions table with the source text of each interpolated expression. The format string approach:

1. Provides a single template string usable as a log aggregation key or cache key
2. Is more familiar to Luau developers accustomed to `string.format`-style patterns
3. Includes expression metadata (source text) useful for debugging and structured logging

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
