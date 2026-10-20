# Decimal Type

## Summary

A builtin `decimal` value type for exact base-10 arithmetic, written `19.99d`. It has the semantics of .NET `System.Decimal`, with a 90-bit mantissa and a scale of 0 to 28, and it is stored inline in the value slot, like `integer` and `vector`.

## Motivation

Luau is a good fit for business rules embedded in an application, such as pricing, fees, tax, invoices and settlement. These rules change more often than the host application, and they benefit from running in a sandbox. They have to give the result a person gets by hand, and `number` can't do that:

```luau
print(0.1 + 0.2 == 0.3)       -- false
print(1.15 * 100)             -- 114.99999999999999
print(string.format("%.2f", 2.675), math.floor(2.675 * 100 + 0.5) / 100) -- 2.67  2.68
```

Integer cents fix addition but not percentages, unit prices or three-decimal currencies. Scripts end up tracking the scale by hand.

An embedder can already build a decimal as a table or as userdata. I built both, and measured them next to `mlua-decimal`, an existing Rust userdata library. They work, but they share the problems the integer RFC lists for 64-bit integers:

- Each runtime defines its own semantics. Rounding, scale and mixing rules differ between hosts, so a script that computes an invoice can give different results in different hosts.
- The type checker sees `userdata` or `table`. Hosts can ship definitions, but even then `price * 0.1` is not caught.
- There are no literals or constant folding. Every constant is `Decimal("19.99")`, parsed at run time.
- Each result is a heap allocation. A same-scale addition takes 4.2 ns with the inline type, 70 ns with a C++ userdata and 184 ns with `mlua-decimal`. A stored value takes 16 bytes inline, 48 bytes as a C++ userdata and 85 bytes with `mlua-decimal`.

A decimal fits in the existing 16-byte `TValue`, so making it a core value type costs no more memory per value than `integer`.

The measurements, the .NET comparison and the reasoning are written up at https://vivainio.github.io/articles/luau-decimal/.

### About the prototype

There is a full prototype of everything in this RFC, so the proposal can be judged against working code: https://github.com/vivainio/luau/pull/1. Claude Code wrote all of the implementation. I set the semantics and checked the behaviour, and the existing `integer` implementation served as the model for every layer. The [walkthrough](https://vivainio.github.io/articles/luau-decimal/implementation-walkthrough/) lists where the model held and the ten places it didn't. Because of how it was written, I'm not offering the code as reviewed code. The evidence for correctness is the testing: fuzzers against Python's `decimal` module and integers, and a comparison harness against .NET. The prototype is not meant to be merged. It shows what a complete version would touch and cost, with tests and runnable demos. If the team accepts some form of this, I expect the team to build it their own way.

### Scope

The design below is the complete version. Most of it can be cut without affecting the rest, and I'd rather see a smaller version accepted than this one rejected. Roughly from most to least separable:

- implicit mixing with `integer`
- arithmetic operators (the library covers every one of them)
- the `d` literal (`decimal.fromstring` covers it, but without constant folding)
- the 90-bit mantissa (a 64-bit one keeps 18 digits and has simpler arithmetic)

What I consider the core of the proposal is one shared definition of decimal arithmetic, a `decimal` primitive in the type checker, and a value that is stored in the slot without allocating.

## Design

### Value

A decimal is a sign, an unsigned mantissa below 2^90, and a scale (the number of digits after the point) from 0 to 28. That is 96 bits, the same as the `TValue` payload. There is no NaN, no infinity and no negative zero.

Every decimal with up to 27 significant digits is representable, and up to 28 digits when the value is below about 1.24e27. With 2 decimal places, the largest value is 12,379,400,392,853,802,748,991,242.23.

The scale is preserved, and comparison ignores it. This is the .NET behaviour, and it lets `tostring(2.50d)` print `2.50`:

```luau
print(2.50d, 2.5d)            -- 2.50  2.5
print(2.50d == 2.5d)          -- true (also for rawequal)
print(1.50d + 1.5d)           -- 3.00
print(1.5d * 2d)              -- 3.0
```

