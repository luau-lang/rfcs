# RFC: Export Blocks

## Summary

This RFC proposes introducing export blocks to Luau's module export syntax. Export blocks allow multiple values to be exported through a single `export` statement, either by explicitly listing named values or by exporting the contents of a table expression. This provides a more concise and visually distinct way to declare a module's public interface, simplifies migration from traditional table-returning modules, and makes it easier to export computed values without manually declaring each export individually.

## Motivation

The introduction of exported values in Luau provides an alternative to returning a table from a ModuleScript. However, the current syntax requires each exported value to be declared individually, which can become verbose when a module exposes many values or when those values are naturally grouped together.

Export blocks aim to improve this experience in several ways.

### Easier migration from legacy modules

Consider migrating something like this:

```luau
return {
    a = a,
    b = b,
    ... other fields
}
```

To migrate, we currently have to individually export each element:
```luau
export const a = a,
export const b = b,
... other fields
```

With export blocks, a simple find-and-replace of `return {}` to `export {}` would suffice:

```luau
export {
    a = a,
    b = b,
}
```

### Simplified exporting of computed values

Some modules construct their public interface dynamically or obtain multiple values from a helper function. With the current syntax, each value must be extracted and exported individually. Consider:

```luau
local function someReturnedExport()
    return {
        get = function() end,
        set = function() end,
        reset = function() end,
    }
end

local exported = someReturnedExport()

export const get = exported.get
export const set = exported.set
export const reset = exported.reset
```

An export block that accepts a table expression could simplify this to:

```luau
export someReturnedExport()
```

Table expressions are functionally the same as doing:
```luau
const exportedValues = someReturnedExport()
export exportedValues
```

Which can be further desugared into:
```lua
const exportedValues = someReturnedExport()
export const get = exportedValues.get
export const set = exportedValues.set
export const reset = exportedValues.reset
```

... the third option of which is already legal code in Luau.

### Grouping related exports

Instead of:
```luau
export apple1 = 1
export apple2 = 2
...

export banana1 = 1
export banana2 = 2
...
```

We can do:
```luau
export {
    apple1 = 1,
    apple2 = 2,
    ...
}

export {
    banana1 = 1,
    banana2 = 2,
    ...
}
```

This is particularly useful for modules that expose constants, configuration values, related functions, or other logically distinct groups of values.

Grouping exports is already possible with individual declarations, but export blocks make the grouping explicit in the syntax, while also making it visually distinct.

### Improved readability

Individual export declarations can look very similar to ordinary local declarations:

```luau
local a = something
local b = somethingElse

export const c = anotherThing
export const d = yetAnotherThing
```

When a module declares its exports at the end of the file, distinguishing its public interface from its implementation can become difficult.

Export blocks make the distinction more apparent:

```luau
local a = something
local b = somethingElse

export {
    c = anotherThing,
    d = yetAnotherThing,
}
```

This provides a clearer visual separation between implementation details and the module's public interface.

### Support for array-style exports

The current named export syntax does not provide a direct way to expose top-level array entries as module exports.

Consider this case:
```luau
return { "foo", "bar", "foobar" }
```

There is currently no equivalent export feature to support this.
Export blocks could support numeric keys by using the same table-constructor semantics as ordinary Luau tables.

```luau
export {
    something,
    somethingElse,
}
```

Consumers could then access these values using numeric indexing on the table returned by `require`.
```luau
const someArray = require("./someArray")
print(someArray[1])
```

## Design

### Syntax

This RFC proposes two forms of export blocks.

The first form is an export block containing a table constructor:

```luau
export {
    a = something,
    b = somethingElse,
}
```

Each field in the table constructor represents an exported value. Named fields become named exports, while list fields become numerically indexed exports.

The second form accepts an expression that evaluautes to a table:

```luau
export someReturnedExport()
```

The contents of the resulting table are exported as if they had been declared individually.

Both forms are restricted to the top level of a module, consistent with the existing export syntax.

### Named export blocks

Named export blocks allow multiple values to be exported through a single declaration.

```luau
local function getValue()
    return 10
end

local function setValue(value)
    -- Implementation
end

export {
    get = getValue,
    set = setValue,
}
```

