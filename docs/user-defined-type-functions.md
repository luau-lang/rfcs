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

Type functions will be defined with the following syntax:
```luau
type function f(...)
    -- implementation of the type function
end
```

For instance, the `rawget` type function can be written as:
```luau
type function rawget(tbl, prop)
    if not tbl:is("table") then
        error("First argument is not a table!") -- fails to reduce
    end

    return tbl:getprops()[prop:getvalue()]
end

type Person = {
    name: string,
    age: number
}

type ty = rawget<Person, "name"> -- ty = string
```

Type functions operate on two stages: type analysis and runtime. When calling type functions at the type level (e.g. annotating a variable as a type function), angle brackets must be used, but when calling them at the runtime level (e.g. calling other type functions within type functions), parenthesis must be used. Declarations of type functions use parenthesis because it defines the runtime operations on the runtime representation of types.

For the first iteration, the body of a type function will be sandboxed, and its scope will be limited, meaning it will be unable to refer statements defined in the outer scope, including other type functions. Additionally, type functions will be limited on what globals/libraries they could call. The list of available globals/libraries are:

- global functions: `assert`, `error`, `next`, `print`, `rawequal`, `select`, `tonumber`, `tostring`, `type`, `typeof`, `ipairs`, `pairs`, `unpack`
- math library
- table library
- bit32 library
- buffer library

There is also a problem of infinitely running type functions that can halt the analysis from making further progress. For example, reducing this type function will halt analysis until the VM stack overflows:
```luau
type function neverending(t)
    return neverending(t) -- note: parentheses are used here because the runtime value of types is being passed in, rather than the static annotation of types
end
```

The simplest approach (and the approach we plan on taking for the first iteration) and one that is currently supported by the Luau API is to enforce a time limit to the type function VM. We are aware that this methods are not consistently reliable. Time-based timeouts are dependent on CPU performance; programs that type check on fast CPUs may not type check on slower CPUs. We will be experimenting with various forms of limiting the execution of type functions and reserve the right to change how termination of type functions is managed.

</details>

To fail reductions, developers can use `error()` with custom error messages. Type functions always expect to have one return value of `type` instance.

To allow Luau developers to modify the runtime values of types in type functions, this RFC proposes introducing a new userdata called `type` (for the purpose of clarity, `type` refers to the userdata and type refers to their actual word). A `type` object is a runtime representation of all types within the program and provides a basic set of library methods that can be used to modify types. They are *only accessible within type functions* and are *not a runtime value/userdata/library anywhere else*.

Because the name clashes with the global function `type()`, this new userdata's `__call` metamethod will be set to the original `type()` function.

<details><summary>`type` library (dropdown)</summary>

Methods under a different type heading (ex: `Singleton`) imply that the methods are only available for those types. At the implementation level, there is a check to make sure that the type-specific methods are being called on the correct types.

#### `type`
All attributes of newly created `type` are initialized with empty tables / arrays and `nil`. For instance, `type.newtable()` initializes its properties with an empty table and index / index result type as `nil`. Additionally, all arguments are passed by references.

| Instance Attributes | Type | Description |
| ------------- | ------------- | ------------- |
| `niltype` | `type` | an immutable runtime representation of the built-in type `nil` |
| `unknown` | `type` | an immutable runtime representation of the built-in type `unknown` |
| `never` | `type` | an immutable runtime representation of the built-in type `never` |
| `any` | `type` | an immutable runtime representation of the built-in type `any` |
| `boolean` | `type` | returns an immutable runtime representation of the built-in type `boolean` |
| `number` | `type` | returns an immutable runtime representation of the built-in type `number` |
| `string` | `type` | returns an immutable runtime representation of the built-in type `string` |

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `__eq(arg: type)` | `boolean` | overrides the == operator to return true if self is syntactically equal to arg |
| `is(arg: typestring)` | `boolean` | returns true if self has the same tag as the argument. List of available tags: "nil", "unknown", "never", "any", "boolean", "number", "string", "boolean singleton", "string singleton", "negation", "union", "intersection", "table", "function", "class".  |

| Static Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getnegation(arg: type)` | `type` | returns an immutable runtime representation of the negation of the argument; the argument cannot be an instance of a table or a function. |
| `getstringsingleton(arg: string)` | `type` | returns an immutable runtime representation of a string singleton type of the argument |
| `getbooleansingleton(arg: boolean)` | `type` | returns an immutable runtime representation of a boolean singleton type of the argument |
| `getunion(arg: {type})` | `type` | returns an immutable runtime representation of union type of its argument |
| `getintersection(arg: {type})` | `type` | returns an immutable runtime representation of intersection type of its argument |
| `newtable(props: {[type]: type}?, indexer: {key: type, value: type}?, metatable: type?)` | `type` | returns a mutable runtime representation of a `table` type. If provided the metatable parameter, this table becomes a metatable. |
| `newfunction(parameters: {type} \| type?, returns: {type} \| type?)` | `type` | returns a mutable runtime representation of a `function` type. Calling `newfunction(X)` will by default set `parameters` to `X` |
| `copy(arg: type)` | `type` | returns a deep copy of the argument |

#### Negation

| Instance Methods | Type | Description |
| ------------- | ------------- | ------------- |
| `gettype()` | `type` | returns the runtime representation of the self's type being negated |

#### StringSingleton

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getvalue()` | `string` | returns self's value of a string singleton |

