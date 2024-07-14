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
    if typelib.isunion(tbl) or typelib.isunion(prop) then
        print("Warning: union types are not supported!") -- outputs a warning
    end

    if not typelib.istable(tbl) then
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

For the first iteration, type functions will support a subset of the Luau language, mainly the constructs that do not allow type functions to run infinitely. Infinitely running type functions are problematic because they can halt the analysis from making further progress. For example, reducing this type function will halt analysis until a stack overflow occurs in the VM:
```luau
type function neverending(t)
    return neverending(t) -- note: parentheses are used here because the runtime value of types is being passed in, rather than the static annotation of types
end
```
As a result, the current plan is to maintain a tightly scoped implementation, focusing on the simplest version of type functions to minimize the risk of developers accidentally starving themselves of features provided by the analysis, and gradually enhance capabilities through successive iterations.

We have considered implementing a user-configurable execution limit based on time or instruction count for reducing type functions where if a type function exceeds the limit, it fails to reduce. However, these methods are not consistently reliable for terminating type functions that take too long to execute. Time-based timeouts are dependent on CPU performance; programs that type check on fast CPUs may not type check on slower CPUs. Similarly, instruction-based timeouts are dependent on the compiler optimizations; programs that type check on one compiler version may not type check on versions without the same optimizations. We will be experimenting with various forms of limiting the execution of type functions and reserve the right to change how termination of type functions is managed.

<details><summary>List of illegal constructs in type functions</summary>

* `while` / `repeat` loops
* invoking other type functions / regular functions / lambdas
    * we will not (and probably never) allow type functions to call regular functions for the sake of maintaining discrete stages between runtime and analysis
* referring to locals / globals in the outer scope
* global functions: `getfenv`, `setfenv`, `pcall`, `xpcall`, `require`
* libraries: `coroutine`, `debug`, `string.gsub`

Note: we are aware that for loops can cause infinite runtime. For the time being, we will not be handling this case. In the event that a developer accidentally creates an infinitely long type function, autocomplete will timeout in their editor environments and running luau-analyze will not complete. They will need to fix the type function and restart their environment / analysis.

</details>

To give warnings, developers can use `print()` with custom warning messages. To fail reductions, developers can use `error()` with custom error messages. If nothing is returned by the type function, it will fail to reduce with the default message: "Failed to reduce \<Name\> type function with no return values".

To allow Luau developers to modify the runtime values of types in type functions, this RFC proposes introducing a new userdata called `typelib`. A `typelib` object is a runtime representation of all types within the program and provides a basic set of library methods that can be used to modify types. As such, under the hood, the `typelib` library will closely mimic the implementation of static types in Luau Analysis. Most importantly, they are *only accessible within type functions* and are *not a runtime type for other use cases than type functions*. 

<details><summary>typelib library (dropdown)</summary>

Methods under a different type heading (ex: `Singleton`) imply that the methods are only available for those types. At the implementation level, there is a check to make sure that the type-specific methods are being called on the correct types. For instance, `getindexer()` asserts that `istable()` is true.

#### typelib
All attributes of newly created typelib are initialized with empty tables / arrays and `typelib.nil`. For instance, `typelib.newtable()` initializes its properties with an empty table and index / index result type as `typelib.nil`.

| Instance Attributes | Type | Description |
| ------------- | ------------- | ------------- |
| `niltype` | `typelib` | an immutable runtime representation of the built-in type `nil` |
| `unknown` | `typelib` | an immutable runtime representation of the built-in type `unknown` |
| `never` | `typelib` | an immutable runtime representation of the built-in type `never` |
| `any` | `typelib` | an immutable runtime representation of the built-in type `any` |

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `issubtypeof(arg: typelib)` | `boolean` | returns true if self is syntactically a subtype or equal to arg in the type hierarchy |
| `__eq(arg: typelib)` | `boolean` | overrides the == operator to return true if self is syntactically equal to arg in the type hierarchy |