### Literals

A numeric literal followed by `d` is a decimal literal. Separators and exponents are allowed: `1.25d`, `1_000.50d`, `1e3d`. The literal must be exactly representable, otherwise it is a parse error. It is never rounded. Hexadecimal and binary literals can't take the suffix, so `0x1d` keeps its current meaning. As with integers, a negative literal is unary `-` applied to the literal.

### Operators

`+ - * / %`, unary `-`, `== ~= < <= > >=` are defined between decimals. `//` and `^` are not defined and raise an error.

- Add and subtract: the result scale is the larger operand scale.
- Multiply: the result scale is the sum of the operand scales, capped at 28.
- Divide: the quotient has as many digits as fit, and trailing zeros are then removed down to `max(scale(a) - scale(b), 0)`. So `6d / 2d` is `3`, `1.00d / 2d` is `0.50`, and `1d / 3d` is `0.333333333333333333333333333`.
- Remainder: `%` has the sign of the dividend, so `-7d % 3d` is `-1`.

Each result is computed exactly and then rounded half-even once. Only the digits after the point are ever rounded. If the integer part does not fit, the operation raises `decimal overflow`. Division by zero raises an error.

An `integer` mixes implicitly in arithmetic and ordering (`price * 3i`, `x < 10i`) and the result is a decimal. A `number` never mixes: `1.5d + 1` and `1.5d < 2` raise an error and are reported by the type checker. `==` is type-strict, as it is for `integer` and `number`, so `1d == 1i` is `false`.

### Library

All functions are in a new `decimal` library. Arguments typed `decimal | integer` accept either, and the result is always a decimal.

- `decimal.create(x: number | integer | string | decimal): decimal?` converts a value to a decimal. A number converts to its shortest round-trip digits, so `decimal.create(0.1)` is `0.1d`. Excess digits are rounded half-even. The function returns `nil` when the value does not fit, is NaN or infinite, or the string is malformed.
- `decimal.fromstring(s: string): decimal?` accepts an optional sign, digits with an optional point and exponent, and surrounding whitespace. It returns `nil` on failure, like `integer.fromstring`.
- `decimal.tostring(d): string` and `decimal.tonumber(d): number` convert out. `tonumber` can lose precision.
- `decimal.add`, `sub`, `mul`, `div`, `rem`, `neg`, `abs` are the operators as functions.
- `decimal.round(d, places: number, mode: string?): decimal` rounds to `places` digits, from 0 to 28. The modes are `"halfeven"` (the default), `"awayfromzero"`, `"tozero"`, `"tonegativeinfinity"` and `"topositiveinfinity"`, with the same names as .NET `MidpointRounding`.
- `decimal.floor`, `ceil`, `trunc` round to an integral value.
- `decimal.sign(d): number`, `decimal.scale(d): number`, `decimal.min(...)`, `decimal.max(...)`, `decimal.lt(a, b)` and `decimal.le(a, b)` cover inspection and comparison.

### Base library

- `type` and `typeof` return `"decimal"`.
- `tostring` and `print` use the stored scale and never use exponent notation, and they print no `d` suffix. A decimal is not coerced to a string by `..` or `string.format("%s")`, the same as an `integer`.
- `rawequal` compares numerically, and `table.find` finds a decimal that compares equal.
- Decimals can't be used as table keys in this proposal.

### Type system

`decimal` is a primitive type, like `integer`. Refinement with `type(x) == "decimal"` works, and `types.decimal` is available in type functions. The operators are typed, so `d * 0.1` is a type error and `d * 2i` is a `decimal`. The prototype supports both the new and the old solver.

### C API

```c
int lua_todecimal(lua_State* L, int idx, int* negative, uint64_t* hi, uint64_t* lo, int* exponent);
void lua_pushdecimal(lua_State* L, int negative, uint64_t hi, uint64_t lo, int exponent);
#define lua_isdecimal(L, n) (lua_type(L, (n)) == LUA_TDECIMAL)
int luaopen_decimal(lua_State* L); // included in luaL_openlibs
```

