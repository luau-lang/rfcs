# Runtime Stack Library

## Summary

A library for manipulating the coroutine stack at the language level, with explicit control, in the form of limited addressability of execution continuations.

## Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

Today, to move from function A to B, an explicit `return` is required. But what if there are many such functions? What if there are even more than many? We may benefit from behavior similar to C's `goto`, but safer and less intrusive. This library allows Luau to use controlled nonlinear transfer of control between parts of Lua execution without forcing the user to thread `return`, `yield`, callbacks, state, and additional continuation objects through every call level. In essence, it solves this:

```md
A() -> B() -> C()
            -> if a then return else b
        -> if c then b else return
    -> if if if if if if if if .................... else return SPECIAL_SIGNAL if SPECIAL_SIGNAL then return if SPECIAL_SIGNAL then return if SPECIAL_SIGNAL then return if SPECIAL_SIGNAL then return
```

Ordinary Lua control flow expresses local execution well: call → result → return. But when a decision made deep inside the execution tree must substantially change execution further up, we have to manually build protocols and huge trees of repetitive boilerplate that only get in the way of understanding things later: "what are we even supposed to do here, there are 92394 signaling protocols, and I need to write the AI code" — `stack` provides a separate mechanism specifically for such cases.

We can also represent the execution stack as a programmable structure rather than merely the result of call/return. A conventional VM primarily treats the stack as a history of calls. The library proposes using it as a control space as well. In other words, execution state becomes addressable.

Provide non-local control while preserving safety boundaries. The main idea of `stack` is to make non-local control spatially constrained to the current execution context:

```md
    StackContext
    ├── A
    │    ├── mark("X")
    │    └── B
    │         └── C
    │              └── goto("X")
    │
    └── lock()
        └── D
            └── X inaccessible
```

The motivation is therefore also to obtain something more powerful than `return`, but safer than arbitrary C `goto`. The `stack` library is not intended to replace `return` or coroutines, but to provide the language with an explicit mechanism for controlling nonlinear execution flow when the ordinary call/return model is insufficient. In modern Lua-like VMs, execution state is almost entirely defined by the current `CallInfo` and the linear call history. `stack` explores an alternative model in which execution state can be preserved, addressed, and continued independently of the ordinary lifetime of an individual `CallInfo`.

The `stack` library is not an extension of the coroutine API. It is an alternative execution-state model built on top of the coroutine runtime. Its primary cost and benefit lie not in the number of files that must be changed, but in requiring the VM to support execution-state lifetime independently of the ordinary `CallInfo` lifetime, allowing users to successfully employ mechanisms with a high level of complexity.

Conceptually, the problem looks like this:

```md
                     Runtime Stack
                            │
            ┌──────────────┴──────────────┐
            │                             │
    Context Transfer              Execution Graph
            │                             │
    mark / hit / goto / on / lock   enter / reentered / split
            │                             │
    non-local transfer              structural change
    ancestry                         in execution
    boundaries                      re-entry
                                    branching
```

## Design

This is the bulk of the proposal. Explain the design in enough detail for somebody familiar with the language to understand, and include examples of how the feature is used.

To develop this library, we need several `struct`s that allow us to navigate the space correctly. At minimum, these are `StackContext`, `StackMark`, and `StackContinuation`. Some functionality will require changes to `CallInfo`, additional data in `Frame`, and possibly a `StackBranch`.

For the user experience, the design is much simpler. To understand how it works, it is enough to read the description below.

The library itself:

```luau
    declare type stack = {
        read mark: (name: string) -> ()
        read hit: (name: string) -> (boolean, ...any)
        read goto: (name: string, ...any) -> ()

        read reentered: () -> (boolean, ...any)
        read enter: (layer: number?, ...any) -> ()

        read count: () -> number
        read lock: (name: string?) -> ()

        read on: (that: ((...any) -> (...any)|any)|thread) -> boolean
        read split: () -> (boolean, ...any)
    }
```

Most of the questions this library addresses concern functional programming, but because it has its own context, it can extend the ways functions may execute to an even larger scale. In essence, results from child functions can be acted upon in the same way as if this were C's `goto label`. This will naturally lead to both positive aspects and total anarchy if misused, but the tool itself can be extremely useful, at least for logical components.

