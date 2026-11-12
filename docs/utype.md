# User-defined runtime types

## Summary

Allows users to attach a `user-type` to data that can be retrieved through the new `utypeof` function. Essentially an analogue of `__name` from Lua 5.4.

## Motivation
> This concerns metatables and, possibly, to a small extent the classes from a future addition to the language.

The most basic motivation is to be able to check user objects and data with a single function. Not to have a huge pile of validation options via structural comparison, prototype validation, or anything else, but to rely on the field specified in the metatable to determine exactly which resource has arrived.

Currently, in version 0.746, the language supports a special metamethod `__type`, which only works for host-userdata objects through the `typeof()` function. In the future, `class/object` will appear, built directly into the language as native types. Obviously, `type()` will return their runtime type for them, while `typeof()` *will not* return the class name, in order to avoid deceiving the host application.

Classes are a separate topic: according to PR #242, `class.isa(object, class)` is the only validation method. For them, we are forced to trace in the code, that is, `require(path.to.the.required.class)`, which may itself have a dependency on us and simply fail to load in that case. This limits what we could do. Although, as an example, `Roblox` with its `Instance` provides a `ClassName: string` field; this could probably be proposed as another solution to the issue. I am against this alone, because it is imposing an architecture.

## Design
> Remember, `typeof()` is only for the host application. Never use `utypeof()` for RTTI unless you know what you are doing, because `utype` was actually intended as a tag-name.

`utypeof(v)` returns the user tag of a value. For tables/userdata, this is `__utype` from the real metatable. For classes/objects, this is the class's `__utype`, and if it is absent, the runtime type, a.k.a. `typeof()`.

To implement this, we need:
- Add a new metamethod to the metatable: `__utype: string`
- New global function: `utypeof: (v: userdata|table|class|object_of_class) -> string`
+ v - unsupported
    > Returns the result of `typeof()`
+ v - userdata/table
    > Returns the `__utype` field declared in the metatable, or the result of `typeof()`
+ v - class/object_of_class
    > Returns the class name. It is known that classes deliberately moved away from this behavior in order not to create conflicts between modules. This is a design limitation, but the runtime is managed directly by the developer, and therefore the right to determine what is what is not up to `class.isa()` alone.

`utypeof()` reads the real metatable even if `__metatable` is defined. `utypeof()` will not read `__utype` from a metatable that has its own metatable; it is `rawget(rawgetmetatable(t), "__utype")`. Class names are not unique across modules; `utypeof` does not solve this and does not try to; moreover, it defends this behavior, and this is stated in Drawbacks.

### Examples

An example of the internal structure of `utypeof()`:

```luau
    function utypeof (v)
        const T, TT = type(v), typeof(v)
    
        if T == "table" or TT == "userdata" then
            const mt = rawmetatable(v)
            const tag = mt and rawget(mt, "__utype")
            if tag ~= nil then return tag end
        end

        if T == "object" then
            const tag = rawget(classmetatable(v), "__utype")
            if tag ~= nil then return tag end
        end

        if T == "class" then
            return classname(v)
        end

        return TT
    end
```

Then the usual examples:

```luau
    function foo (expect: resource_alpha)
        assert(utypeof(expect) == "alpha", "wrong data received")
    end
```

```luau
    const user_object = newproxy(true)
    const user_object_metatable = getmetatable(user_object)
    const raw = Cryptography.Utilities.CSPRNG.RandomString(64)

    user_object_metatable.__tostring = function () return raw end
    user_object_metatable.__utype = "random_unique_key"

    utypeof(user_object) -- "random_unique_key"
```

```luau
    class Omega 
        public bar: string
    end

    utypeof(Omega {}) == "Omega"

    -- In this case, utypeof() returns the declared class name because classes do not allow
    -- defining concrete values as a default. Obviously, there is nothing to ask the object for,
    -- because it can be represented as a table with special rules for setting and reading data in it.
```

```luau
    const data = {
        __utype = "data",
        __index = function (self, i)
            const mt = getmetatable(self)
            return mt and mt[i] or nil
        end,

        new = function (self)
            return setmetatable({}, getmetatable(self))
        end
    }
    setmetatable(data, data)
    
    utypeof(data:new()) == "data"
```

