# Function default arguments

## Summary

Add default values to arguments in function definitions, used when a parameter is unspecified or `nil`.

## Motivation

Frequently when writing code, having a default value for an unspecified argument is desired. This can be found in a range of languages, such as C++ and Python. Luau has no first-party support for this, instead filling omitted arguments as `nil` allowing programmers to begin their functions with a series of statements such as:

```lua
arg = arg or default
```
or
```lua
if arg == nil then arg = default end
```
or
```lua
arg = if arg == nil then default else arg
```

While effective this produces a degree of noise at the start of functions. Notably also the `or` short-circuit approach will coalesce all falsey values to the default value, rather than just `nil` values.

The luau type system also does not currently narrow these types correctly. For example,
```lua
function demo_type(arg: number?)
    if arg == nil then arg = 0 end
    local x = arg
    return x
end
```
will result in `x` having an inferred type of `number?` and the function having a return type of `number?`. Resolving this involves creating a new variable name for the parameter post-defaulting rather than shadowing the name. This RFC explicitly hides the optional nature of arguments from the function body.

Further, this defaulting behavior is hidden from tooling that may want to inspect function signatures, such as tooltips in IDEs. A handful of examples from the Roblox standard library that could benefit have been included here:

```lua
function color3.new(x = 0, y = 0, z = 0)
function Vector3.new(x = 0, y = 0, z = 0)
function CFrame.fromEulerAngles(rx: number, ry: number, rz: number, order = Enum.RotationOrder.XYZ)
function DataStoreService:GetDataStore(name: string, scope = "global", options: DataStoreOptions?)
function DataStoreService:ListDataStoresAsync(prefix = "", pageSize = 0, cursor = "")
```

A slightly more complicated Roblox example could call API functions in the default value, for example:

```lua
local Players = game:GetService("Players")
function give_players_item(item, players = Players:GetPlayers())
```

## Design

This proposal at its core suggests the following modification to the luau grammar:

```diff
- parlist = bindinglist [',' '...' [':' GenericTypePack | Type]]
+ parlist = bindinglistwithdefault [',' '...' [':' GenericTypePack | Type]]

(* snip *)

+ bindinglistwithdefault = binding ['=' exp] [',' bindinglistwithdefault]
```

This allows any function argument to be provided with any expression as its default argument. To explain the proposed semantics, the following demonstration function will be used

```lua
local ARG2_CONSTANT = 2
function demo(
    arg1: number,
    arg2: number = ARG2_CONSTANT,
    arg3 = {x=0, y=0},
    arg4 = print("arg4 default"),
    arg5: number
)
    print(arg1, arg2, arg3, arg4, arg5)
end
```

### Semantics: Default expressions are evaluated at call-time
In python, a common foot-gun is defining a function as `def example(arg=[])` which will use a singleton list instance across all calls of the function. This RFC suggests expressions are evaluated strictly when the function is called, matching the semantics of the `if`-based existing methods.

This is shown both by `arg3` creating a new table every call, and `arg4` outputting `"arg4 default"` to the console.

### Semantics: Default expressions are only evaluated when necessary
If a value is provided for an argument, the default expression is not evaluated.

Proving a value for `arg4` will silence the `"arg4 default"` output in the console.

### Semantics: Passing `nil` is equivalent to an unspecified argument
Following similar semantics to assigning a table entry to `nil`, passing `nil` as a function argument causes the default value to be used.

This can be seen by passing `nil` as any of `arg2`, `arg3` or `arg4`.

As a result of this...

### Semantics: Non-default arguments are allowed to follow default arguments
Unlike in other languages, there is no strict requirement that all arguments after the first default argument must also be default arguments. If we wished to call our demonstration function providing a value for only `arg1` and `arg5` we could call it as

```lua
demo(1, nil, nil, nil, 5)
```

This further allows for any arbitrary mixture of provided and non-provided arguments.

### Semantics: Default value expressions cannot access function arguments
All required default arguments are evaluated before function arguments are bound to the local scope.

For example

```lua
function foo(a, b = a)
    print(a, b)
end
foo(1)
```

will output `1 nil` rather than the potentially expected `1 1`. When evaluating the default value for `b`, the parameter `a` has not yet entered scope and so `= a` attempts to reference `a` from higher scopes. This behavior is analogous to the statement `local a, b = 1, a`.

It is noted that this may not be desired behavior in all cases; one can imagine a function such as the below where it may be beneficial to be able to access the arguments during defaulting:

```lua
function table.remove(t: Array, pos: number = #t)
```

This RFC does not currently consider this, though future works may re-investigate this behavior.

### Type Semantics: Type annotations take precedence over default types when performing inference
When inferring types, an explicit annotation has precedence over the inferred type of the default argument. For example, in the following function the error should be that a `Type 'string' could not be converted into 'number'`.

```lua
function demo_2(a: number = "demo") end
```

### Type Semantics: Inferred types from default values take precedence over inferred types from the function body
If a type has been inferred from the default value, a conflicting type within the function body will cause an error. The following code is invalid

```lua
function demo_3(a = "demo")
    a = 1  -- Invalid
end
```

### Type Semantics: Inferred types from default values are not sealed
When a table type is inferred from a default value, it is not sealed. If the function body makes modifications, these are unified with the inferred type.

In the following example, `point` has a final type of `{x:number,y:number,z:number}`.

```lua
function demo_unsealed(point = {x = 0, y = 0})
    point.z = 0  -- Ok
end
```

If sealing is desired, the programmer may be explicit about this using either an annotation or a cast. The former takes precedence over the inferred type entirely, and the latter ensures the inferred type is sealed. Rather than a unique semantic this is a consequence of previously defined semantics.

```lua
type PointType = {
    x: number,
    y: number,
}
function demo_sealed_1(point: PointType = {x = 0, y = 0})
    point.z = 0  -- Invalid
end
function demo_sealed_2(point = {x = 0, y = 0} :: PointType)
    point.z = 0  -- Invalid
end
```

### Language implementation
Within the AST, it would likely be simpler to add a new `AstArray<AstExpr*> argsDefaults` to `AstExprFunction` alongside `args`, rather than modify `args` to be an `std::pair` due to the existing widespread usage of `args`.

Rather than implement this as a feature of the `CALL` instruction within Luau's VM it is instead suggested by this RFC to implement this as part of the compiler. This both increases compatibility (as the VM remains unchanged) and makes it easier to allow *any* expression to be used.

Each defaulted argument can be implemented using a single inverted `JUMPXEQKNIL` instruction followed by a `compileExpr` call. This costs no additional registers beyond what would be already necessary for evaluation of the expression.

## Drawbacks

Programmers could use this to write some particularly hard to follow code. While `arg = default_factory()` is an intended use case for complex expressions within arguments, it would be viable to have immense numbers of side-effects within a single function call. Likewise, we may see code such as
```lua
function demo(
    arg1 = (function()
        print("Inside nested function")
    end)
)
    arg1()
end
```

This could also be considered a feature though for default callback arguments.

Furthermore, there is no strict necessity for this feature. It is at its core syntactic sugar for something programmers have already been doing for a long time. This may be instead considered syntactic *noise*.

## Alternatives

For programmers, the three examples outlined in Motivation are already wildly used as viable alternatives with semantics mostly matching that of the proposal in this RFC.

Addition of proper type narrowing to the language would resolve the issue where the traditional patterns for default arguments fail to narrow the type correctly. The author of this RFC strongly advocates for proper type narrowing regardless of this RFC's status.

For tooling that wishes to identify default arguments, it would be viable to inspect the AST of a function to check for the three existing patterns in widespread use. The author is unaware of any tooling that does this.