I strongly do NOT recommend considering all of this functionality interchangeable. No, that is fundamentally incorrect, and moreover, different parts of the library will require substantial changes to the LVM if you try to structure and prototype everything at once. Split it into two parts: the execution-context transfer layer (`mark/hit/goto/lock/on`) and the execution-graph STRUCTURING layer (`split/enter/reentered`). For understanding and implementation, execution-context transfer is the simpler part; the second part is conceptually straightforward but incredibly difficult to implement. The Drawbacks section will discuss this at length.

In advance: for all functions, interaction with C functionality or functionality through userdata is unavailable; these are `internal` LVM calls and internal LVM processing. Stack continuations may cross Lua execution frames, but they must terminate at execution-environment boundaries that do not permit re-entry.

Examples follow.

```luau

    const state, out = stack.split()
    -- Imagine the following situation: the function is executing in the middle of its context.
    -- This call makes the function consider itself split into BEFORE and AFTER.
    -- split() creates a child continuation state that shares available lexical/environment
    -- references, but has an independent instruction position and execution storage.
    --
    -- The example becomes:
    --
    ```luau
        local function foo()
            local run = coroutine.running()
            local x = 1
            local y = 2

            const state, out = stack.split()

            local z = 3

            if stack.on(run) then
                print ("We are executing in a sub coroutine", state, out, x, y, z)
                -- Output will be: string, nil, nil, number, number, number
                -- This is because state and out do not exist at that point, and never could have existed.
                -- The output of xyz will be correct.
            else
                print("we executed the sub coroutine and can now safely process it", state, out)
                -- Output will be: string, boolean, any, number, number, number
                -- The only difference is that x and y may be redefined in the sub coroutine, while
                -- z will always statically be 3 after the sub coroutine finishes, because both here
                -- and there, variable z is declared immediately after stack.split(), or elsewhere afterward,
                -- which forces us to redefine a new variable for the context.
                --
                -- If z were const, it would be identical for any context, and the same applies to x and y,
                -- because const is a safe boundary. This can either be ignored, because z can be simply
                -- redefined in both the sub coroutine and the main coroutine, or it can be used as an
                -- optimization, because the variable can easily be placed in a shared buffer - it will
                -- never change anyway. But this applies only to values that the compiler knows statically,
                -- if we previously knew that the value directly contains a number or string or another value
                -- that is NOT a reference type.
                --
                -- Reference types follow a strict rule: all references that are CREATED to an object in the
                -- space which did not previously exist - table, userdata, thread, etc. - will always refer
                -- to a new unique object. Simply put, if before stack.split() we observed a local/const
                -- reference to a reference-type object, we just obtain it and do nothing; otherwise, we create
                -- a new object.
            end
        end
    ```
    --
    -- This allows us to write dangerous code and handle it right there without involving a protection
    -- protocol such as pcall/xpcall, because a crashing coroutine will not crash the stack of its source,
    -- thereby providing a high degree of nonlinearity in programmable code where it is needed.

    const is = stack.on(function or thread)
    -- Returns a boolean indicating whether the given function/coroutine is present in the stack or not.
    -- Simply put, for passed functions we check whether they are present in the ancestry from the current
    -- coroutine position, while for coroutines we check that their ancestry originates from a particular
    -- coroutine. That is, if we previously declare run = coroutine.running() and after stack.split(),
    -- or coroutine.run(coroutine.create(function () end)), pass stack.on(run) inside a function, the result
    -- will be true because that coroutine is present in the ancestry.
    --
    -- Exceptions are operations such as pcall/xpcall and task. For task, stack.lock() can simply be built in;
    -- stack.lock() is described below. For pcall/xpcall, stack.lock() will automatically be applied to the
    -- functions they call. Why?
    --
    -- The reason is that traversing the stack of a coroutine requires preserving the safety of both the
    -- coroutine itself and its private continuation space for stable operation. A region of execution that we
    -- deliberately wrap in pcall/xpcall forces us to regard the next region as potentially extremely dangerous
    -- to our existence. Therefore, allowing it to see our temporal marks and move through the stack beyond the
    -- protection field is forbidden. This region sees only its own stack: that means calling stack.on() for
    -- functions is unavailable. This does not apply to coroutines, because we can extract the coroutine as
    -- a closure value and compare it directly inside the code, coroutine.running() == run.
    --
    -- Essentially, calling stack.on() inside pcall/xpcall will not throw errors, but for functions it will
    -- always return false if they were called across the barrier.

    stack.mark(string)
    -- In C, there is goto label semantics, which allows a program to jump statically between different
    -- execution regions and perform various operations. This extremely unsafe construct is nevertheless a
    -- very serious tool, and stack is interested in implementing it.
    --
    -- This feature is implemented by storing, on the current coroutine stack and under the specified label,
    -- the entire current frame relative to the position of stack.mark() in the space. The variables declared
    -- here will always be preserved while the coroutine stack contains this context. As soon as the coroutine
    -- leaves this context, memory reclamation remains the same as before, and GC can safely destroy the frame.
    --
    -- To provide safety, functionality such as
    ```luau
        local function foo()
            stack.mark("A")
        end

        local function bar()
            stack.goto("A")
        end
    ```
    --
    -- will be restricted: we cannot redefine the same mark after it has been declared.
    -- It remains true after it is reached, but only for the part of the coroutine's stack tree that is
    -- located in the region where stack.mark() was declared, immediately after it, or lower in the
    -- coroutine stack. If we nevertheless try to redefine this mark, we get
    -- error("cannot reuse an already defined label in the space").
    --
    -- Essentially, stack.mark() creates an `execute from A with continuation environment` point in
    -- the coroutine's space.

    const is, out = stack.hit(string)
    -- We can check whether we have previously jumped to this mark and also obtain arguments from that
    -- operation.
    --
    -- Since the mark is available for declaration only once, stack.hit() also works like stack.reentered():
    -- it remembers the movement context only for this function, and when trying to read stack.hit() outside
    -- the function where stack.mark was declared, it returns false, because for a function below the one where
    -- stack.mark() was declared this spatial mark does not exist, or does not belong to it, which is more precise.

    stack.goto(string, ...any)
    -- The goto operation forces the coroutine to abandon its current context: all operations performed up to
    -- this point are lost, but only up to the target mark. The coroutine stack remembers that it previously
    -- entered some sub coroutine or function, and this can be observed through stack.on(). However, it is not
    -- possible to recover all values that were in scope at that time. stack.goto() passes only what it received
    -- in its variadic arguments, and carries only that.
    --
    -- If we try to jump to a mark declared beyond stack.lock(), or try to jump to a mark while we are in a
    -- pcall/xpcall context and that mark is BEYOND it, we receive an error such as
    -- error("stack elevation above the limit is not available in a secure environment").
    --
    -- When using stack.goto(), the coroutine frame that is currently active is destroyed, and we move to the
    -- spatial mark stack.mark() with the state it had at that moment.

    stack.enter(nil or 0 or number, ...any)
    -- If we receive nil or 0, we move up one level and immediately re-enter the same function from which
    -- stack.enter() was called. If we receive 1, we move up two levels and enter, in the stack, the function
    -- that called the function where stack.enter() is defined. We can pass arbitrary values, but not values
    -- that belonged to that line at the moment stack.enter() was called.
    --
    -- This function has restrictions: we can move up one level at most 80 times, as if we were attempting
    -- to launch task 80 times. The limit prevents infinite recursion or endless calls that could cause us
    -- to lose control over the current context. This throws the error
    -- error("the limit of stack raises is exceeded").
    --
    -- The function will not allow moving up the coroutine stack across a pcall/xpcall barrier. We receive
    -- an error such as error("stack elevation above the limit is not available in a secure environment").
    --
    -- Negative values return error("can't move on a stack that doesn't exist"). Values that exceed the depth
    -- of stack.count(), meaning values beyond the number of raises that the coroutine can actually perform on
    -- its own stack - essentially the number of stack steps - return error("an attempt to move along the stack to where there is nothing").
    --
    -- This function has no relation to stack.mark()/stack.goto()/stack.hit(); it is a separate mechanism.
    -- In addition, this function forces GC to retain the current FrameContext so that entry into executors
    -- higher in the stack remains possible. Data can therefore be reclaimed strictly after the coroutine
    -- destroys the lower stack frame. Alternatively, all these values should be considered closure upvalues, because...

    const is, out = stack.reentered()
    -- We have an understanding of what stack.mark() and stack.goto() do: they essentially provide a way to
    -- move upward through the coroutine stack and allow us to travel through stack space so freely that we
    -- could become meta-travelers and perform completely different operations wherever we want.
    --
    -- The function returns stack.reentered() separately for each function. In essence, it takes
    -- debug.info(2, "f") and checks whether the coroutine has entered this same function before. This applies
    -- only in the context of the function into which stack.enter() was executed, while functions farther up
    -- the stack or other coroutines return false unless stack.enter() was also entered there.
    --
    -- The function does not return true if there was a transition through stack.goto(); stack.hit() is used
    -- for that. It also does not react to recursive entries; it reacts only to stack.enter().

    stack.lock()
    -- All these operations allow us to freely travel through the coroutine stack. However, they reduce the
    -- safety of our execution. This may lead to a situation where stack.mark("1") was declared somewhere deep
    -- in the code, and later somewhere below we perform the same stack.mark("1") and the program crashes with
    -- an error about reusing marks.
    --
    -- Or we may attempt to traverse the coroutine stack in places where we are not allowed to: try to pass
    -- through protected regions or execute unsafe code where it is forbidden.
    --
    -- To avoid this, stack.lock() cuts off visibility for functions and execution context below stack.lock()
    -- in the coroutine's own stack. This gives us the right to redefine previously declared stack.mark() values,
    -- but prevents us from ...
    --
    -- If stack.lock() is called with a mark name, its behavior is the same as above, but only for that mark:
    -- it will not be possible to reach it again. This also makes it possible to release resources associated
    -- with that mark.

```

