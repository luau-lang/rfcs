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
type function rawget(tbl, key)

    for k, v in tbl:getprops() do
        if k == key then
            return v
        end
    end

    error("key not found!")
end

type Person = {
    name: string,
    age: number
}

type ty = rawget<Person, "name"> -- ty = string
```

Type functions operate on two stages: type analysis and runtime. When calling type functions at the type level (e.g. annotating a variable as a type function), angle brackets must be used, but when calling them at the runtime level (e.g. calling other type functions within type functions), parenthesis must be used. Declarations of type functions use parentheses because it defines the runtime operations on the runtime representation of types.

For the first iteration, the body of a type function will be sandboxed, and its scope will be limited, meaning it will be unable to refer statements defined in the outer scope, including other type functions. Additionally, type functions will be limited on what globals/libraries they could call. The list of available globals/libraries in type functions is:

- global functions: `assert`, `error`, `next`, `print`, `rawequal`, `select`, `tonumber`, `tostring`, `type`, `typeof`, `ipairs`, `pairs`, `unpack`
- math library
- table library
- bit32 library
- buffer library

There is also a problem of infinitely running type functions. For example, reducing this type function will halt analysis until the VM stack overflows:
```luau
type function neverending(t)
    while true do
        local a = 1
    end
end
```

For our initial iteration, we plan to implement a straightforward approach by enforcing a time limit on the type function VM. We recognize that this method has its limitations, as time-based timeouts can vary significantly based on CPU performance. Programs that type check efficiently on high-performance CPUs may not do so on slower ones. We plan to experiment with different strategies for limiting the execution of type functions and remain open to adjusting our approach based on insights from the community.

Any runtime errors that result from running the body of the type functions will be ported as an analysis error. This means that developers can intentionally fail type function reductions by using `error()` with custom error messages. Type functions expect to have exactly one return value of a `type` instance.

To allow Luau developers to modify the runtime values of types in type functions, this RFC proposes introducing a new userdata called `type` (for the purpose of clarity, `type` (in code block) refers to the userdata and type (the literal string) refers to their English definition). A `type` userdata is a runtime representation of all types within the program and provides a basic set of library methods that can be used to modify types. They are *only accessible within type functions* and are *not a runtime value/userdata/library anywhere else*.

Because the name clashes with the global function `type()`, the `type` userdata's `__call` metamethod will be set to the original `type()` function.

<details><summary>`type` library (dropdown)</summary>

Methods under a different heading (ex: Singleton) imply that the methods are only available for those types. All attributes of newly created `type` instances are initialized with empty tables / arrays and `nil`. Methods with optional arguments will by default set arguments to the leftmost attribute. For instance, `type.newtable(X, Y)` will set properties to be `X`, indexer to be `Y`, and metatable to be `nil`. Additionally, all arguments are passed by references. 

#### `type`

| Instance Attributes | Type | Description |
| ------------- | ------------- | ------------- |
| `niltype` | `type` | an immutable runtime representation of the built-in type `nil` |
| `unknown` | `type` | an immutable runtime representation of the built-in type `unknown` |
| `never` | `type` | an immutable runtime representation of the built-in type `never` |
| `any` | `type` | an immutable runtime representation of the built-in type `any` |
| `boolean` | `type` | an immutable runtime representation of the built-in type `boolean` |
| `number` | `type` | an immutable runtime representation of the built-in type `number` |
| `string` | `type` | an immutable runtime representation of the built-in type `string` |

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `__eq(arg: type)` | `boolean` | overrides the == operator to return true if self is syntactically equal to arg |
| `is(arg: string)` | `boolean` | returns true if self has the same tag as the argument. List of available tags: "nil", "unknown", "never", "any", "boolean", "number", "string", "singleton", "negation", "union", "intersection", "table", "function", "class"  |

| Static Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getnegation(arg: type)` | `type` | returns an immutable runtime representation of the argument as a negated type; the argument cannot be an instance of a table or a function |
| `getsingleton(arg: string \| boolean)` | `type` | returns an immutable runtime representation of the argument as a singleton type |
| `getunion(...: type)` | `type` | returns an immutable runtime representation of an union type of the arguments; requires the arguments size to be at least 2 |
| `getintersection(...: type)` | `type` | returns an immutable runtime representation of an intersection type of the arguments |
| `newtable(props: {[type]: type}?, indexer: {key: type, value: type}?, metatable: type?)` | `type` | returns a mutable runtime representation of a table type; the keys of the table's property must be a runtime representation of the singleton types; if provided the metatable parameter, this table becomes a metatable |
| `newfunction(parameters: {pack: {type}?, variadic: type?}, returns: {pack: {type}?, variadic: type?})` | `type` | returns a mutable runtime representation of a `function` type |
| `copy(arg: type)` | `type` | returns a deep copy of the argument |

