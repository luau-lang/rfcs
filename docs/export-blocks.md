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

Some modules construct their public interface through obtaining multiple values from a helper function. With the current syntax, each value must be extracted and exported individually. Consider:

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

### Better grouping related exports

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

Grouping exports is already possible with individual declarations, but export blocks make the grouping explicit in the syntax, while also making it visually distinct.

### Enhanced distinctness

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

The second form accepts an expression that evaluates directly into the first form:

```luau
export someReturnedExport()
```

The contents of the resulting table are exported as if they had been declared individually. As an added security measure, the linter can raise a warning of a possible runtime error if the type of the expression cannot be resolved.

Both forms are restricted to the top level of a module, consistent with the existing export syntax.

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

However, numeric keys and named keys introduce additional considerations for static analysis and type inference. For simplicity, numeric keys behave like named keys; they cannot have a duplicated definition.

Note that supporting mixed tables would make export blocks more consistent with ordinary Luau table constructors and simplify migration from legacy return tables. But I want to avoid mixed tables as much as possible, so I would prefer if exports were restricted to either pure arrays or pure key-value dictionaries.

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

### Use the legacy behavior

Simply use the old method:

```luau
return {
    a = something,
    b = somethingElse,
}
```
Unfortunately, using returns voids all benefits of exports. This isn't preferred.

## Prior Art

JavaScript supports exporting multiple existing bindings through a single declaration:

```javascript
export { a, b };
```

<details>
<summary>Spoiler warning</summary>
    AI was used to draft this RFC. If there are any issues I missed, please point it out!