For simpler understanding, I suggest formalizing the VM model with four entities. I think this will simplify the entire RFC considerably:

```md
    Activation
        current function execution

    Continuation
        saved Activation state

    Boundary
        a limit that cannot be crossed

    Context
        a tree of available Continuations
```

Then:

```md
    mark()
        create Continuation

    hit()
        inspect Continuation state

    goto()
        discard descendants + resume Continuation

    on()
        inspect ancestry

    lock()
        create Boundary

    up()
        resume ancestor Activation

    reentered()
        inspect Activation identity

    split()
        clone Continuation
```

## Drawbacks

Why should we *not* do this?

This functionality expands control over one's own code to a level partially comparable to C `goto`, but the cost of such an extension is extremely high. This is not an ordinary standard-library API extension, and not even an isolated coroutine capability: full implementation requires changing a substantial portion of the virtual machine and defining new rules for the lifetime of execution frames.

To begin with, such a change requires an absolutely enormous modification to a large part of the LVM:

```md
VM/src/lstate.h
    CallInfo / lua_State / StackContext
VM/src/lstate.cpp
    new/rem runtime state
VM/src/ldo.h
VM/src/ldo.cpp
    stack goto/enter/reenter
    continuation management
    frame unwinding
    boundary handling
VM/src/lvm.h
    internal stack runtime API
VM/src/lvmexecute.cpp
    interpreter integration
    dispatch
    reentry/goto exits
VM/src/lgc.cpp
VM/src/lgc.h
    marking StackContinuation / StackBranch
    thread relations
VM/src/lfunc.cpp
VM/src/lfunc.h
    upvalue / closure interactions
VM/src/lapi.cpp
Compiler/src/
    new bytecode operations
CodeGen/src/
    native/JIT handling
    deoptimization
```

