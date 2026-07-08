# Default parameters and named call arguments

## Summary

This RFC proposes default parameter values and named call arguments for Luau functions. Default parameters allow function authors to declare fallback values directly in the parameter list, while named call arguments allow callers to provide specific parameters without manually passing placeholder values for earlier parameters.

## Motivation

Luau code commonly uses manual `nil` checks at the start of functions to provide fallback values:

```luau
local function createItem(name: string?, amount: number?, visible: boolean?)
    name = if name == nil then "Item" else name
    amount = if amount == nil then 1 else amount
    visible = if visible == nil then true else visible

    -- ...
end
```

This pattern is repetitive, makes the real function body start later, and becomes harder to read as the number of optional parameters grows. It also makes calls awkward when a caller only wants to override a later argument:

```luau
createItem(nil, nil, false)
```

Default parameters make the intended fallback behavior visible at the function boundary, and named call arguments allow the caller to set only the values that matter:

```luau
local function createItem(name: string = "Item", amount: number = 1, visible: boolean = true)
    -- ...
end

createItem(visible = false)
```

This improves readability, reduces boilerplate, and brings Luau closer to features already present in many mainstream languages. It also avoids the allocation and field access overhead of options table patterns for cases that only need a small number of optional parameters.

## Design

A function parameter may specify a default value using `=` after its optional type annotation:

```luau
local function connect(host: string = "localhost", port: number = 8080, secure: boolean = false)
    return host, port, secure
end
```

If a defaulted parameter receives no value, or receives `nil`, the default expression is evaluated and assigned to the parameter before the function body runs:

```luau
local host, port, secure = connect()
assert(host == "localhost")
assert(port == 8080)
assert(secure == false)

host, port, secure = connect("example.com", nil, true)
assert(host == "example.com")
assert(port == 8080)
assert(secure == true)
```

Default expressions are evaluated inside the function scope after parameters are bound. Parameters are processed from left to right, so later defaults can observe earlier parameters after their defaults have been applied:

```luau
local function circle(radius: number = 10, diameter: number = radius * 2)
    return radius, diameter
end
```

A default expression may refer to earlier parameters and outer bindings, but not to the parameter being initialized or to later parameters. This keeps default evaluation deterministic and avoids depending on a parameter whose own fallback has not been applied yet:

```luau
local function invalid(a: number = a) end -- invalid
local function alsoInvalid(a: number = b, b: number = 1) end -- invalid
```

Named call arguments use `name = expression` syntax inside a function call:

```luau
connect(port = 443, secure = true)
connect("example.com", secure = true)
```

Named arguments are matched against the target function's parameter names. They are accepted only when the callee resolves to a statically known function signature with parameter names. Calls through dynamic function values, `any`, overloaded values where a unique signature cannot be selected, or function types that do not expose the referenced names are rejected instead of using runtime keyword lookup.

For function types and declarations, parameter names remain metadata for named calls, autocomplete, and diagnostics; they do not change function type identity or subtyping by themselves. However, a parameter name in a visible API can still become caller visible once named calls are used, so renaming it can break named call sites.

Positional arguments may appear before named arguments, but positional arguments cannot appear after named arguments:

```luau
connect("example.com", secure = true) -- valid
connect(secure = true, "example.com") -- invalid
```

Duplicate arguments for the same parameter are rejected, whether the duplication is named or positional:

```luau
connect(port = 80, port = 443) -- invalid
connect("example.com", host = "other.com") -- invalid
```

Unknown named arguments are rejected:

```luau
connect(timeout = 10) -- invalid unless the function has a parameter named timeout
```

A named argument call is lowered to the equivalent positional call at compile time. Missing earlier arguments are filled as `nil`, allowing default parameters to handle them naturally:

```luau
connect(secure = true)
-- equivalent to:
connect(nil, nil, true)
```

This means named arguments do not require a runtime table, runtime keyword argument object, or reflective parameter lookup. For statically known functions, the compiler can map names to parameter slots directly.

For method calls using `:`, the implicit `self` argument is not available as a named argument. Names are matched only against the explicit parameters after `self`:

```luau
local object = {}

function object:connect(host: string = "localhost", port: number = 8080)
    return host, port
end

object:connect(port = 443) -- valid
object:connect(self = object) -- invalid
```

The type checker should verify that each default expression is compatible with the parameter type. At the call boundary, a defaulted parameter accepts omission or `nil`; inside the function body, the parameter has the declared type after the default has been applied.

```luau
local function retry(count: number = 3)
    -- count is number here
end

retry()
retry(nil)
retry(5)
```

Parameters without defaults keep existing Luau behavior. If a named argument call skips a non defaulted earlier parameter, that slot still receives `nil`; the type checker rejects the call when `nil` is incompatible with the parameter type.

