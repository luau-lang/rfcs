# Runtime user-defined tags (metafield __utype)

## Summary

This RFC adds a `__utype` metafield and a `utypeof()` function: a user-facing runtime identification channel, analogous to `__type`/`typeof()` for host values, but advisory in nature.

## Motivation

For a host application, there is a dedicated metafield `__type` that `typeof()` reads to identify an expected resource. It works only for userdata types created by the host application. It exists because code needs to defend itself against an unsafe user-code environment.

For user code, the situation is somewhat different. It is impossible to guarantee that A == B by a string tag. That problem is pushed onto the developers. But the user still needs to determine what the received object actually is. This is especially visible for tables and userdata created via `newproxy(true)`.

What is `__utype`? An analogue of `__type`, similar to `__name` from Lua 5.5.1 or Lua 5.4. Compared with the latter, it deliberately changes its concept.

By itself, `__utype` solves the `__type` problem of resource identification. Loosely, `Instance.new "Part"` returns `Instance` from `typeof()`. It will work similarly for `DateTime.now()`, `RaycastParams.new()`, `Color3.new()`, `CFrame.new()`, and `Vector3.new()`. Some of them are classes and some are structs. The host application uses `__type` to catch their identifier, although `typeof()` is good for host types because the host itself decides what counts as a type. But when we have a conditional `Quaternion` that is a user implementation, we cannot cheaply verify that it is exactly what we need. There is no such channel that `typeof()` provides for the user.

The original `__name` is used when `error(val)` and `tostring(val)` are called. It is displayed as `[__name]` and `[__name]: 0x0.......` address, respectively. In a program it can be used via `getmetatable(val).__name`. We can already write the same construct separately today. The difference is that a conventionally chosen name is not reserved, and it cannot be obtained through an official path with one function. There is no contract defining the boundaries.

Ultimately, the main motivation is cheap and uniform identification of user data using `__utype`. At the same time, this addition must not replace RTTI, validation, or certification mechanisms.

## Design

To implement it, we only need to add the `__utype` metafield to the metatable and a dedicated global function `utypeof()`. Signatures:

```luau
    type metatable = {
        __utype: string?
    }

    type utypeof = (value: unknown) -> string
```

The behavior of `utypeof()` is unambiguous. The function obtains the real metatable of the object in all cases. It ignores `__metatable`, reads `__utype`, and if it is a string, returns it. Otherwise the function returns `typeof()` of that object. In the type checker, `utypeof()` has no effect on anything. The result is always of type `string`. The value itself is taken at runtime analogously to `rawget(mt, "__utype")`.

For classes and objects, the function will return the class name and nothing else. If classes do not support this and do not intend to, then the result is the same as `typeof()`. Collisions and all cases where this is used as "bad practice" remain the developer's responsibility, because #242 resolves these questions itself. `utypeof()` does not protect; it helps identify.

## Drawbacks

Incorrect use of `utypeof()` leads to collisions between same-named classes from different modules. This must be resolved by the user through the `class` library functions, and `__utype` itself is not intended for such use.

Substitution of `__utype` is allowed. It does not protect against spoofing, errors, or changes. Nor will it protect against collisions. That is not its role.

`utypeof()` does not report the base type. A tag may accidentally be "table" or "userdata". `utypeof(v) == "table"` is not the same as `type(v) == "table"`. This is intentional. For the base type — `type()`, for host — `typeof()`, for the tag — `utypeof()`. Three functions, easy to confuse.

Mutation at runtime. `__utype` can be changed at any moment if the metatable is not frozen.

Host application developers must be explicitly warned to use only `typeof()`, since `__type` is the only guaranteed-protected one.

## Alternatives

* The metatable is available: `getmetatable(val).name`
* Deliberately give `__tostring()` the identification role: `__tostring(val)`
* Deliberately allocate a whole string field `type` in objects
* Implement `utypeof()` on top of `getmetatable(val).name` by convention (use `_G.utypeof()` or `require(global_module).utypeof()`).
* Stay with structural checking at the same weak efficiency (`foreach`)
    > Or separate values through architectural means. This does not always work.

## Prior Art

* Luau already implements `typeof()`. This RFC proposes implementing a channel for language users that represents an insecure but nevertheless public and simple way to identify objects.
* `__name` is a diagnostic field defined in Lua 5.4 (it is absent in Luau). It is read by `tostring()` and `error()` when `__tostring()` is not set, and affects the displayed name of the object and the text of error messages. In other words, `__name` is about display and diagnostics. If an object should look different in `print()` or `error()`, it uses `__name`. `__utype` is about identification. It is read only by the new `utypeof()` function and does not affect either `tostring()` or error messages. It exists precisely so that code can cheaply answer the question "what is this value?" without depending on how the value looks when printed.
