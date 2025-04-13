# Support for Type Packs in User-Defined Type Functions

## Summary

Add support for type packs in user defined type functions.

## Motivation

Currently, user-defined type functions are limited in terms of support for type packs. The only place where a type pack is represented as a runtime type, is within generic `type` instances, 
where the instance can either be a generic type, or a generic type pack. Other kinds of packs such as variadic packs can only be created within 'tail' spots. 

This limits the usage for packs as they are used often in the language, and their creation and usage should be easier and more expanded upon.

## Design

We introduce a new userdata called `pack` alongside `type` to extend the support of type packs within the type runtime.

`pack` instances will have a `kind` property that specifies the kind of the pack. A `pack` can either be of a `literal` kind or of a `variadic` kind.

For example, consider a variadic type pack (e.g `...number`), in the new pack type, it will be represented as a `pack` with a `variadic` kind. 
Or for a normal type pack such as `(number, string)`, it will be represented as a `pack` with a `literal` kind.

`literal` packs will be able to contain other packs, or types more than one, whereas `variadic` kind of packs will only contain *one* type.

The user will be able to obtain the `type`s within `pack`s using the `:unpack()` method, this method will return a table containing all the `type`s or `pack`s within the `pack`.
In `variadic` kind of `pack`s, `:unpack()` will only return a table with one element. This is because variadic packs are essentially many elements of the same type.

In user-defined type functions, type packs can be created by providing types or type packs that will be included within the pack. A pack will also be able to contain other packs, even packs of different kinds.

### Examples

Creating a type function that accepts type packs:

```luau
type function f(arg: pack)
    if typeof(arg) ~= "pack" then error("The given argument is not a type pack!") end
    local unpacked = arg:unpack() -- {string, number}
end
type a = f<(string, number)>
```

Creating a type function that returns a function with a type pack:

```luau
type function f() 
    local newPack = types.pack({types.string, types.number}) -- (string, number)
    local newVariadicPack = types.pack({types.any}, true) -- ...any

    local newfunc = types.newfunction()
    newfunc:setparameters({newPack}, newVariadicPack)
    newfunc:setreturns({newPack}, newVariadicPack)
    return newfunc
end
type A = f<> -- (string, number, ...any) -> (string, number, ...any)
```

The pack type will also enable generic type packs to be provided to type functions from aliases in the type runtime.
Generic type pack arguments will pass the given type pack to the type function.

```luau
type function createCallback(args: pack, returns: pack) -- args will be a literal pack, returns will be a variadic pack.
    local newTable = types.newtable()
    local newfunc = types.newfunction()
    newfunc:setparameters({args})
    newfunc:setreturns({returns})
    newTable:setproperty(types.singleton("f"), newfunc)
    return newTable
end
type Callback<Args..., Rets...> = createCallback<Args..., Rets...>
type A = Callback<(number, string), ...number>
```

If variadic type functions were to be implemented, type packs could be retrieved like this:

```luau
type function f(...)
    local packed = {...} -- {pack}
    local firstPack = packed[1] -- pack.kind: literal
    local secondPack = packed[2] -- pack.kind: variadic
end
type a = f<(string, number), ...number>
```

Evaluating `typeof(...)` on a `pack` userdata will give the string `"pack"`.

```luau
type function f(arg: pack)
    print(typeof(arg)) -- "pack"
end
type a = f<(string, number)>
```

## New pack userdata

### `pack` Instance

| New/Update | Instance Properties & Methods | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New | `kind` | `"literal" \| "variadic"` | indicates the kind of the `pack`. |
| New | `unpack()` | `{pack \| type}` | returns the `type`s contained within the `pack`, returns only one element if `kind` is `variadic`. |

## Updates to the types library and the type userdata

### `types` Library

| New/Update | Library Functions | Return Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New |  `pack(args: {pack \| type}, isvariadic: boolean?)` | `pack` | returns an immutable instance of a type pack; when `isvariadic` is true, `args` table can only have one `type`. |
| Update | `newfunction(parameters: { head: {type}?, tail: type? }, returns: { head: {type}?, tail: type? }, generics: {type}?)` | `type` | `tail` arguments can now accept variadic packs.

#### Function `type` instance

| New/Update | Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| Update | `setparameters(head: {type}?, tail: type?)` | `()` | `tail` argument can now accept variadic packs. |
| Update | `setreturns(head: {type}?, tail: type?)` | `()` | `tail` argument can now accept variadic packs. |

## Drawbacks

This may bring additional complexity to type functions, and now variadic types will no longer have special meanings only in a tail spot.

## Alternatives

Do nothing and force type functions to only accept singular type parameters.
