# Interpolated string function calls

## Summary

Allow calling functions with interpolated string literals without parentheses

The call is desugared into a function call with a table argument.  This table contains an array of string literal segments and the evaluated values from the substitution tokens.

A new core library function is also introduced for performing typical default string interpolation.

This enables domain-specific language patterns like SQL escaping, and HTML templating.

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

### AST rewrites

[lute](https://lute.luau.org/) includes a powerful query system for searching and making procedural edits to Luau code.

For example, a simple rewrite rule that replaces `expr ~= expr` with `math.isnan(expr)` is today written like so:

```luau
local query = require("@std/syntax/query")
local syntax = require("@std/syntax")
local syntax_utils = require("@std/syntax/utils")

local function transform(ctx)
    return query
        .findallfromroot(ctx.parseresult.root, syntax_utils.isExprBinary)
        :filter(function(bin)
            return bin.operator.text == "~="
                and bin.lhsoperand.tag == "local"
                and bin.rhsoperand.tag == "local"
                and bin.lhsoperand.token.text == bin.rhsoperand.token.text
        end)
        :replace(function(bin)
            return `math.isnan({(bin.lhsoperand :: syntax.AstExprLocal).token.text})`
        end)
end

return transform
```

[https://github.com/luau-lang/lute/blob/primary/examples/query\_transformer.luau\#L15](https://github.com/luau-lang/lute/blob/primary/examples/query_transformer.luau#L15)

With interpolated string function calls, the replacement expression can be expressed in a much clearer way:

```
return ast`math.isnan({bin.lhsoperand})`
```

### SQL escaping

Interpolated string calls enable safe, ergonomic SQL query construction where the function can automatically escape interpolated values:

```luau
local sqlite = require("luau_sqlite")

local function takeUserInput(db: sqlite.DB, user: string, comment: string)
    -- Auto-escape inputs to guard against SQL injection
    db:Exec `INSERT INTO user_inputs (user, comment) VALUES ({user}, {comment})`
end
```

### HTML templating

Similarly, HTML renderers can automatically escape interpolated values to prevent XSS attacks:

```luau
local tmpl = require("luau_html_renderer")

local function renderPage(r: tmpl.Renderer, userinput: string)
    -- Automatic XSS protection through escaping
    return tmpl.HTML `The user asked about {userinput}`
end

local bookName = "Pride & Prejudice"
local para = tmpl.HTML `<p>Book name: {bookName}</p>` -- <p>Book name: Pride &amp; Prejudice</p>
```

## Design

### Grammar

The grammar for function calls is extended to allow an interpolated string as the argument:

```
functioncall ::= prefixexp args
args ::= '(' [explist] ')' | tableconstructor | LiteralString | stringinterp
```

### Semantics

When a function is called with an interpolated string literal in this style, the compiler will desugar the expression to a function call of the following form:

```luau
f `Template string {expr1} content {expr2} ...`

-- to

f(
    string.interpolatedstring.create(
        {"Template string ", " content ", " ..."},
        {expr1, expr2},
    )
)
```

The function is passed a table with two keys:

1. `strings` – an array containing all of the string literal sections of the template string, and
2. `expressions` – An array containing the values to be substituted in

To handle the case that any interior `expression` may be `nil`, the desugaring ensures that `#strings` is always one greater than `#expressions`.  If the template string starts or ends with an expression, the first or last elements of `strings` will be the empty string.

```luau
f `{expr1} {expr2}` --> f(string.interpolatedstring.create({"", " ", ""}, {expr1, expr2})
```

The table `string.interpolatedstring` is added to the standard `string` library.  Its contents are roughly

```luau
string.interpolatedstring = table.freeze({
    __tostring=...,
    __concat=...,
    __len=...,
    create=function (strings, expressions)
        return setmetatable({strings=strings, expressions=expressions}, string.interpolatedstring)
    end
})
```

Where each of the metafunctions acts to make the interpolated string value behave somewhat like an ordinary string.

## Core library functions

A new function `string.interpolate` is added to afford easy access to the default interpolation algorithm.

```luau
function string.interpolate(t: {strings: {string}, expressions: {unknown}})
    local n = #t.strings - 1
    local result = ""
    for i = 1, n do
        result ..= t.strings[i] .. tostring(t.expressions[i])
    end
    result ..= t.strings[#strings]
    return result
end
```

This function renders an interpolation template by replacing each `expression` with the `tostring` of the corresponding expression, matching the behavior of normal string interpolation.

```luau
local world = 'World'
local s = string.interpolate `Hello {world}!`
assert("Hello World!" == tostring(s))
```

This will be useful for any application that wants to augment the default string interpolation logic rather than replacing it.

### Examples

```luau
local id = 42
local user = {name = "Alice"}

-- Simple identifier
log:Info `Processing item {id}`

-- Member expression
log:Info `User {user.name} logged in`

-- Multiple expressions
log:Info `{user.name} is processing item {id}`
```

### No expression restrictions

This design uses positional tables, so any expression valid in a regular interpolated string is also valid here, including function and method calls:

```luau
-- Function calls are allowed
log:Info `Result is {compute()}`
-- Desugars to: log:Info(string.interpolatedstring.create({"Result is ", ""}, {compute()})

-- Method calls are allowed
log:Info `Name is {user:getName()}`
-- Desugars to: log:Info(string.interpolatedstring.create({"Name is ", ""}, {user:getName()})

-- Repeated expressions are fine (each is a separate positional entry)
log:Info `{increment()} and then {increment()}`
-- Desugars to: log:Info(string.interpolatedstring.create({"", " and then ", ""}, {increment(), increment()}))
```

Since values are stored positionally rather than keyed by expression text, there is no ambiguity when the same expression appears multiple times or when expressions have side effects.

### Behavior with variadic functions

```luau
local name = "Alice"
print `Hello {name}`
-- Desugars to: print(string.interpolatedstring.create({"Hello ", ""}, {name}))
-- Output: Hello Alice
```

This works because the result of `string.interpolatedstring.create` provides the `__tostring` metamethod.  It is not a string, but can easily be used like a string in simple cases.

### Interaction with existing syntax

All existing valid function call syntaxes are unchanged:

- Regular string literals: `print "hello"`
- Table literals: `print {1, 2, 3}`
- Parenthesized calls: ``print(`hello {name}`)``

The new behavior only applies to calls without parentheses using interpolated strings.

## Drawbacks

### Increased complexity in the grammar

Adding interpolated strings as a parentheses-free call argument adds complexity to the parser. However, the grammar change is minimal and unambiguous.

### Interpolated strings aren't (quite) strings

The value passed to an interpolated string function call behaves a little bit like a string, and can easily be converted into a string, but it isn't actually a string.  It cannot be passed to `string.sub` for instance.

The static type system can catch this for developers who use it, but it may trip up some users from time to time.

### Every call is 3 tables

This isn't the worst thing, but it'll be a tiny bit less efficient than what you'd write if you did things by hand.  The extra enclosing table is worth it because it encapsulates everything into a single value with a useful metatable.

## Alternatives

### Including the source text in the desugaring

An early motivating use case for this feature was to afford nice syntax to a structured logging API.

We debated this for quite some time, but came to the conclusion that, even if we did offer this, it wouldn't enable quite the kind of structured logging API that we were looking for.

Since there were no other motivating use cases that called for extracting the original substitution tokens, this feature has been dropped from the proposal.

### Three-argument desugaring

A previous iteration we considered was for the call to desugar to a function call accepting two or three arguments: A list of string literals, a list of expressions to be intercalated between, and a list of the stringifications of the expressions.

We chose not to move forward with this design because it interacts very badly with functions that are intended to work with actual strings.  For example, under this design, the statement  `` print `my template string` `` would be completely valid, but would actually print out the memory addresses of two or three tables!

This was deemed unacceptably confusing.

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

This approach avoids the double-parsing drawback since all expression metadata is provided at compile time. However it requires every consumer to manually perform byte arithmetic to extract expression names.

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

This approach avoids the learning-curve issue of parenthesized vs. non-parenthesized calls having different semantics for interpolated strings. It also makes the intent explicit at the call site. However, it introduces new syntax (`@` prefix) for a feature that is also achievable through the parentheses-free calling convention and introduces a new type of value which would need a clearly defined interface.

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

### Tagged interpolated strings (JavaScript-style)

JavaScript's tagged template literals pass an array of string parts and the interpolated values as separate arguments:

```javascript
tag`Hello ${name}, you have ${count} messages`;
// Calls: tag(["Hello ", ", you have ", " messages"], name, count)
```

We considered a Luau adaptation:

```luau
tag `Hello {name}, you have {count} messages`
-- Would call: tag({"Hello ", ", you have ", " messages"}, name, count)
```

This was considered and the proposed design shares the same spirit: decomposing the interpolated string into parts that don't require the consumer to parse Luau expressions. The proposed design allows recovering the original template string (useful as an aggregation key) and providing library functions for flexible extraction of expression metadata.

The current proposal is very much a spiritual extension of this, but with the improvement that the interpolated string is passed as just one value.  This value can be used by any function that naively stringifies its argument (like `print`).  It can also be augmented with more keys later.

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

- [String interpolation RFC](https://github.com/luau-lang/rfcs/blob/master/docs/syntax-string-interpolation.md) \- The original RFC that introduced interpolated strings and noted the temporary restriction