For functions without default parameters or named call arguments, existing parsing, type checking, bytecode generation, and runtime behavior remain unchanged.

### Prototype and performance

A prototype implementation is available for review and testing at https://github.com/Aurora-Aeralis/luau/tree/aurora/default-params-named-args. The prototype implements parser, AST, type checking, compiler, pretty printer, and conformance coverage for default parameters and named call arguments.

The following local microbenchmark compares positional calls, hand written default handling, the proposed syntax, and options table alternatives. Each result is the median of 7 rounds with 5,000,000 iterations on the same local Windows build, so the values should be treated as prototype relative measurements rather than final cross platform performance data.

| Case | VM `-O2` | Native `--codegen` |
|---|---:|---:|
| Plain positional, inlineable | 0.052s | 0.018s |
| Plain positional, noinline | 0.198s | 0.122s |
| Manual defaults, full arguments | 0.187s | 0.052s |
| Manual defaults, noinline full arguments | 0.280s | 0.114s |
| Manual defaults, nil arguments | 0.221s | 0.073s |
| Manual defaults, noinline nil arguments | 0.306s | 0.118s |
| Default parameters, full arguments | 0.272s | 0.122s |
| Default parameters, noinline full arguments | 0.271s | 0.118s |
| Default parameters, nil arguments | 0.293s | 0.266s |
| Default parameters, noinline nil arguments | 0.290s | 0.290s |
| Default parameters, named argument | 0.286s | 0.265s |
| Options table, literal | 0.551s | 0.434s |
| Options table, reused | 0.329s | 0.212s |
| Options table, noinline literal | 0.655s | 0.528s |
| Options table, noinline reused | 0.430s | 0.234s |

This shows that default parameters and named arguments are substantially faster than options table calls that allocate table literals. Reused table cases are more mixed in the prototype, but the proposed syntax still preserves a direct function signature, keeps the call target statically visible, and avoids requiring a runtime keyword argument object or reflective lookup.

The prototype is not yet an universal performance win over hand written default checks. In the current implementation, functions with default parameters are excluded from bytecode inlining, and the native codegen path for default substitution still has optimization headroom. These are implementation issues rather than semantic requirements of the feature: named arguments can be lowered to positional arguments at compile time, and default checks only need to run for functions that declare defaults.

## Drawbacks

This adds new syntax and more behavior to the parser, type checker, compiler, formatter, and editor tooling. Even if the runtime behavior is simple, the language surface becomes larger.

Named call arguments make parameter names more significant. Renaming a parameter can become a breaking API change for callers that use named arguments, even though the underlying function type remains structurally compatible.

Defaulted parameters use `nil` as the absence marker. This is convenient for common optional argument patterns, but it means a function cannot distinguish an explicitly passed `nil` from an omitted argument for a parameter that has a default.

Code using this feature will not parse on older Luau versions. Existing Luau code remains compatible, but projects that adopt this syntax require a Luau version that supports it.

There is a small cost for functions that use default parameters, since the function needs to check whether defaulted parameters are `nil`. This cost is only paid by functions that opt into defaults. Calls using named arguments can be lowered to positional calls, so they do not need a runtime allocation or lookup mechanism.

The prototype also needs more performance work before landing. Default parameter functions should be reconsidered for inlining when their defaults are simple enough, and the native codegen lowering for missing or `nil` arguments should be reviewed so default substitution is closer to equivalent hand written checks.

The initial design also limits named arguments to calls where parameter names can be resolved statically. Supporting named calls through arbitrary function values would require either threading type information into compilation or adding runtime parameter metadata, neither of which is part of this proposal.

## Alternatives

The current approach is to write manual fallback checks in the function body:

```luau
local function connect(host: string?, port: number?, secure: boolean?)
    host = if host == nil then "localhost" else host
    port = if port == nil then 8080 else port
    secure = if secure == nil then false else secure
end
```

This works today, but it is verbose, easy to write inconsistently, and hides the default behavior inside the function body instead of exposing it in the function signature.

Another alternative is to use an options table:

```luau
local function connect(options)
    local host = options.host or "localhost"
    local port = options.port or 8080
    local secure = if options.secure == nil then false else options.secure
end

connect({ port = 443, secure = true })
```

Options tables are useful for large configuration objects, but they allocate a table, weaken the direct relationship between the function signature and its supported arguments, and are heavier than needed for ordinary functions with a few optional parameters.

Another alternative is to only add default parameters without named call arguments. This would reduce boilerplate inside functions, but callers would still need to pass `nil` placeholders to override later parameters.

Another alternative is to only add named call arguments without default parameters. This would improve call site readability, but would not remove the common manual fallback checks inside function bodies.

The proposed design combines both features because they address the same recurring problem: functions with optional parameters where callers often need to provide only a specific subset of values.
