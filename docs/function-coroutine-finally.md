# coroutine.finally

## Summary

Add a `coroutine.finally` function that registers a callback on a coroutine. The callback fires exactly once when the coroutine reaches a terminal state: it returned, threw an unhandled error, or was closed via `coroutine.close`. This is a low-level primitive needed to build structured concurrency in userspace. Because the hook lives on the coroutine itself, independently authored concurrency frameworks can compose using it without additional coordination.

## Motivation

`coroutine.close` (added via a prior RFC) lets you *cancel* a coroutine externally. There is no corresponding way to be *notified* when a coroutine reaches a terminal state. Frameworks that manage coroutine lifecycles (task schedulers, promise libraries, nurseries) must work around this by wrapping `coroutine.resume` in bookkeeping, polling `coroutine.status`, or maintaining parallel tracking structures. These workarounds are fragile and do not compose: two independent libraries that both want to observe the same coroutine's lifecycle cannot do so without coordinating on the same wrapper.

Structured concurrency requires two fundamental operations on a unit of work: **cancel** and **join**. `coroutine.close` provides cancel. `coroutine.finally` provides join. Together they form the minimal pair needed to build higher-level primitives (nurseries, task groups, cancellation trees) entirely in userspace, anchored to the `thread` type that the VM already uses as its unit of execution.

Because the hook is registered on the coroutine directly, not on a wrapper object, it enables interoperability by default. A task scheduler can register a `finally` callback to track completion while a separate logging library registers its own callback on the same coroutine for diagnostics, with no coordination between the two.

## Design

`coroutine.finally(co, callback)` registers a callback on coroutine `co` that will be called when `co` reaches a terminal state.

```luau
coroutine.finally(co: thread, callback: (boolean, any) -> ())
```

### Callback invocation

The callback is called with two arguments `(ok, err)`:

- **Normal return:** `callback(true, nil)`. The coroutine's function returned. Return values are not forwarded to the callback; they are available to the caller through `coroutine.resume`'s own return values.
- **Unhandled error:** `callback(false, err)`. The coroutine terminated with an error. The error value is passed as the second argument **when available.**
- **External close:** `callback(false, "coroutine was closed")`. `coroutine.close` was called on the coroutine. The coroutine did not complete its work. A default error message is provided so callbacks can identify cancellation without external bookkeeping.

`coroutine.finally` is a control-flow abstraction, not a data-flow abstraction. The boolean is the only reliable signal. The second argument is an error diagnostic: for unhandled errors and external close it is available when the callback fires inline during `coroutine.resume` or `coroutine.close`, but is not guaranteed in all cases (see "Already-dead coroutines" below). Callers that need the coroutine's return values should use `coroutine.resume`'s own returns.

### Multiple callbacks

Multiple callbacks can be registered on the same coroutine. They are called in LIFO order (last registered, first called). Each callback is implicitly called via `pcall`; if a callback errors, the error is discarded and the remaining callbacks still execute.

### When callbacks fire

Callbacks fire synchronously after the coroutine has transitioned into its terminal state but before `coroutine.resume` or `coroutine.close` returns to its caller. The thread that called resume or close executes the callbacks. If `coroutine.finally` is called on a coroutine that is already dead, the callback fires immediately before `coroutine.finally` returns (see below).

After all callbacks have fired, they are removed and will not fire again. Each callback fires exactly once, or not at all if the coroutine yields indefinitely.

### Already-dead coroutines

If `coroutine.finally` is called on a coroutine that is already dead, the callback fires immediately on the thread that called `coroutine.finally`. The callback receives `(true, nil)` if the coroutine finished normally, or `(false, nil)` if it errored or was closed. The original error value is no longer available because the coroutine has already been reset.

### Garbage collection

Callbacks are not fired when a coroutine is garbage collected without being closed or run to completion. An unreachable coroutine is collected normally and its callbacks are discarded with it.

Callers that need guaranteed cleanup should ensure coroutines are either run to completion or explicitly closed via `coroutine.close`.

### Examples

#### Cancellation cascade

When a parent coroutine is closed, its children are closed automatically. Without `coroutine.finally`, there is no way for external code to arrange this cleanup.

```luau
local parent = coroutine.create(function()
    local child = coroutine.create(function()
        print("child started")
        coroutine.yield()
        print("child finished") -- never reached
    end)

    coroutine.finally(coroutine.running(), function()
        coroutine.close(child)
    end)

    coroutine.resume(child) -- child prints "child started", then yields
    coroutine.yield()        -- parent yields back to caller
end)

coroutine.resume(parent) -- starts parent, which starts child
coroutine.close(parent)  -- parent's finally fires → child is closed
```

This extends to arbitrary depth. A helper function can wire up the parent-child relationship for any coroutine:

```luau
local function spawnChild(fn)
    local child = coroutine.create(fn)
    coroutine.finally(coroutine.running(), function()
        coroutine.close(child)
    end)
    coroutine.resume(child)
    return child
end
```

#### Task group

A task group spawns concurrent work and waits for all of it to finish. It uses `coroutine.finally` on each child to detect completion (the "join" side) and on the parent to cascade cancellation (the "cancel" side).

