# Lambda Function Syntax

## Summary

Provide a parsable, typeable, flexible, readable and, most importantly of all, short syntax for writing one-line functions.

## Motivation

Functions are useful.
- They define an arbitrary behavior.
- They can be defined and used anywhere.

Even though functions can be defined anywhere, it is hard to define them everywhere because anonymous function syntax is generally verbose.

```lua
-- Most of this function definition is made up of relatively long syntax. In
-- comparison, the function's content is much smaller.
--                                     VVVV         VVVVV
local sum = foldLeft(numbers, function(a, b) return a + b end, 0)
```

Lambda functions are a way to shorten anonymous functions in situations where they can be written on one line.

## Design

Lambda functions are function definitions made up of zero or more optionally typed parameters in parentheses `()` and one or more return expressions, all wrapped in backslashes `\\`. Return types can be specified after the backslash delimiters using a colon `:`. Generic types can be specified as well, in angle brackets `<>` before defining parameters.

Here are some examples.

```lua
local lessThan = \(a, b) a < b \
local sortedAsc = \(a, b) math.min(a, b), math.max(a, b) \
local getOne = \() 1 \

local sum = foldLeft(numbers, \(a, b) a + b \, 0)

-- higher-order function: currySub(20)(30) == -10
local currySub = \(a) \(b) a - b \\

-- lessThan typed
local lessThan = \(a: number, b: number) a < b \: boolean

-- generic function
local first = \<T>(ls: {T}) ls[1] \: T
```

### Syntax

