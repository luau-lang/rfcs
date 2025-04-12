# Variadic User-Defined Type Functions

## Summary

Add support for variadic arguments for user-defined type functions.

## Motivation

Today, user-defined type functions can only take a pre-defined set of parameters. This limits the usage in a way that makes it not possible to make the function variadic, where the user can supply
the function as many arguments as provided.

By adding support for variadic functions, we allow users to call user-defined type functions with as many arguments as they wish, providing a more dynamic system.

## Design

The syntax for variadic type functions would be exactly the same as normal variadic functions where `...` can only be legal as the last argument of the function. 
Utilizing either `table.pack(...)` method or `{...}`, the user would recieve a table of `type`s to use within their code. 

### Examples

Creating a variadic type function:

```luau
type function f(...)
    local packed = {...} -- {type}
    for _, type in packed do
        -- Use the provided types.
    end
end

type a = f<string, number>
```

Calling a variadic type function from a type alias:

```luau
type function f(...)
    local packed = {...}
    local firstArg = packed[1] -- tag: string
    local secondArg = packed[2] -- tag: number
    local thirdArg = packed[3] -- tag: table
end

type a<T, U, V> = f<T, U, V>
type b = a<string, number, {}>
```

## Drawbacks

It's unclear if this proposal has any drawbacks.

## Alternatives

Do nothing, and force type functions to only take pre-defined set of parameters.