#### BooleanSingleton

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getvalue()` | `boolean` | returns self's boolean singleton value of either `true` or `false` |

#### Table

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setprop(key: type, value: type?)` | `nil` | adds / overrides (if same key exists) a key, value pair to self's table properties; if value is nil, removes the key, value pair from self's table properties; if the key does not exist and the value is nil, nothing happens |
| `getprop(key: type)` | `type?` | returns the value associated with the key from self's table properties if the key exists, else nil |
| `getprops()` | `{[type]: type}` | returns a table of self's table properties (e.g. `{["age"] = 20}` will return `{type.getstringsingleton("age") = type.getnumber()}`) |
| `setindexer(key: type, value: type)` | `nil` | sets self's indexer key type to the first argument and indexer value type to the second |
| `getindexer()` | `{key: type, value: type}?` | returns a table containing self's indexer key type and value type if they exist, else nil |
| `setmetatable(arg: type)` | `nil` | sets self's metatable to the argument |
| `getmetatable()` | `type?` | returns self's runtime representation of metatable if it exists, else nil |

#### Function

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setparameters(arg: {type} \| type?)` | `nil` | sets self's parameter types to the argument, where an array implies a TypePack and the latter implies a Variadic |
| `getparameters()` | `{type} \| type?` | returns the runtime representation of self's parameter type if it exists, else nil. Return an array implies a TypePack and a single value implies a Variadic |
| `setreturns(arg: {type} \| type?)` | `nil` | sets self's return types to the argument, where an array implies a TypePack and the latter implies a Variadic |
| `getreturns()` | `{type} \| type?` | returns the runtime representation of self's return type if it exists, else nil. Return an array implies a TypePack and a single value implies a Variadic |

#### Union

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomponents()` | `{type}` | returns an array of types that the self's union can represent. For instance, `string \| number` returns `{type.string, type.number}` |

#### Intersection

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomponents()` | `{type}` | returns an array of types represented by self's intersection. For instance, `string & number` returns `{type.string, type.number}` |

#### Class

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getprops()` | `{[type]: type}` | returns the runtime representation self's properties |
| `getparent()` | `type?` | returns the runtime representation of self's parent class if it exists, else nil |
| `getmetatable()` | `type?` | returns the runtime representation of self's metatable if it exists, else nil |
| `getindexer()` | `{key: type, value: type}?` | returns a table containing self's indexer key type and value type |

</details>

The reason for going with userdata instead using another representation or adding a new compile-time interpreter like many other languages is that userdata provides a clean abstraction from the runtime representation and restricts developers to using a set of libraries controlled by the Luau team where we can easily manage changes and listen to feedback. Moreover, because Luau is designed to be easily embeddable, we wanted to use this as an advantage for type functions since they only require using a small runtime variant of the VM, making the implementation be simpler than having to implement a new interpreter.

### Implementation

A `type` library will be implemented using the Luau API to interface between C++ and Luau and support the library methods, including type serialization for arguments and deserialization for return values. To implement type functions, a new AST node called `AstStatTypeFunction` will be introduced and created when parsing a type alias followed by the keyword "function." In the constraint generator, visiting this new AST node will generate a `TypeAliasExpansionConstraint` and in the constraint solver, reducing `TypeAliasExpansionConstraint` will generate a `ReduceConstraint`. To reduce `ReduceConstraints`, user-defined type functions will be integrated as built-in type functions in Luau where when being invoked, their arguments will be serialized into an instance of `type`. These functions will interact with the Luau VM to execute the function body in an established environment with only the specified libraries and constructs available. The return value of the type function will be deserialized and be the value that the `ReduceConstraint` reduces to.

## Drawback

Type functions are handled at the analysis time, while `type` is an implementation in the runtime. As a result, the proposed design causes the analysis time to be dependent on Luau's runtime to reduce user-defined type functions. This is generally discouraged as it is best to isolate the compile time, analysis time, and runtime from each other for the purpose of maintaining a clean separation of concerns, which helps minimize side effects and dependencies across different phases of the program execution and improves the modularity of the compiler. Overlaps between the analysis time and runtime can lead to code that is more complex and harder to manage, potentially increasing the risk of bugs and making the outcomes less predictable.

The build / analysis times will also be negatively impacted as reducing type functions takes variable amount of time based on the program. Developers will be able to write non-performant code that impacts their (and any of their depedent code's) analysis time. The larger the type function, the longer it will take to reduce it in the constraint solver.

## Alternatives

### `table` Runtime Representation

Currently, the runtime representation of types is a userdata called `type`. Another representation is to use the already-existing type in Luau `table`; instead of serializing types into `type`, we can serialize them into tables with predefined properties. For instance, the representation for a string singleton `"abc"` could be `{type = "stringSingleton", value = "abc"}`. So instead of writing:
```luau
type function issingleton(t)
    if t:is("string singleton") then
        return t
    end
end
```
developers could write:
```luau
type function issingleton(t)
    if t.type == "string singleton" then
        return t
    end
end
```

In some sense, this design could be considered "cleaner" than introducing an entirely new userdata, but it requires developers to have a deeper understanding of the type runtime representation. For example, under the proposed design, a new table type can be created using `type.newtable()`, while under the alternative design, tables must be declared with attributes: `{type = "table", props = {}, indexer = {}}`. This adds complexity to the developer experience and increases the chance of making syntax errors in their program.

### More Builtin Type Functions

Another alternative is to simply let go of this idea and create more built-in type functions that cover a wider range of common use cases. This approach clearly lacks the flexibility of fully user-defined type functions because type manipulation would be limited to the predefined set of type functions designed by the Luau team. Furthermore, continuously expanding the set of built-in type functions could lead to bloat and complexity within the core language, making it harder to maintain.

### Type Function Type Checking

In the future, we could investigate adding type checking to user defined type functions. In Haskell's type families, this is done with _kinds_ (aka types of types). Similarly, we could look into introducing kinds to Luau, but for the purpose of this RFC, this feature will be left for future works.