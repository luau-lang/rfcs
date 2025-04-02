# type:issubtypeof

## Summary

New method on the [`type`](./user-defined-type-functions.md#type-instance) userdata in type functions for performing subtype checks.

## Motivation

Currently there is no built in way to do subtype checks in User-Defined Type Functions. This means the developer has to write the following if they wanted to do a subtype check. Subtype checks are useful for type functions, because it makes code like in the following 2 example cases unneeded.

Case 1: Checking if a type is a string or a string singleton
```luau
local is_string = t:is("string") or (t:is("singleton") and type(t:value()) == "string")
```

Case 2: Checking if a type is an enum
```luau
type function isenum(t)
    if t:is("union") then
        local components = t:components()
        local components_that_are_string = 0

        for _, component in components do
            if tv:is("string") or (tv:is("singleton") and type(tv:value()) == "string") then
                components_that_are_string += 1
            end
        end
        return types.singleton(
            components_that_are_string == #components
        )
    else
        return types.singleton(false)
    end
end
```

Additionally it would be a pain for users to write a type function to be able to do subtype checks for any type. As the following example type function is already very long, despite not covering everything.

```luau
type function issubtypeof(a, b)
    local false_type = types.singleton(false)
    local true_type = types.singleton(true)
    local is_subtype = false
    local a_tag = a.tag
    local b_tag = b.tag

    if 
        (a_tag == b_tag) or
        (b_tag == "singleton" and type(b:value()) == a_tag) or
        (b_tag == "negation" and type(b:inner()) == a_tag)
    then
        is_subtype = true
    elseif b_tag == "union" or b_tag == "intersection" then
        local components = b:components()
        local conmponents_that_are_subtype = 0

        for _, component in components do
            if issubtypeof(component):value() then
                conmponents_that_are_subtype += 1
            end
        end

        return types.singleton(#components == conmponents_that_are_subtype)
    elseif b_tag == "table" and a_tag == "table" then
        local b_mt = b:metatable()
        local a_mt = a:metatable()

        if b_mt == a_mt then
            return true_type
        elseif b_mt and a_mt then
            local b_mts = { b_mt }
            local next_b_mt
            local next_a_mt = a_mt

            while next_b_mt do
                table.insert(b_mts, next_b_mt)
                next_b_mt = next_b_mt:metatable()
            end

            while next_a_mt do
                for _, mt in b_mts do
                    if mt == next_a_mt then
                        return true_type
                    end
                end
                next_a_mt = next_a_mt:metatable()
            end
        else
            --[[
                For the purposes of this rfc, my point has already been made. And I do not want to write the rest of this type function that nobody should ever use,
                as its probably going to be an easy way to hit the type function time limit.
            --]]
        end
    end

    return types.singleton(is_subtype)
end
```

## Design

The [`type`](./user-defined-type-functions.md#type-instance) will gain a new method: `issubtypeof`:

```luau
type:issubtypeof(arg: type): boolean
```

This method will return a boolean indicating if the type is a subtype of `arg`, using luau's existing subtyping algorithm. 

## Drawbacks

Adds another method to the type userdata in User-Defined type functions.

## Alternatives

Do nothing, and let users write the checks themselves as it is currently.