| Static Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getnegation(arg: typelib)` | `typelib` | returns an immutable runtime representation of the negation of the argument; the argument cannot be `istable()`, `ismetatable` or `isfunction()` |
| `getboolean()` | `typelib` | returns an immutable runtime representation of the built-in type `boolean` |
| `getnumber()` | `typelib` | returns an immutable runtime representation of the built-in type `number` |
| `getstring()` | `typelib` | returns an immutable runtime representation of the built-in type `string` |
| `getstringsingleton(arg: string)` | `typelib` | returns an immutable runtime representation of a string singleton type of the argument |
| `getbooleansingleton(arg: boolean)` | `typelib` | returns an immutable runtime representation of a boolean singleton type of the argument |
| `getunion(arg: {typelib})` | `typelib` | returns an immutable runtime representation of union type of its argument |
| `getintersection(arg: {typelib})` | `typelib` | returns an immutable runtime representation of intersection type of its argument |
| `newtable(props: {[typelib]: typelib}, indexer: {key: typelib, value: typelib}?)` | `typelib` | returns a mutable runtime representation of a `table` type |
| `newmetatable()` | `typelib` | returns a mutable runtime representation of a metatable represented as a special property of the `table` type |
| `newfunction()` | `typelib` | returns a mutable runtime representation of a `function` type |
| `isnil(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type `nil` |
| `isunknown(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type `unknown` |
| `isnever(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type `never` |
| `isany(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type `any` |
| `isnegation(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a `Negation` |
| `isboolean(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type`boolean` |
| `isnumber(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type `number` |
| `isstring(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the built-in type `string` |
| `isstringsingleton(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a string singleton |
| `isbooleansingleton(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a boolean singleton |
| `isunion(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the union type |
| `isintersection(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of the intersection type |
| `istable(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a `table` type |
| `ismetatable(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a metatable represented as a special property of the `table` type |
| `isfunction(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a `function` type |
| `isclass(arg: typelib)` | `boolean` | returns true if the argument is syntactically a runtime representation of a `class` type |

#### Negation

| Instance Methods | Type | Description |
| ------------- | ------------- | ------------- |
| `gettype()` | `typelib` | returns the runtime representation of the self's type being negated |

#### String

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getmetatable()` | `typelib` | returns the runtime representation of self's metatable |

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
| `setprop(key: typelib, value: typelib?)` | `nil` | adds / overrides (if same key exists) a key, value pair to self's table properties; if value is nil, removes the key, value pair from self's table properties; if the key does not exist and the value is nil, nothing happens |
| `getprop(key: typelib)` | `typelib?` | returns the value associated with the key from self's table properties if the key exists, else nil |
| `getprops()` | `{[typelib]: typelib}` | returns a table of self's table properties (e.g. `{["age"] = 20}` will return `{typelib.getstringsingleton("age") = typelib.getnumber()}`) |
| `setindexer(key: typelib, value: typelib)` | `nil` | sets self's indexer key type to the first argument and indexer value type to the second |
| `getindexer()` | `{key: typelib, value: typelib}?` | returns a table containing self's indexer key type and value type if they exist, else nil |
| `setmetatable(arg: typelib)` | `nil` | sets self's metatable to the argument; both self and the argument need to be `ismetatable()` |
| `getmetatable()` | `typelib?` | returns self's runtime representation of metatable if it exists, else nil; self needs to be `ismetatable()` |

#### Function

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setparameters(arg: {typelib} \| typelib)` | `nil` | sets self's parameter types to the argument, where an array implies a TypePack and the latter implies a Variadic |
| `getparameters()` | `{typelib} \| typelib?` | returns the runtime representation of self's parameter type if it exists, else nil |
| `setreturns(arg: {typelib} \| typelib)` | `nil` | sets self's return types to the argument, where an array implies a TypePack and the latter implies a Variadic |
| `getreturns()` | `{typelib} \| typelib?` | returns the runtime representation of self's return type if it exists, else nil |

#### Union

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomponents()` | `{typelib}` | returns an array of types that the self's union can represent. For instance, `string \| number` returns `{typelib.getstring(), typelib.getnumber()}` |

#### Intersection

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomponents()` | `{typelib}` | returns an array of types represented by self's intersection. For instance, `string & number` returns `{typelib.getstring(), typelib.getnumber()}` |

#### Class

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getprops()` | `{[typelib]: typelib}` | returns the runtime representation self's properties |
| `getparent()` | `typelib?` | returns the runtime representation of self's parent class if it exists, else nil |
| `getmetatable()` | `typelib?` | returns the runtime representation of self's metatable if it exists, else nil |
| `getindexer()` | `{key: typelib, value: typelib}?` | returns a table containing self's indexer key type and value type |

</details>

The reason for going with userdata instead using another representation or adding a new compile-time interpreter like many other language is that userdata provides a clean abstraction from the runtime representation and restricts developers to using a set of libraries controlled by the Luau team where we can easily manage changes and listen to feedbacks. Moreover, because Lua was designed to be easily embeddable with C++, we wanted to use this as an advantage for type functions since they only require using a small runtime variant of the VM, making the implementation be simpler than having to implement a new interpreter.

### Implementation

A `typelib` library will be implemented using the Lua API to interface between C++ and Luau and support the library methods, including type serialization for arguments and deserialization for return values. To implement type functions, a new AST node called `AstStatTypeFunction` will be introduced and created when parsing a type alias followed by the keyword "function." In the constraint generator, visiting this new AST node will generate a `TypeAliasExpansionConstraint` and in the constraint solver, reducing `TypeAliasExpansionConstraint` will generate a `ReduceConstraint`. To reduce `ReduceConstraints`, user-defined type functions will be integrated as built-in type functions in Luau where when being invoked, their arguments will be serialized into an instance of `typelib`. These functions will interact with the Luau VM to execute the function body in an established environment with only the specified libraries and constructs available. The return value of the type function will be deserialized and be the value that the `ReduceConstraint` reduces to.

## Drawback

Type functions are handled at the analysis time, while `typelib` is an implementation in the runtime. As a result, the proposed design causes the analysis time to be dependent on Luau's runtime to reduce user-defined type functions. This is generally discouraged as it is best to isolate the compile time, analysis time, and runtime from each other for the purpose of maintaining a clean separation of concerns, which helps minimize side effects and dependencies across different phases of the program execution and improves the modularity of the compiler. Overlaps between the analysis time and runtime can lead to code that is more complex and harder to manage, potentially increasing the risk of bugs and making the outcomes less predictable.

The build / analysis times will also be negatively impacted as reducing type functions takes variable amount of time based on the program. Developers will be able to write non-performant code that impacts their (and any of their depedent code's) analysis time. The larger the type function, the longer it will take to reduce it in the constraint solver.

## Alternatives

### `table` Runtime Representation

Currently, the runtime representation of types is a userdata called `typelib`. Another representation is to use the already-existing type in Luau `table`; instead of serializing types into `typelib`, we can serialize them into tables with predefined properties. For instance, the representation for a string singleton `"abc"` could be `{type = "stringSingleton", value = "abc"}`. So instead of writing:
```luau
type function isSingleton(t)
    if typelib.isstringsingleton(t) then
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

In some sense, this design could be considered "cleaner" than introducing an entirely new userdata, but it requires developers to have a deeper understanding of the type runtime representation. For example, under the proposed design, a new table type can be created using `typelib.newtable()`, while under the alternative design, tables must be declared with attributes: `{type = "table", props = {}, indexer = {}}`. This adds complexity to the developer experience and increases the chance of making syntax errors in their program.

### More Builtin Type Functions

Another alternative is to simply let go of this idea and create more built-in type functions that cover a wider range of common use cases. This approach clearly lacks the flexibility of fully user-defined type functions because type manipulation would be limited to the predefined set of type functions designed by the Luau team. Furthermore, continuously expanding the set of built-in type functions could lead to bloat and complexity within the core language, making it harder to maintain.

### Type Function Type Checking

In the future, we could investigate adding type checking to user defined type functions. In Haskell's type families, this is done with _kinds_ (aka types of types). Similarly, we could look into introducing kinds to Luau, but for the purpose of this RFC, this feature will be left for future works.