```luau
    -- In this case, imagine that the module is a dependency of a class. We have no right
    -- to obtain the class as a top-level require or even as a lazy require, because let us assume
    -- that all of the module's functions are used by dependency classes.

    const module = table.freeze {

        get_super_data = function (self, of)
            const UT = utypeof(of)

            if UT == "point" then
                -- ...
            elseif UT == "vector" then
                -- ...
            elseif UT == "matrix" then
                -- ...
            elseif UT ==  "coordinate" then
                -- ...
            end
        end
    }
```


## Drawbacks

`utypeof()` returns what was declared in `__utype`. This only enables verification of a resource based on what the metatable provides. It will not detect data substitution, nor function substitution. It is guaranteed to return the class name. Validation for them and for objects is simply the construct `(type(v) == "object" or type(v) == "class") and utypeof(v) == "needle"`. Checking resources will become easier, especially if they are frozen via `table.freeze()`, as are their metatables.

This RFC is philosophically incompatible with classes. While classes choose to validate themselves via unique data, `utypeof()` reports their raw name deliberately, because this is `user defined data`. A feature that globally affects the two most important elements of the language provides not an RTTI mechanism, but a `user_type`. A tag. `typeof(RobloxInstance)` will return `Instance`, and let `instance:IsA("something")` check which instance it is. `utype` itself is closer to `ClassName` than to the RTTI `:IsA()`. It is not `utypeof()`'s problem that the developer needs to understand which class this is. The developer knows what they want and knows how to work with it. If for some reason they use two different but identical classes, that is their problem, and the standard `class` mechanism is specifically intended for this, to decide who this is in the code. At the same time, the API between these two classes may be completely different, yet they define the same implementation. Is this a problem of `utypeof()`? Probably not. The language protects itself, while the user writes the code and sets it up themselves. It is possible to cut class support and return the same `typeof()` result. BUT then we will not be able to find out the name of this object. In the end, tracing will be through `__tostring()` or analogues, which, by the way, may be used for other purposes, and it may be locale-dependent, change at runtime, depend on state — that is exactly why a separate stable channel is needed. In other words, if `print(vector)` should return xyz, then why does a class not have the right to look like a `struct`? In C++, the difference between them is inverted in terms of access, and only by convention is it accepted that a struct is data and classes are a system, and at the same time Luau is already much closer to C/C++ simply because it is a Lego constructor that does not impose other people's rules.


One could ask "why" if it does not protect the code. Well, there is no automatic check for NaN either, and this is a "feature" of the language. Moreover, `utype` is not about protection, but about identification with a single function in a general, broader sense. Specific specifications and checks are performed by the user based on what they are working with.

## Alternatives

`table_value.type`, `non_host_userdata.ClassName`, `__tostring`, `__metatable`, structural checks.

You can safely reject this mechanism, and we will have to keep checking everything through painstaking certification or hope that the right information comes to us. At least the metatable can be frozen, and one can hope that no one will tamper with exploit code and remove the freeze through magic. Moreover, the metatable is not about objects, but about data properties.

## Prior Art

This innovation does not affect anything other than the metatable and classes. It composes well with them and will work for as long as metatables are supported at all. Classes are the only ones that require special implementation.

* **`type`/`typeof`.** `type` in Luau returns only the base type of a value (`"table"`, `"userdata"`, ...). `typeof` is an extension that, for *host*-userdata, can return the value of the `__type` field from the metatable. At the same time, for custom userdata created via `newproxy`, `typeof` deliberately returns `"userdata"`, "to make sure host-defined types can not be spoofed" ([Luau library](https://luau.org/library)). That is, Luau already has exactly the mechanism being proposed, but it is closed to user code. `__utype`/`utypeof` opens it up for tables and custom userdata.
* **`__name`.** In Lua 5.3/5.4, the `__name` field in a metatable can be used by `tostring` and in error messages; this is also a user-defined string name, but without a separate function to retrieve it ([Lua 5.4 manual, §2.4](https://www.lua.org/manual/5.4/manual.html#2.4)). `__utype` is essentially `__name` with an explicit API and without affecting `tostring`.
* **OOP libraries.** `middleclass`, `30log`, `Penlight`, and similar libraries for Lua usually check membership through the prototype chain and class identity, rather than through a string name. A class name does not guarantee uniqueness, so this approach is considered less reliable.
* **Luau classes.** In [`syntax-classes.md`](syntax-classes.md), `type()` and `typeof()` return `"class"` for a class and `"object"` for an instance, rather than the class name. `utypeof()` deliberately breaks this.