More operations may need to be added, or behavior may need to be changed even more drastically. It is NOT A FACT that native codegen or JIT will be able to handle this. Native codegen may implement jumps through the relevant operations, while JIT may simply NOT implement them at all, because this is literally a jump to other instructions back through the coroutine's stack space.

And this is only what immediately comes to mind. In practice, the changes will affect even more areas of the runtime, because existing Luau assumes a fairly strict `CallInfo` lifetime model: a call creates a frame, execution moves forward, `return` destroys the corresponding state, and the stack and upvalues are adjusted according to the normal lifetime of the function.

But stack.mark() breaks that assumption. After a mark is created, an execution frame may have to remain accessible for a non-local continuation. Therefore, the VM must distinguish between a frame that is no longer used by ordinary execution and a frame that has already been left but is still part of an existing continuation.

This means GC can no longer be treated purely as a collector of Lua objects. The runtime must additionally track saved execution state and guarantee that it does not retain pointers to invalidated stack slots, `CallInfo`, open upvalues, or other VM structures.

`stack.split()` is especially difficult. In this case we have not merely a saved entry point, but multiple execution paths existing simultaneously. They may share Lua objects, closures, and upvalues while having independent `pc`, `CallInfo`, and local-value state. This effectively requires the VM to support branching continuation state.

That immediately complicates memory management. GC must understand the relationship between these states and must not release one execution context before another. Open upvalues become a separate problem because they can point directly to stack slots.