This is the grammar, borrowing non-terminals from [Luau's grammar](https://luau-lang.org/grammar). It would be appended to the `simpleexp` non-terminal, denoted as `simpleexp'`.

```
lambdafunc = '\' ['<' GenericTypeList '>'] '(' [parlist] ')' explist '\' [':' ReturnType]

simpleexp' = simpleexp | lambdafunc
```

The first backslash is only allowed where an expression is expected, and the last backslash is only allowed where an operator is expected. So in most cases, the backslash delimiters should be unambiguous.

The grammar imposes the following restrictions to prevent the remaining cases where they *are* ambiguous. There are examples included as rationale.

1. Lambda functions must return at least one expression.

```lua
-- returning 0 expressions is illegal syntax
local discard = \(...) \

-- Alternative: use regular function syntax instead
local discard = function(...) end

-- Note: it's still legal to call a function that returns 0 values
local debug = \(msg) print(msg) \
```

```lua
-- arbitrary ambiguous case that breaks LL(n)
-- using asymmetric delimiters would fix this issue

-- local cmp, B, C, rest = function() end < A, B, C, ...
local cmp, B, C, rest = \() \ < A, B, C, ...

-- local getGetOne = function() return function<A, B, C, ...>() return 1 end end
local getGetOne = \() \<A, B, C, ...>() 1 \\
```

2. Lambda function delimiters must not be used as implicit delimiters to a function call.

```lua
-- using lambdas as implicit call delimiters is illegal syntax
local ok, ten = pcall \() 10 \

-- Alternative: wrap the lambda in parentheses
local ok, ten = pcall(\() 10 \)
```

```lua
-- ambiguous case that breaks LL(n)
-- using asymmetric delimiters would fix this issue

--[[
local v1, v2, v3, ... = ArgsSorter(function(a, b)
  return a < b
end)(x, y, z, ...)
]]
local v1, v2, v3, ... = ArgsSorter \(a, b) a < b \ (x, y, z, ...)

--[[
local sorter = ArgsSorter(function(a, b)
  return a < b(function(x, y, z, ...) return 2 end)
end)
]]
local sorter = ArgsSorter \(a, b) a < b \(x, y, z, ...) 2 \\
```

```lua
-- another ambiguous case that breaks all parsers
-- moving the return type inside the delimiters would fix this issue

-- annotating the lambda function with a return type of typeof(...)
-- local generate64 = generatorOf(function(): typeof(...) return 64 end)
local generate64 = generatorOf \() 64 \: typeof(...)

-- calling the `typeof` method on the return value of `generatorOf(...)`
-- local generatorType = generatorOf(function() return 64 end):typeof(...)
local generatorType = generatorOf \() 64 \ :typeof(...)
```

Restriction 1 may be acceptable because lambda functions typically do not create side-effects and therefore must return something, but there is no convenient reason for restriction 2 beyond sound parsing.

### Semantics

Lambda functions behave just like anonymous functions with the only difference being that its function body is exactly one `return` statement. For example, all of these statements yield equivalent runtime and type semantics.

```lua
-- function sugar syntax
local function add(a: number, b: number): number return a + b end

-- anonymous function syntax
local add = function(a: number, b: number): number return a + b end

-- lambda function syntax
local add = \(a: number, b: number) a + b \: number

-- In all cases, they can be used like this
print(add(10, 20)) --> 30
```

## Drawbacks

This adds a special symbol `\` to the language and increases the language's complexity.
- It is not the most accessible on all keyboards, e.g. QWERTZ and AZERTY, so it may be annoying for non-US users to write.
- It may conflict with a potential set difference operator in type space, which is usually denoted as `T \ U` in mathematics. e.g. `\() nil :: T \ U \`
- This syntax might appear alien to users. Haskell may be one of the only languages that use `\` to refer to a function.

## Alternatives

This syntax does not have to be implemented at all. It encourages users to write less error-prone code at best via functional programming, and it is another thing for a newcomer to learn how to read at worst. Some users may prefer writing their code in an inherently less performant style.

There were a lot of other syntaxes considered before this, some examples and explanations why they were not chosen are listed below:

```lua
-- moving the return type inside the delimiters
-- less human-parsable but more machine-parsable
-- helps with removing restriction 2
local add = \(a: number, b: number): number a + b \

-- more human-parsable like this...?
local add = \(a: number, b: number): (number) a + b \

-- more human-parsable like this, but defeats the purpose
local add = \(a: number, b: number): number
  a + b
\
```

```lua
-- syntaxes with different delimiters
-- some use asymmetric delimiters, which help with removing both restrictions
local add = |(a, b) a + b | -- ambiguous when casting the return expression
local add = #[(a, b) a + b ] -- unrelated symbols
local add = \ a, b -> a + b \ -- makes parameters harder for a human to parse
local add = .\(a, b) a + b \ -- '.' seems forgettable...
local add = |(a, b) a + b \ -- mismatching delimiters don't look right...
local add = \$(a, b) a + b \ -- unrelated '$' symbol
local add = |a, b|[ a + b ] -- unrelated symbols; where would generics go?
```

```lua
-- parameter-less syntax, kind of like a format string
-- It would be modeled semantically like this:
-- function(...) return select(1, ...) + select(2, ...) end
local add = \%1 + %2\
local add = \%\a, b\ a + b \ -- a co-existing named variant
local add = \%(1: number) + %(2: number)\: number -- type annotations look ugly

local add = |$1 + $2| -- ambiguous when casting, e.g. |$1 :: number | string|
local add = |$|a, b| a + b | -- similar named variant

local add = &(&1 + &2) -- Elixir's shorthand

-- no parameter names mean less readability when symbols matter more
-- compare this to: \(a, b, t) a + (b - a) * t \
local lerp = \%1 + (%2 - %1) * %3\

-- higher-order functions get confusing
-- compare this to:
-- function(a) return function(b) return a - b end end
-- \(a) \(b) a - b \\
local currySub = \\%1 - %%1\\
local currySub = \%\a\ \%\b\ a - b \\
```

```lua
-- delimiter-less syntax
-- this was too different from anonymous functions in the original RFC

local add = do(a, b) a + b
local add = do(a: number, b: number): (number) a + b
local add = \(a, b) a + b
local add = \a, b -> a + b -- Haskell's lambda

-- currying is a little more carefree
local currySub = do(a) do(b) a - b

-- multi-returns require a no-op call or syntactic equivalent
local ret = do(...) ...
local sortedAsc = do(a, b) ret(math.min(a, b), math.max(a, b))
```

The two restrictions from the Syntax section could be lifted by careful symbol choices and ordering, but the RFC's syntax seems to be the easiest to read and write while still being syntactically sound, even with the two aforementioned restrictions.

There are open-source libraries that attempt to fill this "short function" gap, like [Penlight's](https://github.com/lunarmodules/Penlight) placeholder expression library and lambda compiler.

```lua
-- placeholder expressions from pl.func
local utils = require("pl.utils")
util.import("pl.func")

-- pl.func
local lessThan = I(Lt(_1, _2))
local sortedAsc = I(Args(_(math.min)(_1, _2), _(math.max)(_1, _2)))
local getOne = I(Var(1))

foldLeft(numbers, I(_1 + _2), 0)

-- not very readable, but reproducible
local currySub = I(_(bind1)(I(_1 - _2), _1))
```

```lua
-- pl.utils.string_lambda
local L = require("pl.utils").string_lambda

local lessThan = L'|a, b| a < b'
local sortedAsc = L'|a, b| math.min(a, b), math.max(a, b)'
local getOne = L'|| 1'

foldLeft(numbers, L'|a, b| a + b', 0)

-- currySub is not reproducible
```
