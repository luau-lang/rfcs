# Interpolated string function calls

## Summary

Allow calling functions with interpolated string literals without parentheses, optionally followed by a context table literal argument. This enables domain-specific language patterns like structured logging where the template, interpolation values, and optional additional context are all passed to the function.

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

This proposal lifts that restriction and extends it further to support an optional trailing table literal, enabling powerful DSL patterns. The primary motivating use case is structured logging, where it's valuable to capture both the formatted message and the template with its interpolation values for machine processing:

```luau
local name = "Alice"
local log = require("@rbx/logging")

-- Calls log:Info with three arguments:
--   "Hello Alice" (formatted string)
--   "Hello {name}" (template)
--   {name = "Alice"} (interpolation values)
log:Info `Hello {name}`

-- Calls log:Info with four arguments, adding structured context:
--   "Hello Alice" (formatted string)
--   "Hello {name}" (template)
--   {name = "Alice"} (interpolation values)
--   {userId = 12345, region = "us-east"} (additional context)
log:Info `Hello {name}`, {userId = 12345, region = "us-east"}
```

This pattern is common in structured logging, where preserving the template enables aggregating logs by message pattern while the interpolation values and context provide searchable structured data for platforms like Splunk, Datadog, or Elasticsearch.

Without this feature, there are two alternatives:

1. Manually construct all arguments:

   ```luau
   log:Info(`Hello {name}`, "Hello {name}", {name = name}, {userId = 12345, region = "us-east"})
   ```

   This is tedious, duplicates the `name` variable twice (and the template string once), and is easy to get wrong when refactoring.

2. Use a logging library that performs its own template parsing at runtime:

   ```luau
   log:Info("Hello {name}", {name = name, userId = 12345, region = "us-east"})
   ```

   This doesn't require new syntax, but requires the logging library to implement its own string interpolation separate from the language's built-in interpolation. It also cannot benefit from compile-time optimizations or static analysis that language-level support would enable.

## Design

### Grammar

The grammar for function calls is extended to allow an interpolated string as the argument, optionally followed by a comma and a table literal:

```
functioncall ::= prefixexp args
args ::= '(' [explist] ')' | tableconstructor | LiteralString | stringinterp [',' tableconstructor]
```

Note that the optional `, tableconstructor` is only valid following `stringinterp`, not following a regular `LiteralString` or standalone `tableconstructor`.

### Semantics

When a function is called with an interpolated string literal in this style, the call receives multiple arguments derived from the interpolated string:

1. **Formatted string**: The fully interpolated result (what you would get from the expression today)
2. **Template string**: The original template with placeholders intact, e.g. `"Hello {name}"`
3. **Interpolation table**: A table mapping placeholder names to their values at call time, e.g. `{name = "Alice"}`
4. **Context table** (optional): If a table literal follows the interpolated string, it is passed as a fourth argument

For method calls (`:` syntax), `self` is passed first as usual, followed by these arguments.

### Examples

```luau
local id = 42
local user = {name = "Alice"}

-- Basic usage: passes three arguments
log:Info `Processing item {id}`
-- Desugars to: log:Info("Processing item 42", "Processing item {id}", {id = 42})

-- With context table: passes four arguments
log:Info `User {user.name} logged in`, {timestamp = os.time()}
-- Desugars to: log:Info("User Alice logged in", "User {user.name} logged in", {user = {name = "Alice"}}, {timestamp = 1234567890})
```

### Behavior with variadic functions

Functions that accept variadic arguments will receive all the desugared arguments. For example, Roblox's `print` concatenates its arguments with spaces:

```luau
local name = "Alice"
print `Hello {name}`
-- Desugars to: print("Hello Alice", "Hello {name}", {name = "Alice"})
-- Output: Hello Alice Hello {name} table: 0x...
```

This is likely not the desired output. Developers wanting simple string interpolation should use parentheses:

```luau
print(`Hello {name}`)  -- Output: Hello Alice
```

The parentheses-free form is intended for functions specifically designed to receive the expanded arguments, such as structured logging APIs.

### Handling of duplicate placeholder names

If the same identifier appears multiple times in a template, it appears once in the interpolation table:

```luau
log:Info `{x} + {x} = {result}`
-- Interpolation table is {x = 10, result = 20}, not {x = 10, x = 10, result = 20}
```

### Interpolation table keys for member expressions

When a placeholder contains a member expression like `{user.name}`, the interpolation table uses nested tables mirroring the access path:

```luau
log:Info `{user.name} is {user.age} years old`
-- Interpolation table is {user = {name = "Alice", age = 30}}
```

This approach is more ergonomic for consuming code, which can use natural table access like `context.user.name`, and is more idiomatic to Lua/Luau's table-centric design.

When multiple member expressions share a common root, their values are merged into a single nested structure. If a simple identifier `{user}` and a member expression `{user.name}` both appear, the simple identifier provides the full object, which already contains the nested values.

An alternative design would use flat string keys:

```luau
-- Alternative (not proposed): {["user.name"] = "Alice", ["user.age"] = 30}
```

This would be simpler to implement (no merging logic required) but less ergonomic, requiring consuming code to use string keys like `context["user.name"]`.

### Interaction with existing syntax

This feature does not change the behavior of:

- Regular string literals: `print "hello"` continues to pass a single string argument
- Table literals: `print {1, 2, 3}` continues to pass a single table argument
- Parenthesized calls: `print(`hello {name}`)` continues to pass a single formatted string

The new behavior only applies to calls without parentheses using interpolated strings.

### Expression restrictions in templates

When used in this calling style, the interpolated expressions within the template may not contain function or method calls:

