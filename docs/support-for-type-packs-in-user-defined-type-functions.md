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
type A = f<(string, number)>
```

Type-packs can now be placed at the `tail` argument of function types:

```luau
type function f() 
    local newPack = types.pack({types.string, types.number, types.pack({any}, true)}) -- (string, number, ...any)
    local newfunc = types.newfunction()
    newfunc:setparameters({}, newPack)
    newfunc:setreturns({}, newPack)
    return newfunc
end
type A = f<> -- (string, number, ...any) -> (string, number, ...any)
```

*To maintain backwards-compatibility, `tail` argument will still be able to take `type`s.*

The pack type will also enable generic type packs to be provided to type functions from aliases in the type runtime.
Generic type pack arguments will pass the given type pack to the type function.

```luau
type function createCallback(args: pack, returns: pack) -- args will be a literal pack, returns will be a variadic pack.
    local newTable = types.newtable()
    local newfunc = types.newfunction()
    newfunc:setparameters({}, args)
    newfunc:setreturns({}, returns)
    newTable:setproperty(types.singleton("f"), newfunc)
    return newTable
end
type Callback<Args..., Rets...> = createCallback<Args..., Rets...>
type A = Callback<(number, string), ...number> -- { f: (number, string) -> ...number }
```

Evaluating `typeof(...)` on a `pack` userdata will give the string `"pack"`.

```luau
type function f(arg: pack)
    print(typeof(arg)) -- "pack"
end
type a = f<(string, number)>
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
| Update | `newfunction(parameters: { head: {type}?, tail: type \| pack? }, returns: { head: {type}?, tail: type \| pack? }, generics: {type}?)` | `type` | `tail` arguments can now accept packs.

#### Function `type` instance

| New/Update | Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| Update | `setparameters(head: {type}?, tail: type \| pack?)` | `()` | `tail` argument can now accept packs. |
| Update | `setreturns(head: {type}?, tail: type \| pack?)` | `()` | `tail` argument can now accept packs. |

## Drawbacks

This may bring additional complexity to type functions, and since we're bringing a new userdata to the runtime, type functions now get two types of parameters, `type` or `pack`.
This doesn't pose a problem when the arguments are annotated (`arg: type`), but in unannotated arguments (`arg`), this may pose a problem.

To solve this problem, inference can treat all provided arguments as `type`, until it encounters a `pack` related function.

## Alternatives

Do nothing and force type functions to only accept singular type parameters.