In addition, we must define what happens to side effects. `stack.split()` cannot create a complete copy of the program world: both branches may continue to reference the same state object. Therefore, execution state is copied, but the surrounding world state is not. This creates a complicated semantic model. Without a very strict specification, users will expect `split()` to behave as a full snapshot/clone of a coroutine, while in reality it would only split execution.

Another separate problem is that stack.goto() and stack.enter() are not ordinary jumps through bytecode. `stack.goto()` can destroy several active `CallInfo`s and restore the state of an older continuation. `stack.enter()` can not simply return to the caller, but re-run an older frame. The Luau VM can no longer assume that control flow follows only the three previously known paths; an additional axis appears. This complicates practically every location where the runtime assumes call/return sequencing, including yield/resume, error handling, protected calls, C continuations, and debugger integration.

`pcall`/`xpcall` are especially dangerous. They are already boundaries for error handling and continuation. Adding another non-local transfer mechanism means the runtime must strictly define which protected boundaries may be crossed and which may not.

The same applies to task, yieldable C functions, coroutine.resume(), and any other mechanisms that can save an execution continuation. The compiler also becomes more complex.

If stack.mark("A") becomes a runtime-visible label, the compiler can no longer freely consider arbitrary portions of local control flow unreachable. It needs to understand that some bytecode region may be reached through a non-local jump. The compiler must guarantee that state x accessible to a continuation really corresponds to the specified semantics. During optimization, local state cannot simply be moved, eliminated, or reused as if only linear control flow existed.

In other words, stack.mark() becomes effectively a compiler-visible control-flow barrier. Debugging and debugger tooling also become more complicated. After stack.goto(), execution history and the current call chain are no longer the same concept. The debugger will have to present a special representation for this.

There is also the issue of frame identity. After stack.enter(), stack.goto() — what exactly constitutes the same frame?

There is an even more fundamental architectural drawback: this mechanism sharply increases the number of valid control-flow paths. In other words, the complexity of program analysis becomes substantially higher. This matters especially for static analysis, optimization, and the user. The code becomes more powerful, but at the same time loses a significant amount of local predictability. There is also a risk of very bad runtime failure modes. The runtime must limit not only the depth of the ordinary Lua call stack, but also the number of continuations, reentries, branches, and possible transitions between them. Otherwise, the new system can create control-flow cycles that cannot be expressed with ordinary call/return, with no obvious point at which recursion should stop.

Finally, the maintenance cost becomes very high. Even if the first implementation works, every subsequent VM change will require compatibility checks against the new execution model. Almost any change affecting `CallInfo`, stack, upvalues, yield, or continuation may potentially break `stack`.

Therefore, this is not merely a new language feature. It is a permanent obligation for the runtime to support a second, substantially more complex control-flow model. As a result, the main drawback of the proposal is not that `stack` is difficult to implement once. The main drawback is that, after it is implemented, the entire VM is obligated to account for the existence of nonlinear execution control.

This is a very high architectural cost for a capability that most programs will not need. I would especially keep the final point: the main drawback is not "many files have to be changed", but that after this, `CallInfo` ceases to be an internal detail of the ordinary call/return mechanism and becomes part of a nonlinear execution model. This is a very large architectural commitment for LVM. Therefore, `CallInfo` should not become a long-lived object.

## Alternatives

What other designs have been considered? What is the impact of not doing this?

Extend `debug.info()` to include information about all arguments passed to a function; extend `coroutine` with `coroutine.on`, since the method effectively belongs to `coroutine` because it is tied to operations on them. In fact, the entire `stack` library is a collection of uses of the `debug` and `coroutine` libraries and could be implemented even today. The only such mechanism is `error`, because it can be intercepted through `pcall/xpcall` and prevented from killing the coroutine, thereby accomplishing the required task, but at present this is merely a workaround.

But there are other options.

The simplest option is not to change the VM at all and instead pass a special result upward:

```luau
    local result = C()

    if result == SPECIAL_SIGNAL then
        return SPECIAL_SIGNAL
    end

    -- ... and at each return level, pass the special signals upward

    local result = B()

    if result == SPECIAL_SIGNAL then
        return result
    end
```

This fully follows the existing Lua model and requires no runtime changes. The cost is exactly the problem for which stack is being proposed: every intermediate layer must know that a special control-flow protocol exists. With a large execution tree, this quickly turns into noise made of `return`s, status checks, and additional structures. In essence, this is the safest option, but it scales poorly with call depth and the number of independent signals.