#### Negation

| Instance Methods | Type | Description |
| ------------- | ------------- | ------------- |
| `gettype()` | `type` | returns the `type` being negated |

#### Singleton

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getvalue()` | `string \| boolean` | returns the string value of the  singleton |

#### Table

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setprop(key: type, value: type?)` | `nil` | adds / overrides (if same key exists) a key, value pair to the table's properties; if value is nil, removes the key, value pair; if the key does not exist and the value is nil, nothing happens |
| `getprop(key: type)` | `type?` | returns the value associated with the key from the table's properties; if the key does not exists, returns nil |
| `getprops()` | `{[type]: type}` | returns a table of the table's properties |
| `setindexer(key: type, value: type)` | `nil` | sets the table's indexer with key type as the first argument and value type as the second argument |
| `getindexer()` | `{key: type, value: type}?` | returns the table's indexer as a table; if the indexer does not exist, returns nil |
| `setmetatable(arg: type)` | `nil` | sets the table's metatable to the argument |
| `getmetatable()` | `type?` | returns the table's metatable; if the metatable does not exist, returns nil |

#### Function

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setparameters(pack: {type}?, variadic: type?)` | `nil` | sets the function's parameter types to the arguments |
| `getparameters()` | `{pack: {type}?, variadic: type?}` | returns the function's parameter types as a table |
| `setreturns(pack: {type}?, variadic: type?)` | `nil` | sets the function's return types to the arguments |
| `getreturns()` | `{pack: {type}?, variadic: type?}` | returns the function's return types as a table |

#### Union

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomponents()` | `{type}` | returns an array of `types` being unioned |

#### Intersection

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomponents()` | `{type}` | returns an array of `types` being intersected |

#### Class

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getprops()` | `{[type]: type}` | returns the class's properties |
| `getparent()` | `type?` | returns the class's parent class; if the parent class does not exist, returns nil |
| `getmetatable()` | `type?` | returns the class's metatable; if the metatable does not exists, returns nil |
| `getindexer()` | `{key: type, value: type}?` | returns the class's indexer as a table; if the indexer does not exist, returns nil |

</details>

## Drawback

A drawback to the proposed design is that it makes analysis somewhat be dependent on runtime because type functions are handled during analysis and `type` is an implementation in the runtime. This is generally discouraged for the purpose of maintaining a clear separation of concerns to minimize side effects across different phases of the program execution. 

Enforcing time limits on type functions is quite problematic for the same reasons mentioned previously. 

## Alternatives

### Table Runtime Representation

Instead of serializing types into `type`, we can serialize them into tables with predefined properties. For instance, the representation for a string singleton `"abc"` could be `{type = "singleton", value = "abc"}`. So instead of writing:
```luau
type function issingleton(t)
    if t:is("singleton") then
        return t
    end
end
```
developers could write:
```luau
type function issingleton(t)
    if t.type == "singleton" then
        return t
    end
end
```

In some sense, this design could be considered better to write than using userdata in an object-oriented manner. However, using what's already built into the language makes it harder for the Luau team to apply restrictions to type functions. For instance, requiring all union types to have at least 2 elements is something that is hard to enforce under this design because developers could just write `{type = "union", components = {{type = "string"}}}`. Finally, this design makes the creation of new run time instances messy and prone to errors (imagine if your type function didn't work because you forgot to close a bracket!)

### Compile-time Interpreter

A lot of languages have their own compile-time interpreter to reduce things like type functions. They have their benefits such as allowing us to completely isolate analysis and runtime and allow developers to write type functions using syntax that would be illegal in standard Luau (which would involve having to create a language inside Luau). Other than the fact it requires higher complexity and maintenance, the design also goes against one of our goals for type functions. We wanted the experience of writing type functions to feel familiar and the exact same as writing normal Luau code. And because Luau is easily embeddable, it would suffice to use the existing VM and add a new userdata to achieve our goals.

### More Builtin Type Functions

Another alternative is to create more built-in type functions that cover a wider range of programming patterns. This approach clearly lacks the flexibility of fully user-defined type functions because type manipulation would still be limited to the predefined set of type functions designed by the Luau team. Furthermore, continuously expanding the set of built-in type functions leads to bloat and complexity within the language, making it harder to maintain.

## Future Works

In the future, we could investigate adding type checking to user defined type functions. In Haskell's type families, this is done with _kinds_ (aka types of types). Similarly, we could look into introducing kinds to Luau, but for the purpose of this RFC.

Other future works include expanding the set of libraries to provide support for more paradigms such as supporting generic types or named parameters for function types. We would additionally want to expand the scope of type functions to be able to refer other type functions and variables and ultimately allow developers to write type functions as freely as possible.