# coroutine.finally

## Summary

Add a `coroutine.finally` function that registers a callback on a coroutine.
The callbacks fire when the coroutine reaches a terminal state: it returned, threw an unhandled error, or was closed via `coroutine.close`.
This is a low-level primitive needed to build structured concurrency in userspace.
Because the hook lives on the coroutine itself, independently authored concurrency frameworks can compose using it without additional coordination.

## Motivation

`coroutine.close` (added via a prior RFC) lets you *cancel* a coroutine externally.
There is no corresponding way to be *notified* when a coroutine reaches a terminal state.
Frameworks that manage coroutine lifecycles (task schedulers, promise libraries, nurseries) must work around this by wrapping `coroutine.resume` in bookkeeping, polling `coroutine.status`, or maintaining parallel tracking structures.
These workarounds are fragile and do not compose: two independent libraries that both want to observe the same coroutine's lifecycle cannot do so without coordinating on the same wrapper.

Structured concurrency requires two fundamental operations on a unit of work: **cancel** and **join**.
`coroutine.close` provides cancel.
`coroutine.finally` provides join.
Together they form the minimal pair needed to build higher-level primitives (nurseries, task groups, cancellation trees) entirely in userspace, anchored to the `thread` type that the VM already uses as its unit of execution.

Because the hook is registered on the coroutine directly, not on a wrapper object, it enables interoperability by default.
A task scheduler can register a callback to track completion while a separate logging library registers its own callback on the same coroutine for diagnostics, with no coordination between the two.

## Design

`coroutine.finally(co, callback)` registers a callback on coroutine `co` that will be called when `co` reaches a terminal state.

```luau
coroutine.finally(co: thread, callback: (status: "finished" | "error" | "cancelled", ...any) -> ())
```

Calling `coroutine.finally` is an error when:

- coroutine is already dead
- coroutine is the main thread of the VM

### Callback invocation

The callback is called with a status and a set of values depending on it.

- **Normal return:** `callback("finished", ...results)`. The coroutine function returned. All result values are forwarded to the callback.
- **Unhandled error:** `callback("error", err)`. The coroutine terminated with an error. The error value is passed as the only additional argument.
- **External close:** `callback("cancelled")`. `coroutine.close` was called on the coroutine. The coroutine did not complete its work. No extra values are provided.

As such, `coroutine.finally` forwards the complete coroutine outcome to callbacks.

### Multiple callbacks

Multiple callbacks can be registered on the same coroutine.
They are called in LIFO order (last registered, first called).

If a callback errors, callback invocations are interrupted and the callback error is returned to `coroutine.resume` and `coroutine.close` overriding the original one.
In case of a host `lua_runfinalizers` call, it reports the resulting outcome.
It is important to not have silent errors and allow debugging of callbacks.
Callbacks can use `pcall`/`xpcall` inside to not interrupt other listeners with their mistakes, but that choice is explicit.

### When callbacks fire

Callbacks fire synchronously after the coroutine has transitioned into its terminal state but before `coroutine.resume` or `coroutine.close` returns to its caller.
The thread that called `coroutine.resume` or `coroutine.close` executes the callbacks.

Each callback fires at most once.
Inside the callbacks, operations on a coroutine such as reset (`lua_resetthread`) and reuse are allowed and do not interfere with callback result delivery.

Callbacks are allowed to yield.

### Garbage collection

Callbacks are not fired when a coroutine is garbage collected without being closed or run to completion.
An unreachable coroutine is collected normally and its callbacks are discarded with it.

Callers that need guaranteed cleanup should ensure coroutines are either run to completion or explicitly closed via `coroutine.close`.

`lua_resetthread` does not run the callbacks, host must ensure the callbacks have been launched prior to reset or accept that callbacks will be dropped.

### C API

For the host to interface with callbacks, a set of functions is added:

```c
LUA_API int lua_hasfinalizers(lua_State* L);
```

Checks if 'finally' callbacks have been installed on a thread `L`.
Returns 1 if it has and 0 otherwise.

```c
LUA_API int lua_runfinalizers(lua_State* L, lua_State* co);
```

Runs the 'finally' callbacks of the thread `co` using thread `L`.

`co` cannot equal `L`.
`co` must have reached its completion.
`L` must be idle or reset.

Note that the host is not required to run 'finally' callbacks after each terminal `lua_resume`.
Host should call `lua_runfinalizers` after terminal resumes performed by their own scheduler (for example on threads that have used host-provided yielding primitives).
User-defined coroutine lifecycles using `coroutine.resume`/`coroutine.close` do not require host involvement.

Returns a status similar to `lua_resume`.

