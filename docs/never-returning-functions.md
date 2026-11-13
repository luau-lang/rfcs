# Control-flow analysis for `never`-returning functions

## Summary

Use function return types to determine whether calls can return normally during control-flow analysis. A call to a function returning `never` terminates the current control-flow path.

## Motivation

Some functions cannot return normally. They may raise an error, enter an infinite loop, or otherwise prevent execution from continuing. Their return type should communicate this behavior to the type checker, allowing it to recognize unreachable code and account for the absence of a normal return path.

Luau already declares `error` as returning `never`. However, `error`'s behavior is currently handled as a special case in control-flow analysis rather than derived from the function's return type. Other functions with the same guarantee, including user-defined functions and Roblox APIs such as `LogService:Error`, cannot benefit from the same behavior.

## Design

We will make a call to a function whose resolved return type is `never` terminate the current control-flow path. Subsequent statements reachable only through that path are considered unreachable.

```luau
function fail(message: string): never
    error(message)
end

fail("something went wrong")

print("unreachable") -- unreachable
```

This also affects return-path analysis. A function does not need to explicitly return after calling a function that cannot return normally:

```luau
function parse(input: string): string
    if input == "" then
        fail("input cannot be empty")
    end

    return input
end
```

A function declared to return `never` must not have any reachable path that completes normally. For example, the following function is valid because every path either raises an error or diverges:

```luau
function loop(condition: boolean): never
    if condition then
        error("failed")
    else
        while true do
            task.wait()
        end
    end
end
```

A function with a reachable path that falls through normally is not compatible with a `never` return type:

```luau
function fail(condition: boolean): never
    if condition then
        error("Failed")
    end
    -- reachable
end
```

The rule is based on the resolved return type of the call. For generic functions, the return type is considered after type arguments have been instantiated. If the resulting return type is `never`, the call is `never`-returning.

We will not introduce control-flow-based return type inference, at least for the time being. In particular, we will not infer `never` solely because a function body has no reachable path that completes normally. The return type used for control-flow analysis must be available from the function's resolved signature, either through an explicit annotation or existing return type inference.

For example, a function whose return expression has type `never` can still have an inferred `never` return type:

```luau
local function fail()
    return error("failed")
end
```

Inferring `never` for functions that do not explicitly return a `never`-typed expression, but instead only raise errors or diverge, is outside the scope of this proposal.

## Drawbacks

An incorrect `never` annotation can cause code to be considered unreachable even when the function returns normally at runtime. As with other return types, the type checker relies on function signatures accurately describing their behavior.

## Alternatives

* Do nothing
* Introduce a function attribute such as `@noreturn`, which has precedent in other languages. However, Luau already represents `never`-returning functions such as `error` with the `never` return type
* Determine whether a function is `never`-returning by analyzing all of its control-flow paths. However, this cannot work for declaration files, where the function body is not present, so annotations would still be necessary

## Prior Art

* Luau already declares `error` as returning `never` even though its control-flow behavior is hardcoded.
* TypeScript has control-flow analysis for calls to `never`-returning functions. Calls to eligible functions with explicitly declared return types affect control flow, including unreachable-code detection and return-path analysis. TypeScript also restricts which calls participate in this analysis to avoid triggering additional type inference during control-flow analysis.