# `coroutine.noyield()` function


## Summary

A new standard library function, `coroutine.noyield(fn, ...)`, which runs a function without allowing it to yield.


## Motivation

Currently, there is no non-hacky, standard, stable, and documented way to make Luau deliberately enforce that a function does not yield, though this behavior already exists for calls made through the C boundary. Formerly in Lua, this was achievable through `pcall`, as it did not allow the protected function to yield (with the drawback of errors not being propagated).

Sometimes it is useful to enforce that a function does not yield, and instead runs "atomically", mostly to avoid race conditions or guarantee an instant result. Below are some examples, current possible solutions, and their drawbacks.


### Examples of problematic yielding


#### Cleanup callbacks

We have this function:

```luau
local function bindToTag(tag: string, callback: (object: Instance) -> (() -> ()))
```

The function calls `callback` whenever an instance with the given tag is added, and expects `callback` to return a cleanup function that is called when the instance is removed. If `callback` yields, it may cause the instance to be removed before the cleanup function is returned, causing race conditions and a memory leak if the cleanup function is stored.


#### Race conditions

We have these classes:

```luau
type Event<T> = {} -- skipped for brevity

type State<T> = {
	read value: T,
	changed: Event<T>,
}

type SettableState<T> = State<T> & {
	setValue: (self: SettableState<T>, newValue: T) -> ()
}

type MappedState<S, T> = State<T> & {
	source: State<S>,
	mapper: (sourceValue: S) -> T
}
```

An `Event` can be listened to by providing a callback, and fired given a value. `State` is an abstract class, derived classed are allowed to write to `value` and fire `changed`. `SettableState` allows setting the value through a method, while `MappedState` derives its value from another `State` (`source`) through a user-provided `mapper` function.

Problems arise if the `mapper` function yields: the value of `MappedState` will remain stale until `mapper` is resumed and returns. This is already undesired; to add insult to injury, If `changed` fires while a call to `mapper` is yielding, `mapper` will fire again, potentially returning before the first call. When the first call returns, it will overwrite the value with a stale one.

Additionally, when iterating `Event`'s callbacks, if firing one of them yields, the call of the remaining callbacks will be delayed. The `Event` could also fire again while yielding, causing the remaining callbacks to receive the event out of order.


#### Uncontrolled context switches

We have this function:

```luau
local function inspect<T>(value: T, analyzers: { (value: T) -> () })
```

`value` might be mutable (such as, a `table`), and `analyzers` is a list of functions which run an analysis on `value`, and are not allowed to mutate it. If one of the analyzers yields and defers resumption to external code (such as the task scheduler in Roblox), other code from unrelated contexts where mutating `value` is allowed might run, causing the remaining analyzers to run on a later version of `value`.


#### Time-critical code

We're using a game engine that provides the following API:

```luau
type Engine = {
	beforeFrame: () -> ()?
}
```

`beforeFrame` is called before each frame is rendered, and is expected to run quickly. If the provided function yields, the frame will be delayed until it resumes, causing stutter.

Because the current API, which is just a single callback, is not very flexible, we wrap it in our earlier `Event` type. We want to ensure thet the callbacks do not yield, to avoid stutter.


### Solutions:


#### Document that a function must not yield.