If the result status is `LUA_YIELD` or `LUA_BREAK`, callback execution is suspended on thread `L`.
Host must `lua_resume` the `L` at the appropriate time and will observe the results on it, `co` can be discarded.

Otherwise, the result behavior follows the terminal `coroutine.resume`, but the status `boolean` is not placed on the stack.

```c
LUA_API void lua_addfinalizer(lua_State* L, lua_State* co, int idx);
```

Adds the 'finally' callback at index `idx` to the thread `co`.

Callback cannot be `nil`.

An error is raised if:

- `co` is already dead
- `co` is the main thread of the VM

```c
struct lua_Callbacks
{
    ...
    void (*userfinalizer)(lua_State* L, lua_State* co);
    ...
};
```

When set, called just before a 'finally' callback is about to be installed by thread `L` on `co`.
This allows the host to observe the attempt or raise an error to prohibit the installation.

### coroutine.wrap

`coroutine.wrap` will error when it runs to termination and there are callbacks installed.
This is because `coroutine.wrap` is intended to wrap an anonymous coroutine inside a function call and the thread can only be extracted by indirect `coroutine.running` means.

### Examples

#### Cancellation cascade

When a parent coroutine is closed, its children are closed automatically.
Without `coroutine.finally`, there is no way for external code to arrange this cleanup.

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
    coroutine.yield()       -- parent yields back to caller
end)

coroutine.resume(parent) -- starts parent, which starts child
coroutine.close(parent)  -- parent's finally fires -> child is closed
```

This extends to arbitrary depth.
A helper function can wire up the parent-child relationship for any coroutine:

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

A task group spawns concurrent work and waits for all of it to finish.
It uses `coroutine.finally` on each child to detect completion (the "join" side) and on the parent to cascade cancellation (the "cancel" side).

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

        coroutine.finally(co, function(status, ...)
            children[co] = nil
            if status == "error" then
                table.insert(errors, (...))
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
local group = coroutine.create(function()
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
end)

coroutine.resume(group)
```

#### Composable observers

Two independent libraries can observe the same coroutine without coordinating on a shared wrapper.
This is the key advantage over wrapping `coroutine.resume`: wrapping doesn't compose because two wrappers on the same coroutine will conflict.

```luau
local co = coroutine.create(work)

-- Library A: scheduler tracks active coroutines
coroutine.finally(co, function()
    activeTasks[co] = nil
end)

-- Library B: diagnostics logs failures
coroutine.finally(co, function(status, ...)
    if status == "error" then
        log("task failed:", ...)
    end
end)

coroutine.resume(co)
-- Both callbacks fire (LIFO: B runs first, then A).
```

## Drawbacks

This adds per-coroutine state, which slightly increases memory usage for coroutines.

Coroutine result or error is discarded if a callback errors.
Callback erroring also prevents other callbacks from running.
An alternative would be to aggregate errors or propagate the original one of the completed coroutine, at the cost of a more complex error model.

If a coroutine is abandoned (becomes unreachable without being closed or run to completion), its finally callbacks are collected by the GC and never fire.

## Alternatives

Lua 5.4 introduced to-be-closed variables (`local <close> x = resource()`), where a variable marked with `<close>` has its `__close` metamethod called automatically when the variable goes out of scope, whether by normal control flow or an error.
This gives Lua 5.4 scoped, deterministic cleanup similar to RAII in C++ or `defer` in Go.
One major difference is that to-be-closed variables require new language syntax and VM support for the variable annotation.
`coroutine.finally` provides similar cleanup guarantees as a plain library function with no other changes to the language.

Frameworks can poll `coroutine.status` to detect completion, but this requires either busy-waiting or inserting checks after every resume call.
It is error-prone and does not compose.

Frameworks can wrap `coroutine.resume` to intercept completion.
Two frameworks wrapping the same coroutine's resume path will conflict unless they coordinate.
`coroutine.finally` avoids this by supporting multiple independent callbacks natively.

`coroutine.close` could be enhanced to accept an optional reason parameter (`coroutine.close(co, reason)`) that would be forwarded to finally callbacks, letting frameworks communicate *why* a coroutine was cancelled (timeout, parent cancellation, sibling error).
This is out of scope for this proposal but fully compatible as a future addition.

`coroutine.finally` could return a handle to deregister the callback.
This adds complexity to the API surface; the minimal design here can be extended later if the need arises.

The name `coroutine.onclose` was considered but rejected because the callback fires on all terminal events (return, error, close), not just `coroutine.close`.
The name `finally` communicates "runs regardless of outcome," mirroring the widely understood semantics of `try/finally` in other languages.
