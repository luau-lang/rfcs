# User-Defined Type Functions

## Summary

The new type solver for Luau supports a limited number of built-in type functions that enable developers to be expressive with their code. To further enhance this expressivity and the flexibility of the type system, this RFC proposes the design and implementation of user-defined type functions, a feature that will allow developers to define custom type functions.

## Motivation

The primary motivation for introducing user-defined type functions is to extend the flexibility of the type system and empower developers to create more expressive and type-safe libraries. Current limitations in the type system restrict custom type manipulations, which are crucial for creating advanced patterns and abstractions. By supporting user-defined type functions, we aim to:
- Enable more precise type definitions that can capture complex relationships and constraints
- Facilitate the creation of reusable and composable libraries
- Enhance code safety and maintainability with better type-level programming

The expected outcome is a more versatile type system that can adapt to a wider range of programming patterns, ultimately improving the Luau developer experience.

## Design

Type functions wil be defined with the following syntax:
```luau
type function f(...)
    -- implementation of the type function
end
```

For instance, the `rawget` type function can be written as:
```luau
type function rawget(tbl, prop)
    return tbl.getProps()[prop.getValue()]

    -- returning nothing / nil means that this type function will fail to reduce
end

type Person = {
    name: string,
    age: number
}

type ty = rawget<Person, "name"> -- ty = string
```

Type functions operate on two stages: type analysis and runtime. They are _called_ at the type level of Luau, but _runs_ at runtime to operate on the values of types (such as the arguments of the function). For this reason, type functions are declared using parenthesis and invoked using angle brackets. Moreover, every type functions will be part of and ran within the same scope to allow type functions to call other type functions. We will not (and probably never) allow type functions to call regular functions for the sake of sandboxing the runtime and analysis time. There will also be a user-configurable timeout for type functions based on the number of instructions. For Roblox, this could be done by editing the config file for RCC and Studio and for non-Roblox users, this could be done by setting a flag when running the compiler.

To allow Luau developers to modify the runtime values of types in type functions, this RFC proposes introducing a new userdata called `lType`.  An `lType` object is a runtime representation of all types within the program and provides a basic set of library methods that can be used to modify types. As such, under the hood, each argument of a type function is serialized into a userdata called `lType`. Moreover, they are *only accessible within type functions* and are *not a runtime type for other use cases than type functions*. 

<details><summary>lType library methods (dropdown)</summary>
Note: methods under a different type heading (ex: `Singleton`) imply that the methods are only available for those types. At the implementation level, there is a check to make sure that the type-specific methods are being called on the correct types (e.g, for `getIndexer()`, assert that `isTable()` is true).

#### Any

| Function Declaration | Return Type | Description |
| ------------- | ------------- | ------------- |
| `isstring()` | `boolean` | returns true if self is of type `string` |
| `isnumber()` | `boolean` | returns true if self is of type `number` |
| `isboolean()` | `boolean` | returns true if self is of type `boolean` (e.g. true or false) |
| `istable()` | `boolean` | returns true if self is of type `table` |
| `isthread()` | `boolean` | returns true if self is of type `thread` |
| `isfunction()` | `boolean` | returns true if self is of type `function` |
| `isbuffer()` | `boolean` | returns true if self is of type `buffer` |
| `isnil()` | `boolean` | returns true if self is of type `nil` |
| `isclass()` | `boolean` | returns true if self is of type `class` (do we need this?) |
| `isbooleansingleton()` | `boolean` | returns true if self is a boolean singleton |
| `isstringsingleton()` | `boolean` | returns true if self is a string singleton |
| `isa(arg: lType)` | `boolean` | returns true if arg is the same type as self |

#### Primitive

| Function Declaration | Return Type | Description |
| ------------- | ------------- | ------------- |
| `gettype()` | `string` | returns either "nil", "boolean", "string", "thread", "function", "table", or "buffer" |

#### Singleton

| Function Declaration | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getvalue()` | `string` | returns either "true", "false", or a string singleton |

#### Table

| Function Declaration | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getprops()` | `table` | returns a type representation of tables (e.g. {name = "John"} will return {[string] = "string"}) |
| `getindexer()` | `table` | returns a type representation of arrays (e.g. {1, "hi", 3} will return {[number] = "number" \| "string"}) |

#### Class

| Function Declaration | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getname()` | `string` | returns the name of self's class |
| `getparent()` | `lType` | returns lType userdata of self's parent |

</details>

### Implementation

The implementation of user-defined type functions can be broken down into the following parts:

**AST**: Introduce a new AST node for type functions called `AstStatTypeFunction`.

**Parser**: When parsing TypeAliases, if the next lexeme is "function", parse it into the `AstStatTypeFunction` node.

**Constraint Generator**: Generate a `TypeAliasExpansionConstraint` when visiting the `AstStatTypeFunction` node.

**Constraint Solver**: Generate a new `ReduceConstraint` and append it to the list of unsolved constraints when reducing `TypeAliasExpansionConstraint`.

**Reduction Constraint**: Serialize the function arguments into `lType` and execute the body of the function. Reduce to the deserialized version of the function return value.

**lType**: Using the Lua API, create a library that interfaces between C++ and Luau and supports all of the function calls, serialization, and deserialization.

## Drawback

Type functions are handled at the analysis time, while `lType` is an implementation in the runtime. As a result, the proposed design causes the analysis time to be dependent on Luau's runtime to reduce user-defined type functions. This is generally discouraged as it is best to isolate the compile time, analysis time, and runtime from each other for the purpose of maintaining a clean separation of concerns, which helps minimize side effects and dependencies across different phases of the program execution and improves the modularity of the compiler. Overlaps between the analysis time and runtime can lead to code that is more complex and harder to manage, potentially increasing the risk of bugs and making the outcomes less predictable.

The build / analysis times will also be negatively impacted by this feature as reducing type functions takes variable amount of time based on the program. The larger the type function, the longer it will take to reduce it in the constraint solver.

## Alternatives

### `table` Runtime Representation

Currently, the runtime representation of types is a userdata called `lType`. Another representation is to use the already-existing type in Luau `table`; instead of serializing types into `lType`, we can serialize them into tables with predefined properties. For instance, the representation for a string singleton `"abc"` could be `{type = "stringSingleton", value = "abc"}`. So instead of writing:
```luau
type function isSingleton(t)
    if t:isStringSingleton() then
        return t
    end
end
```
developers could write:
```luau
type function isSingleton(t)
    if t.type == "stringSingleton" then
        return t
    end
end
```

In some sense, this design could be considered "cleaner" than introducing an entirely new userdata, but it requires developers to have a deeper understanding of the type runtime representation. For example, under the proposed design, a new table type can be created using `lType.new("table")`, while under the alternative design, tables must be declared with attributes: `{type = "table", props = {}, indexer = {}}`. This adds complexity to the developer experience and increases the chance of making syntax errors in their program.

### More Builtin Type Functions

Another alternative is to simply let go of this idea and create more built-in type functions that cover a wider range of common use cases. This approach clearly lacks the flexibility of fully user-defined type functions because type manipulation would be limited to the predefined set of type functions designed by the Luau team. Furthermore, continuously expanding the set of built-in type functions could lead to bloat and complexity within the core language, making it harder to maintain.

### Type Function Type Checking

In the future, we could investigate adding type checking to user defined type functions. In Haskell's type families, this is done with _kinds_ (aka types of types). Similarly, we could look into introducing kinds to Luau, but for the purpose of this RFC, this feature will be left for future works.