Another option is to use the existing error mechanism as a non-local goto: `error(SIGNAL)`

and then:

```luau
local ok, result = pcall(run)
```

This already allows us to jump across an arbitrary number of Lua frames without threading the result through each return. This is why `error` is, in a certain sense, an existing analogue of part of stack.goto().

However, this is semantically incorrect use of error handling. The error becomes not a report of an error, but an ordinary control-flow mechanism. In addition, this approach is tied to unwind semantics, interacts with pcall/xpcall, traceback, and protected boundaries, and does not provide addressable execution state. It cannot naturally express independent `mark`, `hit`, `split`, or `enter` operations as parts of one model. So this is the closest existing workaround, but not a full alternative to the `stack` architecture. There has even been an attempt to implement exactly this mechanism (I have not checked it):

```luau
    --!strict

    const stack = {}

    do
        type userdata_proxy_link = any
        type user_function = (...any) -> ...any
        type layer = number

        type catch_internal_type = {
            read catch_up: number,
        }

        const create_ltype = newproxy
        const transfer_raise = error
        const unexpected_error = error
        const unexpected_assert = assert

        const ltype: string = type(create_ltype())
        const layers = {} :: { [userdata_proxy_link]: number? }
        const links = {} :: { [userdata_proxy_link]: user_function? }
        const f_layers = setmetatable({} :: { [user_function]: { layer? }? }, table.freeze({ __mode = "ks" }))

        const try_catch_selected_function = function(): user_function
            local i = 1

            while true do
                i += 1

                const try_catch_needle_function = debug.info(i, "f")
                const T = type(try_catch_needle_function)

                if T == "function" then
                    if f_layers[try_catch_needle_function] then
                        return try_catch_needle_function
                    end

                    break
                elseif T == "nil" then
                    break
                end
            end

            return unexpected_error("cannot enter, there is no registered function in the stack")
        end

        const catch_xpcall_err = function(err: unknown)
            if type(err) ~= ltype or not links[err] then
                unexpected_error(err)
            end

            const f = unexpected_assert(links[err], "unexpected catch type")
            const f_layer = unexpected_assert(f_layers[f], "unexpected function")
            const expected_layer = layers[err]

            if expected_layer then
                const pos = table.find(f_layer, expected_layer)
                if pos then
                    table.remove(f_layer, pos)
                else
                    stack.raise(expected_layer)
                    layers[err] = nil
                    links[err] = nil
                    return
                end
            else
                table.remove(f_layer)
            end

            if #f_layer == 0 then
                f_layers[f] = nil
            end

            layers[err] = nil
            links[err] = nil
        end

        @checked
        function stack.define<T..., U...>(layer: number?, f: (T...) -> U..., ...: T...): U...
            const f_layer: { number? } = f_layers[f] or {}
            table.insert(f_layer, layer)
            f_layers[f] = f_layer

            return table.unpack({ xpcall(f, catch_xpcall_err, ...) :: any }, 2)
        end

        @checked
        function stack.raise(to_layer: number?)
            const link = create_ltype()
            const stack_function_caller: user_function? = try_catch_selected_function()

            links[link] = stack_function_caller
            layers[link] = to_layer

            transfer_raise(link)
        end
    end

    return stack

```

A continuation can also be represented explicitly:

```luau
    A(function(...)
        B(function(...)
            C(function(...)
                ...
            end)
        end)
    end)

    -- or store continuations as separate objects:

    local continuation = {
        run = function(...) ... end
    }
```

This is already much closer to the model that `stack` actually hides. Instead of changing the VM, the user represents continuations as data. The advantage is that such code requires no special runtime capabilities and can potentially be implemented on top of ordinary Lua. The disadvantage is that continuation becomes part of the application API. The user must manually create, pass, store, and invoke them. As a result, the execution-flow control library turns into an explicit CPS architecture, while `stack` attempts to eliminate precisely the need to keep threading continuation objects through the call tree.

Nonlinear execution can be represented as a finite state machine:

```luau
    local state = "C"

    while state do
        if state == "A" then
            ...
            state = "B"
        elseif state == "B" then
            ...
            state = "C"
        elseif state == "C" then
            ...
            state = "A"
        end
    end
```