The mantissa is passed as two 64-bit halves so that the API doesn't depend on the 90-bit width. I'm open to a string-based or a different shape here.

## Drawbacks

- It's a new value type with operators. The integer RFC deliberately did not define arithmetic operators, and this proposal does. Alternatives explains why.
- Implicit mixing with integers is the only implicit conversion between two numeric types in Luau.
- It's a large change. In the prototype, the complete version touches about 57 files, most of them with small changes in the same places `integer` touched, plus a new library (~850 lines) and a portable 256-bit arithmetic helper (~300 lines). A smaller version would touch less.
- Native code generation would need inline lowering to get the full benefit. The prototype doesn't have it: compiled code calls the same helper as the interpreter.
- Table keys are not supported, because keys must hash consistently with numeric equality (`2.5d` and `2.50d`). Normalizing before hashing would solve this, but the prototype doesn't do it.
- The range is fixed. At 27 to 28 digits, a value is one digit and a 64x range short of .NET. A wider type would need a different, heap-allocated representation.
- The suffix `d` is no longer available for other literal kinds.
- Serialization and inspection libraries that handle every Luau value need updating, as they did for `integer`.

## Alternatives

**Do nothing.** Embedders keep writing their own decimals, with differing semantics, no static typing, and additions that take 70 to 184 ns instead of 4.2 ns.

**Library functions only, as in the integer RFC.** The integer RFC gives four reasons for leaving out operators. For decimals they look like this:

- Integers rely on bitwise operations, which need library calls anyway. Decimals have none.
- Integer division has competing definitions (truncated or floored). Decimal division has a well-defined result: the exact quotient, rounded half-even to the available precision. Only the result scale is a choice, and this proposal uses the .NET rule.
- Users who wanted speed would still call library functions. With decimals, the operators are the fast path: same-scale `+`, `-`, `*` and comparisons run inline in the interpreter.
- Polymorphic operators can deoptimize untyped code. This is the same tradeoff `vector` already makes.

Money code is mostly arithmetic expressions, such as `subtotal * (1d + vatRate) - discount`. Written as `decimal.sub(decimal.mul(subtotal, decimal.add(1d, vatRate)), discount)`, it is hard to check against the formula in a spec. Still, a library-only version would be useful (see Scope).

**No integer mixing.** Requiring `decimal.create(qty)` everywhere is consistent but noisy, and the conversion is always exact. It can be left out.

**A 64-bit mantissa.** The first prototype used a 64-bit mantissa, with only 18 digits and simpler arithmetic. 90 bits is the most that fits in the slot, and the extra cost is small: same-scale addition went from 2.7 ns to 4.2 ns.

**A heap-allocated (GC object) decimal.** This would allow full .NET or IEEE 754-2008 decimal128 precision, but every result would allocate and the fast paths would be lost.

**Host extension points instead of a type.** Embedders could register a literal suffix and an operator type table for a userdata. This is smaller and fixes literals and typing, but it doesn't give one definition or remove the allocation.

**IEEE 754 decimal64 or decimal128.** These formats have NaN, infinities, signed zero and cohorts, which the money use case doesn't need, and decimal128 doesn't fit in the slot.

## Prior Art

- .NET `System.Decimal` is the semantic model. Of 75 cases run through both implementations, 64 give identical output. The 11 differences are explained at https://vivainio.github.io/articles/luau-decimal/checked-against-dotnet/: the 28th digit, the range, one error message and double conversion.
- Java `BigDecimal` and Python `decimal` are arbitrary precision. Python's module also preserves scale and uses half-even rounding by default.
- SQL `DECIMAL(p, s)` fixes precision and scale per column, and it is a common place these values come from.
- Within Luau, the design follows `integer`: a suffix literal, a primitive type, a library named after the type, and no implicit conversion to or from `number`.