This is conceptually equivalent to:

```luau
local function getValue()
    return 10
end

local function setValue(value)
    -- Implementation
end

export const get = getValue
export const set = setValue
```

The export block is syntactic sugar for the individual export declarations. It does not introduce a new scoping mechanism or change the semantics of the values being exported.

The expressions assigned to each field are evaluauted according to normal Luau evaluaution rules.

### Exporting a table expression

An export statement may also accept an expression that evaluautes to a table.

```luau
local function createExports()
    return {
        get = function()
            return 10
        end,

        set = function(value)
            -- Implementation
        end,
    }
end

export createExports()
```

This form exports the entries of the returned table rather than exporting the table itself as a single value.

It is conceptually equivalent to:

```luau
local exported = createExports()

export const get = exported.get
export const set = exported.set
```

This is especially useful when a module already has a helper that constructs its public interface, or when its exports are calculated by another function.

The expression must evaluaute to a table. The language should reject an incompatible expression when its type is statically known, and provide a runtime error when the result cannot be verified statically.

### Multiple export blocks

A module may contain multiple export blocks.

```luau
export {
    apple1 = 1,
    apple2 = 3,
}

export {
    banana1 = something,
    banana2 = somethingElse,
}
```

All blocks contribute to the same module export table. The blocks are organizational constructs and do not create separate namespaces.

Export names must remain unique across the entire module, not merely within an individual block. Duplicate names should produce an error, consistent with existing export rules.

### Array and mixed-table exports

Export blocks should follow ordinary Luau table-constructor semantics.

For example:

```luau
export {
    "foo",
    "bar",
    frog = something,
    cat = somethingElse,
}
```

This should produce exports equivalent to a table containing the following entries:

```luau
-->> someModule
{
    [1] = "foo",
    [2] = "bar",
    frog = "Frog",
    cat = "Cat",
}
```

Consumers could access the exports through the module table:

```luau
local Module = require("./someModule")

print(Module[1]) --> "foo"
print(Module[2]) --> "bar"
print(Module.frog) --> "Frog"
print(Module.cat) --> "Cat"
```

Supporting mixed tables would make export blocks more consistent with ordinary Luau table constructors and simplify migration from legacy return tables.

However, numeric keys and named keys introduce additional considerations for static analysis and type inference. The implementation must define how numeric exports are represented in the inferred type of the required module and how duplicate keys are handled.

I am against mixed tables in general, so I would prefer if exports were restricted to either pure arrays or key-value dictionaries, which would also help in implementation.

### Interaction with existing export restrictions

Export blocks should follow the same restrictions as existing value exports.
The table-expression form must also define behavior for invalid results and conflicting keys, as described above.

## Drawbacks

### Additional syntax complexity

Export blocks introduce another way to express module exports. Luau already supports individual value exports and type exports, so adding blocks increases the number of supported forms that developers and tooling must understand.

This also increases the complexity of the parser, formatter, type checker, and language tooling.

### Potentially redundant functionality

Most of the functionality provided by export blocks can already be achieved with individual export declarations.

Developers who prefer explicit declarations may see export blocks as unnecessary syntactic sugar.

### Less explicit declarations

Individual export declarations make it immediately apparent which identifiers are exported. With table-expression exports, the exported names may be less obvious without examining the expression's implementation.

This could make some modules harder to understand, especially when the expression returns a table assembled dynamically.

### Increased complexity around numeric exports

Allowing array-style and mixed-table exports introduces additional complexity for type inference, duplicate detection, and editor support.

## Alternatives

### Continue using individual export declarations

Everything in this RFC, except array exports, are already possible with the current syntax:

```luau
export const a = something
export const b = somethingElse
```

### Export a local table

Developers can continue to construct a local table and export its individual fields:

```luau
local exports = {
    a = something,
    b = somethingElse,
}

export const a = exports.a
export const b = exports.b
```

This works, but requires each exported field to be declared again. Export blocks eliminate this duplication.

## Prior Art

JavaScript supports exporting multiple existing bindings through a single declaration:

```javascript
export { a, b };
```