This is an option for complex scenarios, AI, parser pipelines, and workflow systems. The main advantage is that control flow becomes explicitly represented as data rather than hidden in the runtime.

But this is already a different programming model. It works well for known states, but is much less suitable for an arbitrary local call tree. `stack` attempts to preserve ordinary function syntax and add non-local control on top of it, whereas a state machine forces the user to model the execution graph in advance.

Some problems can be solved by creating multiple coroutines and coordinating them through coroutine.resume, yield, events, or message queues.

For example:
```luau
    local worker = coroutine.create(function()
        ...
        coroutine.yield(...)
        ...
    end)
```

In this case, execution state is already an object that can be saved and resumed.

This is an important argument in favor of the existing coroutine API: Lua already has the concept of suspended execution state.

However, coroutines do not provide a non-local jump within the same execution tree. `resume` continues a specific coroutine from the point of `yield`, rather than allowing code deep inside C() to transfer control to a previously declared `mark("A")`.

In other words, coroutines solve the problem of preserving execution state, but not the problem of addressable transfer between parts of that state.

We could build an API roughly like this:

```luau
    local continuation = stack.capture()

    continuation:resume(...)
    continuation:cancel()
```

Unlike stack.split(), the continuation here would simply be ordinary userdata/object managed by the user.

This is conceptually much simpler for the VM: instead of making the entire execution stack addressable, the runtime preserves only a specific continuation.

The cost is that the user must manage the lifetime of these objects manually. In addition, it becomes difficult to express operations such as `stack.goto("A")`, because instead of spatial addressing we get an object model like `continuation_A:resume(...)`. That is closer to first-class continuations than to stack traversal.

Another fundamental alternative is to implement full first-class continuations in the language:

```luau
    local continuation = callcc(function(k)
        ...
    end)

    continuation(...)
```

Then the problem is solved at a more general level: continuation becomes a language object and can be saved, passed around, and restarted. In terms of expressiveness, this is even more powerful than some parts of the proposed `stack`. But that is precisely why it is even harder for a Lua-like VM. The same issues appear with frame preservation, local state, upvalues, GC, yield, protected calls, and native boundaries. In addition, call/cc introduces much stronger semantics than most expected `stack` use cases require.

`stack` can effectively be viewed as a limited, structured form of continuations, where access to them is determined by position in the execution tree and existing security boundaries.

Another alternative is to add ordinary VM branch instructions: `JUMP`, `JUMP_IF`, or a specialized runtime opcode for non-local exit. This can be useful for compiler-generated control flow, but is almost useless for the proposed task as a user API.

A normal jump operates inside a single frame/control-flow graph. It does not solve the task:

`C -> B -> A`

when C must transfer execution to state A, which was saved earlier. This still requires continuation/frame management, so simply adding `JUMP` does not solve the problem.

Part of the proposal can indeed be prototyped today through:

```luau
    debug.info()
    debug.traceback()
    coroutine.running()
    coroutine.create()
    coroutine.resume()
    pcall()
    xpcall()
    error()
```

For example, `stack.on()` can be approximately implemented by analyzing the current call chain, while `stack.hit()` and some variants of `mark()` can be backed by a table of contexts. This is useful specifically as a proof of concept: the API and usability can be tested without changing the LVM.

But there is a fundamental limitation: the debug API allows us to observe execution state, but does not allow us to safely restore an arbitrary old execution state. We can determine that a function exists in the call history, but we cannot tell the VM: destroy the current `CallInfo`s and continue this old frame with its local state.

Therefore, the debug API is suitable for prototyping the semantics of `on`/`hit`, but not for implementing a real `goto`/`enter`/`reentered`/`split`.

Instead of changing the VM, we can attempt to transform user code in the compiler. We can transform it into an explicit state machine:

```luau
    while true do
        if pc == 0 then
            ...
        elseif pc == 1 then
            ...
        elseif pc == 2 then
            ...
        end
    end
```

Then `stack.goto("A")` becomes a change to `pc`.

This is potentially an interesting compromise. The runtime remains almost unchanged, while the complicated control model is implemented at the compiler IR level. But there is a serious drawback: it is very difficult to preserve the original semantics of Lua locals, closures, upvalues, `debug.info`, coroutine, yield, and native calls this way. In addition, `stack` assumes the ability to cross ordinary Lua call boundaries, while compiler transformation works only where the compiler can see and transform the corresponding code.

In summary, the options are:

```md
                        VM cost     expressiveness     safety
    return              none        low                high
    error               none        medium             medium
    CPS                 none        high               high
    state machine       none        high               high
    stack               high        high               medium/high
    call/cc             extreme     very high          low
```

Ultimately, there are several classes of alternatives: explicit result propagation, exception-based unwinding, CPS/continuation objects, coroutine orchestration, state machines, compiler transformation, and full first-class continuations/effects. The first options do not require VM changes, but force the user to manually model execution flow. The latter provide the required expressiveness, but run into roughly the same problem for which `stack` exists: the VM must retain and manage execution state independently of the ordinary call/return lifetime.

Therefore, `stack` is not the only possible solution. Its distinction lies in choosing a particular point in the trade-off: execution state remains part of the ordinary Lua execution model, but receives limited addressability and structured non-local transitions. At the same time, `mark`/`hit`/`goto`/`lock`/`on` can be treated as a separate layer of safe control-context transfer, while `split`/`enter`/`reentered` require a much deeper change to the continuation model.

If `stack` is not implemented, Lua retains a simpler and more predictable call/return model, and developers have to use existing mechanisms — return, error, coroutine, callbacks, CPS, or their own state machines. This is substantially cheaper for the VM, but complex nonlinear execution graphs remain the responsibility of user code.

## Examples

Adding this library to the LVM is a very unpleasant task. Even though the operation of all expected code is specified fairly clearly here, I think it is necessary to show examples of why this should exist at all, and what we would have to suffer for it:

Suppose:

```AI
 └─ Planner
     └─ Executor
         └─ Action
             └─ validation
                 └─ resource check
```

The resource check discovers: "we need to completely change the strategy." Under the ordinary model, it has to know how to report this through `Action`, `Executor`, and `Planner`. In other words, the lower layer starts knowing about the protocol of the upper layers. `stack` allows this code to address the execution context directly: `stack.goto("retry_planning")` — this solves the class of problems known as cross-layer control signaling.

Suppose we need separation of control flow from data flow. Ordinary Lua passes data through `return result`, and those data are simultaneously used as a control mechanism: `if result == SPECIAL then ...`

This produces a mixture of data + control flow.

The library allows them to be separated:

data:
    return value

control:
    stack.goto("X")

In other words, a special transition does not have to become a special value, protocol, or userdata.

Suppose we need a programmable execution history.

`stack.on()` solves this class of problem: `stack.on(foo)` asks whether `foo` is present in the ancestry of the current execution context. This is useful when behavior depends not only on the current function, but also on how it was reached.

For example:

```md
    request
    └─ AI
        └─ behavior
        └─ action
```

and code inside `action` can distinguish whether the action was called from AI execution or called independently. This is already a class of execution-context introspection problems.

Suppose we need an early exit from complex algorithms without exception semantics. There is an entire class of algorithms where `error()` is technically suitable but semantically incorrect.

For example:
```md
    parser
    validator
    AI behavior tree
    workflow
    transaction
    resource acquisition
    search algorithm
```

Inside: `D -> discovered a successful result` -> we need to stop a large amount of nested search. We can use `error(SUCCESS)` and catch it through `pcall`, as the current example implementation of `goto/mark` suggests, but then successful control transfer looks like an error.

`stack.goto()` allows us to express: "I did not fail; I changed the execution route." This is a separate class of non-error exceptional control flow.

Suppose we need execution-graph management.

We need to solve not just the call-tree problem, but the execution-graph problem. In the ordinary model: `A -> B -> C -> return`. Therefore, we can end up with:

```md
      A
     / \
    B   B'
    |    |
    C    D
     \  /
      ...
```

In other words, execution state begins to be treated as a programmable graph. This is already a problem in the domain of:

```md
continuations
coroutine scheduling
workflow execution
stateful interpreters
control-flow graphs
```

But here we enter a completely different level of complexity.

These examples are certainly interesting, but what are the unsolvable problems it still has, even though it appears to solve them? It is not a good replacement for:

```md
    return
    coroutine
    async/await
    event system
    state machine
    message passing
    exception handling
```

It is needed when we specifically require:

```md
    deep inside the current execution tree
            ↓
    change the subsequent execution
            ↓
    without manually threading that decision
            ↓
    through every intermediate call
```

`up`/`reentered` add re-entry into existing activations, while `split` adds branching of execution state. These are already not the same problem, but the next levels of the same model.