Problems:
- [Silent-error prone](#silent-error-prone)


#### Assert that all user-provided functions may yield, and account for it.

We account for yielding in the code that calls user-provided functions, keeping track of race conditions and creating coroutines where necessary, even if undesireable. This still does not fix [uncontrolled context switches](#uncontrolled-context-switches).

Problems:
- [Silent-error prone](#silent-error-prone)
- [Performance cost, complexity, and uncertainty](#performance-cost-complexity-and-uncertainty)
- [Creation of a new coroutine](#creation-of-a-new-coroutine)
- [Error handling](#error-handling)
- [Coroutine ref](#coroutine-ref)
- [Variadic arguments and returns packing](#variadic-arguments-and-returns-packing)
- [Helper functions](#helper-functions)


#### Create a new coroutine for the function, so yields will not propagate to the caller:

<details>
<summary>with `coroutine.create` and `coroutine.resume`:</summary>

```luau
local function noYieldHelper<R...>(coro: thread, ok: boolean, ...: R...): R...
	if not ok then
		error(debug.traceback(coro, tostring(...)))
	end

	if coroutine.status(coro) ~= "dead" then
		local trace = debug.traceback(co, "Function yielded")
		coroutine.close(coro)
		error(trace)
	end

	return ...
end

local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local coro = coroutine.create(fn)

	return noYieldHelper(coro, coroutine.resume(coro, ...))
end
```

</details>

<details>
<summary>with `coroutine.wrap`:</summary>

```luau
local function noYieldHelper<R...>(coroRef: {thread}, ...: R...): R...
	if coroutine.status(coroRef[1]) ~= "dead" then
		local trace = debug.traceback(coroRef[1], "Function yielded")
		coroutine.close(coroRef[1])
		error(trace)
	end

	return ...
end

local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local coroRef: {thread} = {}

	return noYieldHelper(
		coroRef,
		coroutine.wrap(
			function(...)
				coroRef[1] = coroutine.running()
				return fn(...)
			end
		)(...)
	)
end
```

</details>

<details>
<summary>In Roblox specifically, with `task.spawn`:</summary>

```luau
local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local result: { [number]: any, n: number }? = nil

	local coro = task.spawn(function(...)
		result = table.pack(fn(...))
	end, ...)

	if result then
		return unpack(result, 1, result.n)
	else
		local trace = debug.traceback(coro, "Function yielded or errored")
		coroutine.close(coro)
		error(trace)
	end
end
```

</details>

Problems:
- [Silent-error prone](#silent-error-prone)
- [Creation of a new coroutine](#creation-of-a-new-coroutine)
- [Error handling](#error-handling)
- [Coroutine ref](#coroutine-ref)
- [Variadic arguments and returns packing](#variadic-arguments-and-returns-packing)
- [Helper functions](#helper-functions)


#### Use a function that is documented to call a function without yielding.

Non-exhaustive list:

<details>
<summary>`xpcall` error handler:</summary>

```luau
local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local args = table.pack(...)

	local _, result = xpcall(
		error,
		function()
			return table.pack(
				xpcall(
					fn,
					function(err)
						return debug.traceback(err)
					end,
					unpack(args, 1, args.n)
				)
			)
		end
	)

	if not result[1] then
		error(result[2])
	else
		return unpack(result, 2, result.n)
	end
end
```

</details>

<details>
<summary>`table.foreach`/`table.foreachi`:</summary>

This is the most performant solution that correctly enforces non-yielding calls as per [benchmarks](#benchmarks).

```luau
local singleElementTable = {true}

local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local args = table.pack(...)

	local result = table.foreachi(singleElementTable, function()
		return table.pack(fn(unpack(args, 1, args.n)))
	end)

	return unpack(result, 1, result.n)
end
```

</details>

<details>
<summary>`table.sort`:</summary>

```luau
local twoElementTable = {true, false}

local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local args = table.pack(...)
	local alreadyCalled = false
	local result: { [number]: any, n: number }

	table.sort(twoElementTable, function(a)
		if not alreadyCalled then
			alreadyCalled = true
			result = table.pack(fn(unpack(args, 1, args.n)))
		end

		return a
	end)

	return unpack(result, 1, result.n)
end
```

</details>

Problems:
- [Hacky implementation detail](#hacky-implementation-detail)
- [Cryptic error message](#cryptic-error-message)
- [Deprecated functions](#deprecated-functions)
- [Misuse of features](#misuse-of-features)


#### Use a metamethod that is not allowed yielding:

<details>
<summary>with `__index` on a table:</summary>

```luau
-- If you want to be nitpicky about performance, replace with newproxy(true)
local noYieldTable = setmetatable({}, {
	__index = function(_, info)
		local fn, args = info[1], info[2]
		return table.pack(fn(unpack(args, 1, args.n)))
	end
})

local function noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local result = noYieldTable[{fn, table.pack(...)}]
	return unpack(result, 1, result.n)
end
```

</details>

Problems:
- [Hacky implementation detail](#hacky-implementation-detail)
- [Cryptic error message](#cryptic-error-message)
- [Misuse of features](#misuse-of-features)


### Implementing a custom yielding library

<details>
<summary>For example:</summary>

```luau
local NoYield = {}

local noYieldCoroutines = setmetatable({}, { __mode = "k" })

function noYield.isyieldable(): boolean
	return coroutine.isyieldable() and not noYieldCoroutines[coroutine.running()]
end

function noYield.yield(...: any): ...any
	if not noYieldCoroutines[coroutine.running()] then
		return coroutine.yield(...)
	else
		error("attempt to yield across noyield boundary")
	end
end

local function noYieldXpcallHelper<A...>(coro: thread, success: boolean, ...: A...): A...
	noYieldCoroutines[coro] = nil

	if not success then
		error(...)
	end

	return ...
end

function NoYield.noYield<A..., R...>(fn: (A...) -> R..., ...: A...): R...
	local coro = coroutine.running()

	if noYieldCoroutines[coro] then
		return fn(...)
	end

	noYieldCoroutines[coro] = true

	return noYieldXpcallHelper(coro, xpcall(
		fn,
		function(err)
			return debug.traceback(err)
		end,
		...
	))
end

return NoYield
```

</details>

Problems:
- [Silent-error prone](#silent-error-prone)
- [Performance cost, complexity, and uncertainty](#performance-cost-complexity-and-uncertainty)
- [Error handling](#error-handling)
- [Helper functions](#helper-functions)
- [Dependency and portability](#dependency-and-portability)


#### Patch the coroutine global through `_G`, `shared`, or `setfenv`

Implementation skipped for brevity.

Problems:
- [Global patching](#global-patching)


### Problems


#### Silent-error prone

Documenting that a function must not yield is not enforceable, and may be broken by user code. A called function may also not expect to have its thread closed unexpectedly, or continue after an error condition. This can cause silent bugs, that are hard to track down, as detailed in the [examples of problematic yielding](#examples-of-problematic-yielding).


#### Performance cost, complexity, and uncertainty

Treating all user-provided functions as yieldable has a performance cost, and complicates code that would otherwise be simple, requiring accounting for race conditions and asynchrony where it is already an undesired behavior.

If the resumption of a yield is deferred to an external scheduler, such as Roblox's task scheduler, code running between yields is uncontrolled, and may mutate state in unexpected ways that can not be controlled. This also adds uncertainty to when the thread will resume. These cases can't be handled, and are [silent-error prone](#silent-error-prone).

There's performance-critical situations where we need to run many functions that must not yield, and the overhead of creating a coroutine for each call is considerable.

Coroutine caching can mitigate the performance cost, but adds complexity, limits the use of coroutines, and still has a performance cost.


#### Creation of a new coroutine

For clarity, let us define:
- `A`: Called function that must not yield.
- `B`: Caller function that calls A, and enforces that A does not yield by running it in a new coroutine and checking its status.

`A` has its own coroutine and stack, and is technically allowed to yield. This presents several problems. 

1. Creating a coroutine has a performance cost, both in time and memory. This is especially relevant if many functions are called and expected not to yield, or in performance-critical code.

2. `A` does not preserve the stack of `B`, despite wanted behavior being more akin to that of a direct call. This obscures traces and requires more stack space.

3. `A` has no knowledge that it is not allowed to yield, as `coroutine.isyieldable` will return `true`, and the call to `coroutine.yield` will not error itself. Handling the mistake of yielding from within the function through `coroutine.isyieldable`/`(x)pcall(coroutine.yield)` is impossible (depending on the implementation, the thread may be closed unexpectedly, or continue running unnecesarily after an error condition. Both may also cause [silent or obscure errors](#silent-error-prone)).

4. Yielding inside `A` will cause an error in `B`, whereas an error in `A`'s call to `coroutine.yield` would be preferred.

5. The values of `coroutine.running()` in `A` and `B` are different, despite wanted behavior being more akin to that of a direct call. For their purpose, they're the same thread, and it'd be desireable for them to act as such.

6. The value of `B`'s `coroutine.status()` is `"normal"` as it is resuming `A`, despite wanted behavior being more akin to that of a direct call.

7. Nested calls to `noYield` will create multiple coroutines unnecessarily, with the same drawbacks as above multiplied.

Problems 3-7 can be mitigated with the techniques from solutions ["implementing a custom yielding library"](#implementing-a-custom-yielding-library)" or ["patch the coroutine global"](#patch-the-coroutine-global-through-_g-shared-or-setfenv), but they add complexity and have their own drawbacks.


#### Error handling

Because the call is protected, error rethrowing logic is necessary. Accompanying the rethrown error with a stack adds complexity and requires transforming the error, losing its identity. In the specific case of `coroutine.wrap`, rethrowing is automatic, but the called function's stack is lost. With `task.spawn`, the error is lost and logged to the console, with the stack of the called function only.


#### Coroutine ref

Since `coroutine.wrap` does not return the created coroutine, a helper function is needed to capture it and check its state, with the added complexity.


#### Variadic arguments and returns packing

In order to handle variadic arguments and return values, we need to use `table.pack`, `unpack`, tables, and upvalues (if applicable), which add complexity and have a performance cost.


#### Helper functions

Extra helper functions add complexity to the code and the stack.


#### Hacky implementation detail

This is a hacky solution, relying on implementation details that may change or are undocumented, is not self-explanatory, and is complex without knowledge of the C-call boundary, and may not be portable to other Lua(u) versions.

For example, the `xpcall` handler in Lua 5.2 is allowed to yield. While right now Luau explicitly disallows yielding in these hacks due to implementation details, an official guarantee that they will continue to work restricts future changes to actual, intended features.


#### Cryptic error message

The error thrown when yielding is `"attempt to yield across metamethod/C-call boundary"`, which is cryptic since it describes the hack used to implement the feature.


#### Deprecated functions

`table.foreach`/`table.foreachi` are deprecated and not recommended. They may be subject to removal, or changes that break this hack.


#### Misuse of features

This is a confusing misuse of a function or feature that relies on an implementation detail. Mimicking the behavior of the actual use adds complexity, especially for `table.sort`.


#### Dependency and portability

Implementing a custom yielding library adds complexity and is is impractical. Furthermore, it requires all code to use the custom library and cooperation between them. It is not compatible with third-party code that use the standard coroutine library, or even another library for yielding.

This can be mitigated by using another hacky solution... With the problems they have themselves - you might as well just do them instead.


#### Global patching

The use of `_G`, `shared`, or `setfenv` is advised against for reasons that will not be stated here. It also introduces the issue of needing all calling functions to use the patched environment.


## Design

A new standard library function, `coroutine.noyield<A..., R...>(fn: (A...) -> R..., ...: A...): R...`, which marks the current coroutine as not allowed to yield by the user, calls `fn` with the provided arguments, unmarks the coroutine, and returns all the results of `fn`.

No new coroutine or stack are created. The function is called "directly", with the following expected behavior:

- Calls to `coroutine.isyieldable` while `fn` is running return `false`.

	<details>
	<summary>Example:</summary>

	```luau
	print(coroutine.isyieldable()) -- true

	coroutine.noyield(function()
		print(coroutine.isyieldable()) -- false
	end)
	```

	</details>

- Calls to `coroutine.yield` while `fn` is running error. If the coroutine was already not yieldable due to a C call, the normal `"attempt to yield across C-call boundary"` error is thrown. If the coroutine was yieldable, the error thrown is `"attempt to yield across noyield boundary"`.

	<details>
	<summary>Examples:</summary>

	```luau
	coroutine.noyield(function()
		print(coroutine.isyieldable()) -- false
		coroutine.yield() -- error: attempt to yield across noyield boundary
	end)

	print("Not reached")
	```

	```luau
	local t = setmetatable({}, {
		__index = function()
			coroutine.noyield(function()
				coroutine.yield()
			end)
		end
	})

	coroutine.noyield(function()
		print(pcall(coroutine.yield)) -- false, attempt to yield across noyield boundary
		print(t[true]) -- error: attempt to yield across C-call boundary
	end)

	print("Not reached")
	```

	</details>

- `fn` is run in the same coroutine used to call `coroutine.noyield`.

	<details>
	<summary>Example:</summary>

	```luau
	local coro = coroutine.running()

	coroutine.noyield(function()
		print(coroutine.running() == coro) -- true
	end)
	```

	</details>

- The value of `coroutine.status()` is `"running"`, both for the the coroutine calling `coroutine.noyield` and inside `fn`, as they are the same thread.

	<details>
	<summary>Example:</summary>

	```luau
	local coro = coroutine.running()
	print(coroutine.status(coro)) -- running

	coroutine.noyield(function()
		print(coroutine.running() == coro) -- true
		print(coroutine.status(coro)) -- running
		print(coroutine.status(coroutine.running())) -- running
	end)
	```

	</details>

- Errors thrown inside `fn` propagate as normal.

	<details>
	<summary>Example:</summary>

	```luau
	local function willError()
		coroutine.noyield(function()
			error({})
		end)
	end

	willError()
	-- error: table: 0x...
	-- trace: stdin -> willError -> coroutine.noyield -> (anonymous) -> error
	```

	</details>

- The stack inside `fn` is preserved, and is identical to a direct call, other than the addition of the call to `coroutine.noyield`.

	<details>
	<summary>Example:</summary>

	```luau
	local function printStack()
		print(debug.traceback())
	end

	printStack()
	-- trace: stdin -> printStack

	coroutine.noyield(printStack)
	-- trace: stdin -> coroutine.noyield -> printStack
	```

	</details>

- `fn` is still allowed to create and resume other coroutines, which may themselves yield, as normal.

	<details>
	<summary>Example:</summary>

	```luau
	coroutine.noyield(function()
		print(coroutine.isyieldable()) -- false

		local coro = coroutine.create(function()
			print(coroutine.isyieldable()) -- true
			return coroutine.yield()
		end)

		coroutine.resume(coro)
		print(coroutine.resume(coro, "OK")) -- true, OK
	end)
	```

	</details>

- All arguments and return values are passed as normal, including variadic arguments and returns.

	<details>
	<summary>Example:</summary>

	```luau
	print(coroutine.noyield(function(a, ...)
		return a, ...
	end, 1, 2, 3, nil, 5, nil)) -- 1, 2, 3, nil, 5, nil
	```

	</details>

- Nested `coroutine.noyield` calls are supported, and, other than the added `coroutine.noyield` call to the stack, are functionally identical to doing `fn(...)`.

	<details>
	<summary>Example:</summary>

	```luau
	print(coroutine.isyieldable()) -- true

	coroutine.noyield(function()
		print(coroutine.isyieldable()) -- false

		coroutine.noyield(function()
			print(coroutine.isyieldable()) -- false
		end)

		print(coroutine.isyieldable()) -- false
	end)

	print(coroutine.isyieldable()) -- true
	```

	</details>


## Drawbacks

1. This is an addition to the standard library, not present in standard Lua.

2. Coroutines now have an additional state (yieldable or not by user code), which adds complexity to the implementation. This state must be preserved, so the appropiate error message is thrown when a yield occurs. It is necessary to track this state and correctly update it when `fn` both returns or errors.

3. Coroutines and yielding are an important feature of Luau, and this feature may encourage users to overuse or misuse no yield enforcement, creating less flexible code.

4. Handling the case of a call yielding is already possible, the most idiomatic way being [creating a new coroutine](#create-a-new-coroutine-for-the-function-so-yields-will-not-propagate-to-the-caller). Even if it has its drawbacks, it is a known and established pattern.


## Alternatives

1. Though the ability to enforce non-yielding calls is useful, directly exposing it it is not a strictly necessary addition - the existing "hacks" are sufficient and performant, though inelegant. If Luau has no plans to change them, they can be documented as the recommended way to enforce non-yielding calls.

2. The implementation detail of throwing a different error message for debugging clarity when yielding across a no-yield boundary may be unnecessary, especially if the traceback is preserved, and the existing `"attempt to yield across C-call boundary"` can be sufficient. In this case, `coroutine.noyield` would no longer need to track if yielding is allowed by the user, making it effectively a sandboxed proxy to the `lua_call` C function.

3. It may be useful to instead add a `coroutine.markyieldable(coro: thread, yieldable: boolean)` function that manually marks a coroutine as yieldable or not, thus dynamically allowing a thread, or other created threads to yield only during specific situations. `coroutine.noyield()` however is a simpler and more user-friendly abstraction, as a system to manually mark coroutines would be more complex to use and error-prone.

4. It'd also be possible to add a `coroutine.createNoYield(fn: () -> ()): thread` function that creates a coroutine that is not allowed to yield once resumed. This would still create a new coroutine, with its own stack.

5. Coroutines in general are not type-safe. An alternative would be to add type system support for yielding and resuming types. This would allow marking functions as not allowed to yield at the type level, and have the type checker enforce it. Coroutines are rather dynamic in nature, and this would be a complex addition to the type system, both for Luau's development and users, with limited benefits. Regardless, `coroutine.noyield()` is a runtime, not compile-time feature, and would still be useful.


## Benchmarks

<details>
<summary>Benchmark code:</summary>

```luau
--!strict
--!optimize 2

local benchmarks = {
	["Direct call"] = function()
		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			return fn(...)
		end
	end,
	
	["pcall"] = function()
		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			return select(2, pcall(fn, ...))
		end
	end,
	
	["coroutine create"] = function()
		local function noYieldHelper<R...>(coro: thread, ok: boolean, ...: R...): R...
			if not ok then
				error(debug.traceback(coro, tostring(...)))
			end

			if coroutine.status(coro) ~= "dead" then
				local trace = debug.traceback(co, "Function yielded")
				coroutine.close(coro)
				error(trace)
			end

			return ...
		end

		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local coro = coroutine.create(fn)

			return noYieldHelper(coro, coroutine.resume(coro, ...))
		end
	end,
	
	["coroutine wrap"] = function()
		local function noYieldHelper<R...>(coroRef: {thread}, ...: R...): R...
			if coroutine.status(coroRef[1]) ~= "dead" then
				local trace = debug.traceback(coroRef[1], "Function yielded")
				coroutine.close(coroRef[1])
				error(trace)
			end

			return ...
		end

		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local coroRef: {thread} = {}

			return noYieldHelper(
				coroRef,
				coroutine.wrap(
					function(...)
						coroRef[1] = coroutine.running()
						return fn(...)
					end
				)(...)
			)
		end
	end,
	
	["task spawn"] = function()
		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local result: { [number]: any, n: number }? = nil

			local coro = task.spawn(function(...)
				result = table.pack(fn(...))
			end, ...)

			if result then
				return unpack(result, 1, result.n)
			else
				local trace = debug.traceback(coro, "Function yielded or errored")
				coroutine.close(coro)
				error(trace)
			end
		end
	end,
	
	["xpcall handler"] = function()
		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local args = table.pack(...)

			local _, result = xpcall(
				error,
				function()
					return table.pack(
						xpcall(
							fn,
							function(err)
								return debug.traceback(err)
							end,
							unpack(args, 1, args.n)
						)
					)
				end
			)

			if not result[1] then
				error(result[2])
			else
				return unpack(result, 2, result.n)
			end
		end
	end,
	
	["foreach"] = function()
		local singleElementTable = {true}

		return function<A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local args = table.pack(...)

			local result = table.foreach(singleElementTable, function()
				return table.pack(fn(unpack(args, 1, args.n)))
			end)

			return unpack(result, 1, result.n)
		end
	end,
	
	["foreachi"] = function()
		local singleElementTable = {true}

		return function<A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local args = table.pack(...)

			local result = table.foreachi(singleElementTable, function()
				return table.pack(fn(unpack(args, 1, args.n)))
			end)

			return unpack(result, 1, result.n)
		end
	end,
	
	["sort"] = function()
		local twoElementTable = {true, false}

		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local args = table.pack(...)
			local alreadyCalled = false
			local result: { [number]: any, n: number }

			table.sort(twoElementTable, function(a)
				if not alreadyCalled then
					alreadyCalled = true
					result = table.pack(fn(unpack(args, 1, args.n)))
				end

				return a
			end)

			return unpack(result, 1, result.n)
		end
	end,
	
	["userdata index"] = function()
		local noYieldTable = newproxy(true)
		getmetatable(noYieldTable).__index = function(_, info)
			local fn, args = info[1], info[2]
			return table.pack(fn(unpack(args, 1, args.n)))
		end

		return function <A..., R...>(fn: (A...) -> R..., ...: A...): R...
			local result = noYieldTable[{fn, table.pack(...)}]
			return unpack(result, 1, result.n)
		end
	end,
}

local function createVarReturn()
	return 1, 2, 3, nil, 5, nil
end

local function runAssertions(...)
	assert(select(1, ...) == 1)
	assert(select(2, ...) == 2)
	assert(select(3, ...) == 3)
	assert(select(4, ...) == nil)
	assert(select(5, ...) == 5)
	assert(select(6, ...) == nil)
	assert(select("#", ...) == 6)
end

local function myFunc(...)
	runAssertions(...)
	return ...
end

for k, v in benchmarks do
	print(k)
	local implementation = v()
	
	local started = os.clock()
	
	for i = 1, 100_000 do
		runAssertions(implementation(myFunc, createVarReturn()))
	end
	
	local ended = os.clock()
	
	print(ended - started)
end
```

</details>

Run on Roblox Studio 0.690.0.6900721 (64bit). Microsoft Windows 11 Pro, version 24H2, OS build 26100.4946. Intel Core Ultra 7 155H, 1400 Mhz, 16 logical processors. 32 GB RAM.

| Position | Benchmark        | Time (s)             |
|----------|------------------|----------------------|
| 1        | Direct call      | 0.020197099991491996 |
| 2        | pcall            | 0.023403800005326048 |
| 3        | foreachi         | 0.04204840000602417  |
| 4        | userdata index   | 0.042342799992184155 |
| 5        | foreach          | 0.0428230999968946   |
| 6        | sort             | 0.04651909999665804  |
| 7        | coroutine create | 0.055262299996684305 |
| 8        | coroutine wrap   | 0.0674972999986494   |
| 9        | task spawn       | 0.11786820000270382  |
| 10       | xpcall handler   | 0.3082933999976376   |