```luau
local function taskGroup(fn)
    local parent = coroutine.running()
    local children = {}
    local errors = {}
    local cancelled = false

    coroutine.finally(parent, function()
        cancelled = true
        for co in children do
            coroutine.close(co)
        end
    end)

    local function spawn(work)
        local co = coroutine.create(work)
        children[co] = true

        coroutine.finally(co, function(ok, err)
            children[co] = nil
            if not ok and err ~= nil then
                table.insert(errors, err)
            end
            if not cancelled and next(children) == nil then
                coroutine.resume(parent)
            end
        end)

        return coroutine.resume(co)
    end

    fn(spawn)

    if next(children) ~= nil then
        coroutine.yield()
    end

    if #errors > 0 then
        error(errors[1])
    end
end
```

Usage:

```luau
taskGroup(function(spawn)
    spawn(function()
        fetchData("endpoint-a")
    end)
    spawn(function()
        fetchData("endpoint-b")
    end)
end)
-- Continues here only after both tasks finish.
-- If either task errored, the first error is re-thrown.
```

#### Composable observers

Two independent libraries can observe the same coroutine without coordinating on a shared wrapper. This is the key advantage over wrapping `coroutine.resume`: wrapping doesn't compose because two wrappers on the same coroutine will conflict.

```luau
local co = coroutine.create(work)

-- Library A: scheduler tracks active coroutines
coroutine.finally(co, function()
    activeTasks[co] = nil
end)

-- Library B: diagnostics logs failures
coroutine.finally(co, function(ok, err)
    if not ok then
        log("task failed:", err)
    end
end)

coroutine.resume(co)
-- Both callbacks fire (LIFO: B runs first, then A).
```

## Drawbacks

This adds per-coroutine state (a callback list stored in a weak-keyed registry table), which slightly increases memory usage for coroutines that register finally callbacks. Coroutines without registered callbacks pay no cost beyond the registry table lookup on terminal events.

Callbacks run synchronously before `coroutine.resume` or `coroutine.close` returns. This is the simplest and most predictable execution model, but it may surprise users who expect async behavior. A callback that performs expensive work will delay the return of resume/close.

Callback errors are silently discarded. This is a deliberate trade-off: one misbehaving callback should not prevent other callbacks from running or corrupt the resume/close return path. The downside is that bugs in finally callbacks can be difficult to diagnose. An alternative would be to aggregate errors or propagate the first one, at the cost of a more complex error model.

If a coroutine is abandoned (becomes unreachable without being closed or run to completion), its finally callbacks are collected by the GC and never fire.

## Alternatives

Lua 5.4 introduced to-be-closed variables (`local <close> x = resource()`), where a variable marked with `<close>` has its `__close` metamethod called automatically when the variable goes out of scope, whether by normal control flow or an error. This gives Lua 5.4 scoped, deterministic cleanup similar to RAII in C++ or `defer` in Go. One major difference is that to-be-closed variables require new language syntax and VM support for the variable annotation. `coroutine.finally` provides similar cleanup guarantees as a plain library function with no other changes to the language.

Frameworks can poll `coroutine.status` to detect completion, but this requires either busy-waiting or inserting checks after every resume call. It is error-prone and does not compose.

Frameworks can wrap `coroutine.resume` to intercept completion. Two frameworks wrapping the same coroutine's resume path will conflict unless they coordinate. `coroutine.finally` avoids this by supporting multiple independent callbacks natively.

`coroutine.close` could be enhanced to accept an optional reason parameter (`coroutine.close(co, reason)`) that would be forwarded to finally callbacks, letting frameworks communicate *why* a coroutine was cancelled (timeout, parent cancellation, sibling error). This is out of scope for this proposal but fully compatible as a future addition.

Today, a coroutine's return values are only available to the immediate caller of `coroutine.resume`. Preserving them on the coroutine would decouple *producing* a result from *consuming* it, which is a prerequisite for building higher-level concurrency abstractions like synchronous await or joinable tasks. The coroutine memory model could be revised to preserve a coroutine's return values on the VM stack after it reaches a terminal state, rather than discarding them when the coroutine reaches a terminal state. This would allow `coroutine.finally` callbacks to receive the full return values instead of just the `(ok, err)` signal, and would enable new APIs that read a completed coroutine's results without requiring the caller to have been the one that called `coroutine.resume`. The current design is fully compatible with that future extension to the coroutine memory model, where if return values are later preserved, they could be forwarded to callbacks or exposed through a new API without any changes to `coroutine.finally`'s semantics. `coroutine.finally` is *at a minimum* a control-flow primitive: the `(ok, err)` callback signature provides the boolean completion signal and error diagnostic needed for lifecycle management. Forwarding return values is a data-flow concern that belongs in a separate proposal.

`coroutine.finally` could return a handle to deregister the callback. This adds complexity to the API surface; the minimal design here can be extended later if the need arises.

The name `coroutine.onclose` was considered but rejected because the callback fires on all terminal events (return, error, close), not just `coroutine.close`. The name `finally` communicates "runs regardless of outcome," mirroring the widely understood semantics of `try/finally` in other languages.
