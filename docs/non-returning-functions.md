# Non-returning functions

## Summary

Treat calls to functions returning `never` as non-returning in control-flow analysis.

## Motivation

Annotating functions as non-returning is useful for functions that cannot successfully complete, such as functions that unconditionally raise an error or otherwise terminate execution. Luau already annotates `error` as returning `never`, but control-flow analysis does not currently use this information to determine that execution cannot continue after a call.

## Design

A call expression with a return type of `never` is considered non-returning. Control flow does not continue after such a call.

```luau
local function fail(): never
    error("fail")
end

fail()

print("unreachable")
```

In the example above, the call to `fail` is non-returning, so the following `print` call is unreachable.

A function declared to return `never` must have no reachable path that completes normally:

```luau
local function fail(): never
    if condition then
        error("fail")
    else
        error("fail")
    end
end
```

The following function is invalid because it has a reachable path that completes normally:

```luau
local function fail(): never
    if condition then
        error("fail")
    end
end
```

The rule applies to the resulting return type of a call, so generic functions whose return type is instantiated as `never` are also considered non-returning.

## Drawbacks

This proposal gives `never` significance in control-flow analysis in addition to its existing role as an uninhabited type.

## Alternatives

* As always, do nothing.
* Introduce a function attribute such as `@noreturn`, which has precedent in other languages. However, Luau already represents non-returning functions such as `error` with the `never` return type.
* Determine whether a function is non-returning by analyzing all of its control-flow paths. However, this cannot work for declaration files, where the function body is not present, so annotations would still be necessary.

## Prior Art

* **TypeScript:** Calls to functions with an explicit `never` return type are treated as non-returning by control-flow analysis. The return type of the call is what determines this behavior, including after generic type instantiation.
* **Rust:** The `!` never type is also used to represent diverging expressions. Calls producing `!` are treated as diverging, and functions returning `!` must not have a reachable path that completes normally.