```luau
-- Valid: simple identifiers
log:Info `Hello {name}`

-- Valid: member expressions
log:Info `User {user.name} logged in`

-- Valid: arithmetic and other operators
log:Info `Sum is {a + b}`
-- Desugars to: log:Info("Sum is 30", "Sum is {a + b}", {a = 10, b = 20})

-- Valid: other expressions
log:Info `Count is {#items}`     -- {items = {...}}
log:Info `Active: {not disabled}` -- {disabled = false}

-- Invalid: function calls
log:Info `Result is {compute()}`  -- parse error

-- Invalid: method calls
log:Info `Name is {user:getName()}`  -- parse error
```

Function and method calls are restricted because the same call expression can appear multiple times and return different values:

```luau
log:Info `{increment()} and then {increment()}`
-- Formatted: "1 and then 2"
-- Template: "{increment()} and then {increment()}"
-- Context: ??? (can't use "increment()" as key twice with different values)
```

For all other expressions, the compiler extracts the referenced identifiers and includes them in the interpolation table. Identifiers have stable values within a single expression, so duplicates are not a problem.

For function calls, use a local variable or the traditional parenthesized call syntax:

```luau
local result = compute()
log:Info `Result is {result}`  -- Valid: use a local variable

-- Or use parentheses for full flexibility
log:Info("Result is " .. tostring(compute()))
```

#### Potential future extension

A future version could allow function calls by using indexed keys to distinguish multiple calls to the same expression:

```luau
log:Info `{increment()} and then {increment()}`
-- Could produce: {["increment()#1"] = 1, ["increment()#2"] = 2}
```

This would also capture any identifiers referenced in the call arguments:

```luau
log:Info `{compute(x)} and {compute(x)}`
-- Could produce: {x = 10, ["compute(x)#1"] = 42, ["compute(x)#2"] = 43}
```

However, this approach has a complication: function calls can mutate values, affecting subsequent expressions in the template:

```luau
log:Info `{x} {mutate(x)} {x}`
-- First {x} sees original value
-- mutate(x) changes x
-- Third {x} sees mutated value
-- But context only captures x once—which value?
```

This affects all identifiers, not just function call results. To handle this correctly, the indexed approach would need to apply to every expression in the template, not just function calls:

```luau
log:Info `{x} {mutate(x)} {x}`
-- Could produce: {
--   ["x#1"] = {value = 1},
--   ["mutate(x)#2"] = {result = nil, args = {x = {value = 1}}},
--   ["x#3"] = {value = 2}
-- }
```

This adds significant complexity. An alternative would be to document that the context captures values at an unspecified point during evaluation, and that templates with side effects may produce inconsistent results. This is left for future consideration.

## Drawbacks

### Increased complexity in the grammar

Adding optional trailing arguments after interpolated strings adds complexity to the parser. However, the grammar remains unambiguous since the comma can only appear in this context.

### Use-case-specific syntax

The optional trailing context table argument is motivated primarily by structured logging, where additional metadata beyond the interpolated values is useful. In a pure language context, this extra argument may seem overly specific to one use case. However, the context table is optional, and the core feature (passing template and interpolation values to functions) is broadly useful for DSL patterns beyond logging.

### Learning curve

Developers need to understand that interpolated string calls without parentheses behave differently from parenthesized calls. This could be confusing:

```luau
log:Info `Hello {name}`        -- passes 3 arguments
log:Info(`Hello {name}`)       -- passes 1 argument
```

This distinction is intentional and valuable for DSL use cases, but documentation will need to clearly explain the difference.

### Roblox built-in logging functions

In the Roblox engine, `print`, `warn`, and `error` have non-standard behavior: calls to them are treated as logging calls and feed into log telemetry. These functions handle arguments differently:

- `print`: Accepts any number of arguments and prints their values (without calling `tostring`, though `__tostring` metamethods fire for tables)
- `warn`: Accepts any number of arguments, converts them to strings, and joins them with spaces, outputting as a yellow warning with timestamp
- `error`: Expects a single message argument and terminates execution

For `print` and `warn`, parentheses-free interpolated string calls would produce confusing output since the template string and interpolation table would be printed alongside the formatted message. For `error`, the first argument is still the formatted message, so it would function correctly; the extra arguments would simply be ignored.

Roblox could update these functions to leverage the extra arguments for structured log telemetry. Alternatively, a new structured logging library could be designed from the start to accept parentheses-free interpolated string calls, while developers continue using parenthesized calls for the legacy functions.

### Restriction on function calls

The restriction on function and method calls may frustrate developers who want to inline computed values. It also introduces inconsistency: regular string interpolation allows `{compute()}`, but the parentheses-free calling form does not. However, this restriction avoids the complexity of handling duplicate call expressions that return different values (e.g., `{increment()} and {increment()}` returning 1 and 2). A potential future extension using indexed keys is described in the Design section.

## Alternatives

### Tagged interpolated strings (JavaScript-style)

JavaScript's tagged template literals pass an array of string parts and the interpolated values as separate arguments:

```javascript
tag`Hello ${name}, you have ${count} messages`;
// Calls: tag(["Hello ", ", you have ", " messages"], name, count)
```

This was considered but rejected because:

1. It doesn't provide the template string directly, requiring reconstruction
2. It doesn't provide a table mapping names to values
3. The array-based API is less natural for Luau's table-centric design

## References

- [String interpolation RFC](https://github.com/luau-lang/rfcs/blob/master/docs/syntax-string-interpolation.md) - The original RFC that introduced interpolated strings and noted the temporary restriction
- [API-1080](https://roblox.atlassian.net/browse/API-1080) - Structured logging APIs proposal that motivated this RFC
