# Improved type checking rules of cast operator

## Summary

Replace the bivariance rule of the cast operator `::` by a rule of intersection inhabitance.

## Motivation

This [RFC](./syntax-type-ascription.md) proposed a rule such that the cast is legal if and only if it is an upcast, i.e. given a term `number`, you can upcast into `number | string`. An implementation error instead actually allowed _downcasts_ only, i.e. given a term `number | string`, you can downcast into `number` but not the other way.

We relaxed it in this [RFC](./syntax-type-ascription-bidi.md) by making the cast operator _bivariant_, which is to test downcasts _or_ upcasts, so now you can cast from `number` to `number | string` and also `number | string` to `number`.

The current rule almost works for some use cases, but not all. For instance, you cannot cast `string` into `"a"?` because neither `string <: "a"?` nor `"a"? <: string` holds true:
  - Upcast: `string` is not a subtype of `"a"`
  - Downcast: `"a"` is a subtype of `string`, but `nil` is not a subtype of `string`.

The workaround currently is to have an intermediary `any` or `unknown` cast: `(e :: any) :: "a"?` where `e : string`.

## Design

We propose that cast operator should instead test for whether there exists a common type from the type of the expression and the type we wish to cast into.

For example, `e :: T` will report an error if and only if `typeof(e) & T` is uninhabited, unless `typeof(e)` is already uninhabited. More concretely:

```luau
local function noop(x) end

local function f(e: number | string)
    -- OK
    noop(e :: string)           -- (number | string) & string ~ string
    noop(e :: number)           -- (number | string) & number ~ number
    noop(e :: string | boolean) -- (number | string) & (string | boolean) ~ string

    -- Not OK
    noop(e :: boolean) -- (number | string) & boolean ~ never
    noop(e :: never)   -- (number | string) & never ~ never

    -- Special cases
    noop(error("") :: string) -- OK
end
```

The reason why the special case oughtn't report an error is to support ad hoc typed holes pattern instead of having to hand-craft an expression that matches that type:

```luau
local x = error("") :: string | number
-- versus
local x = if math.random() > 0.5 then "hello" else 5
```

We don't apply the same special case for `T`, otherwise we won't report an error when `e : string` and `T` is `never`. This would mean we get to support the exhaustive analysis use case:

```luau
local function f(e: number | string)
    if typeof(e) == "number" then
        -- ...
    elseif typeof(e) == "string" then
        -- ...
    else
        noop(e :: never) -- Statically asserts that `e` is indeed `never`
    end
end
```

## Drawbacks

This does relax the rules of the cast operator significantly and despite that it's what users actually do with it, it's more likely to be error-prone than before, e.g. casting from `number | string` to `string | boolean` without `any` intermediary cast.

## Alternatives

Remove all the type checking rules and just let it run amok, such as casting `number` right into `string` with no type errors.
