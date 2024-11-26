# Support Generics in Metamethods

## Summary

Currently, functions executed within a metatable (for example a defined `__index` metamethod); do not support generic types and they are considered unrecognised if you attempt to use them.

## Motivation

Currently, more complex data structures which rely on metatables cannot be typechecked against. For example take the following data-schema:
```luau
local foo = {
    ["bar"] = {
        ["Metadata"] = {...},
        ["Value"] = "ExampleString"
    },
    ["taz"] = {
        ["Metadata"] = {...},
        ["Value"] = 122321,
    },
    ...
}
```
Say we only want to access the `Value` parameter when directly indexing a key in the dictionary. However, the metadata is still important so we shouldn't remove it and hence we instead decide to use a metatable to mimic the behaviour:
```luau
local fooMetatable = setmetatable({}, {__index = function(_, key)
    return rawget(foo, key).Value
end})

print(fooMetatable.bar) -- "ExampleString", but no typechecking :C
```
Unfortunately, given the current constraints of the luau typechecker and our usage of a metatable; we've lost the ability to typecheck against the results of the function.

## Design
This issue would be become solvable if generic types were recognised when run in a metamethod; for example, we could do this:
```luau
local function ourIndexFunction<K>(_, key: keyof<typeof(foo)>&K)
    return rawget(foo, key).Value :: index<index<typeof(foo), K>, "Value">
end
local fooMetatable = setmetatable({}, {__index = ourIndexFunction})

print(fooMetatable.bar) -- "ExampleString"; typechecking *should* work here but it doesn't :\
```
> I'm aware that the `keyof(typeof(foo))&K` disables the auto-complete behaviour however it is very possible to work around that. I just wrote a very quick and simple code-sample to demonstrate what the ideal solution would be for the return typechecking.

Unfortunately, as of currently, the above code "works" but doesn't provide any typechecking support for the returned value. This is despite the fact that it works if you run it in a "normal" function not housed under a metamethod:
```luau
local function ourIndexFunction<K>(_, key: keyof<typeof(foo)>&K)
    return rawget(foo, key).Value :: index<index<typeof(foo), K>, "Value">
end

local result = ourIndexFunction({}, "bar")
print(result) -- "ExampleString", and we have typechecking!? (type is string or number depending on the key we've referenced)
```

## Drawbacks
- I'm not personally familiar on the internals of the typechecker but potentially this could lead to minimal performance decreases.
- While not necessarily a drawback; it could be seen that the typechecker should just be able to resolve these simple metamethods itself. However, I still think this "manual override" would be helpful regardless, since it would both be presumably easier to implement in the short-term as well as allow for more fine-tuned typechecking if the automated system gets it wrong.

## Alternatives
### The current alternatives are to either:
- Drop the typechecker and give all table entries a vague type; such as `any`. This isn't ideal since you effectively end up with no typechecking for the returned values.

- Try to simplify your data-structure so that the metadata is seperate from the values, dropping the need for a metatable entirely; this may seem great on the surface but ultimately can make your code more messy as you need to seperate away related metadata from the value.
_Example:_
```luau
local foo = {
    ["Metadata"] = {
        ["bar"] = {...},
        ["taz"] = {...},
        ...
    },
    ["Value"] = {
        ["bar"] = "ExampleString",
        ["taz"] = 122321,
        ...
    }
}
```

- Hack together a solution with functions which support generic types. The tradeoff here is that you'll need to use your own custom function to retrieve / set the data rather than being able to just use the key on the dictionary itself.

- Optionally, you could just directly access `.Value` every time you wish to access the data; although this is quite inconvenient and certainly isn't ideal for code readability.

Another reason why the above alternatives aren't feasible for my use-case is that I want to use this in some of my upcoming public modules and hence don't want to have a 'unique way' to set values just so that I can satisfy the typechecker.