# Shorthand field initialization

## Summary

This RFC proposes a shorthand syntax for declaring fields in tables, which allows writing
`.fieldName` instead of `fieldName = fieldName` when the value of the field is being set to a
variable of the same name as the field.

## Motivation

It is very common for the variable which you assign as the value of a field in a table to have the
same name as said field.

In these cases, you end up writing the name of the variable at least three times. The following
block of code illustrates an example of this.
```lua
local str = ":)"
local tbl = {
    str = str,
}
```

This repetition makes it less ergonomic to declare *variables* which you assign to
fields, as opposed to *embedding* the value of those variables directly in the initialization of the
field. As an example of this, the above code snippet could be written as the following:
```lua
local tbl = {
    str = ":)",
}
```

This modification is often not ideal. For example, if the computation of the value of some fields
depends on the value of other fields, or if the expression which would be used instead of the
variable in the field initialization is too complex. Another example is module exports.

A module exporting some function `init` *may* look like the following:
```lua
local function init()
    -- ...
end

return table.freeze({
    init = init,
})
```
As you can see, the one field's variable (`init`) has to be written at least three times. Using a
different method of exporting variables from modules, you may end up with the following code:
```lua
local module = {}

function module.init()
end

return table.freeze(module)
```
Using this method, the name of the returned table needs to be written (2 + n) times, where n is the
number of variables you export. Every exported variable also needs to be written at least one or
three times. Potentially three times because you may need to localize exported variables before
adding them to the returned table. An example of this would be the following.
```lua
local module = {}

export type TypeWhichSomeExportedVariableHas = {
    str: string,
}

-- localize first so that we can specify the type of `SOME_EXPORTED_VARIABLE`.
local SOME_EXPORTED_VARIABLE: TypeWhichSomeExportedVariableHas = {
    str = ":)",
}

module.SOME_EXPORTED_VARIABLE = SOME_EXPORTED_VARIABLE

return table.freeze(module)
```
So, this method may (arguably) be more ergonomic or less ergonomic than the method which first was
mentioned. However, using shorthand field initialization syntax instead, it could be written like
this:
```lua
export type TypeWhichSomeExportedVariableHas = {
    str: string,
}

local SOME_EXPORTED_VARIABLE: TypeWhichSomeExportedVariableHas = {
    str = ":)",
}

return table.freeze({
    .SOME_EXPORTED_VARIABLE,
})
```
Which (arguably) is better.

## Design

### Grammar

The proposed grammar expands the `field` rule to the following:
```
field = '[' exp ']' '=' exp | NAME '=' exp | exp | '.' NAME
```

### Example

For some module exporting a type, `Phone`, and a function, `init`, which creates a table of type
`Phone`, the proposed syntax may be used in the following way:
```lua
export type Phone = {
    number: number,
    owner: string,
}

local function init(number, owner): Phone
    return {
        -- NOTE: The following line of code may be written in a number of different, but equivalent
        -- ways. For example:
        -- * .number;
        -- * number = number,
        -- * number = number;
        .number,
        -- NOTE: The following line of code may be written in a number of different, but equivalent
        -- ways. For example:
        -- * .owner;
        -- * .owner (since it's the last element)
        -- * owner = owner,
        -- * owner = owner;
        -- * owner = owner (since it's the last element)
        .owner,
    }
end

return table.freeze({
    -- NOTE: The following line of code may be written in a number of different, but equivalent 
    -- ways. For example:
    -- * .init;
    -- * .init (since it's the last element)
    -- * init = init,
    -- * init = init;
    -- * init = init (since it's the last element)
    .init, 
})
```

## Drawbacks

Introducing this shorthand syntax adds a small level of complexity to the language, which
potentially is confusing to newcomers. It also occupies syntax which might preferably be used for
other features in the future.

## Alternatives

The added grammar to the `field` rule needs to have a prefix included, since it otherwise would be
indistinguishable from initialization of array-like table fields, but there are alternatives for
what that prefix may be.

Additionally, if records are introduced in the future, this type of syntax could be used for them
exclusively, and without a prefix, arguably making code look "cleaner".