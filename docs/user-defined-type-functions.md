# User-Defined Type Functions

**Status**: Implemented

## Summary

The new Luau type inference engine (or type solver, for short) adds support for a system of built-in type functions that compute result types from provided argument types. This system is already being used to power support for overloadable operators, and the broad design philosophy is to include built-in type functions that correspond to built-in runtime-level operations in the language. To further enhance the expressivity of the type system, this RFC extends that system with a proposed design for user-defined type functions, allowing developers to define their own type functions in ordinary Luau code.

## Motivation

The primary motivation for introducing user-defined type functions is to increase the expressiveness of the type system and empower developers to create more type-safe abstractions in their libraries. Current limitations of the type system prevent programmers from constructing types in relational ways, e.g. producing a new table where the types of every property have changed in some predictable way (like being made optional). These limitations can prevent developers from writing precisely-typed abstracts in their libraries, leading to less type-safe abstractions that rely on the use of `any` casting or less-than-desirable structural changes to their designed APIs. By adding support for user-defined type functions, we aim to:
- Enable more precise type definitions that can capture complex relations on types
- Facilitate the creation of type-safe, reusable and composable libraries
- Enhance Luau's support for type-level programming

The expected outcome is a more versatile type system that can adapt to a wider range of programming patterns, improving the Luau developer experience.

## Design

Type functions will be defined with the following syntax:
```luau
type function f(...)
    -- implementation of the type function
end
```

The bodies of these functions will consist of fundamentally entirely ordinary Luau code that can interact with types as runtime values. For instance, the built-in `rawget` type function could be written as:
```luau
type function rawget(tbl, key)
    if not tbl:is("table") then
        error("first parameter must be a table type!")
    end

    for k, v in tbl:properties() do
        if k == key then
            if v.read ~= v.write then
                error("mismatched read/write types found for the property")
            end

            return v.read
        end
    end

    error("key not found!")
end

type Person = {
    name: string,
    age: number
}

type ty = rawget<Person, "name"> -- resolves to `ty` defined as `string`
```

This function takes `tbl`, a table type, and `key`, the type of a property key for the table (typically a string singleton type), and computes the corresponding value type for the property `key` in the table `tbl`. If `tbl` is not a table type, or `key` is not found in `tbl`, then the function raises an error.

The scoping and shadowing rules of user-defined type functions will be made to match the existing rules for type aliases (i.e. `type Array<T> = {T}`) which are, as it turns out, a sort of total form of type functions in the first place. This means that they are order-independent, and can refer to one another, as well as to type aliases. We do not intend to introduce an additional set of scoping rules, so ensuring they work the same as type aliases already do is a requirement for the design.

### The type runtime

To understand how type functions behave, we have to consider a split between stages: type analysis time and runtime. The full types of Luau's type system currently exist only during type analysis, and never during runtime. This RFC proposes changing Luau to include an additional stage, type runtime, that is part of the overall type analysis. During type analysis, when we encounter a user-defined type function, e.g. `rawget<...>` in the example above, we will serialize the type parameters of that call into a form that Luau can manipulate, and then execute the body of the type function in a Luau VM (this stage would be the "type runtime"), and reify its results (returning a value, erroring, etc.) back into type analysis. This means that the type solver will need to maintain a VM instance for evaluating user-defined type functions, and that each reference to a type function corresponds to evaluating a function call in that VM.

The key to making this type runtime work is the introduction of a `type` userdata that Luau's types can be serialized into. For clarity in this RFC, `type` (in code block) always refers to the userdata and type (the unadorned string) refers to the ordinary meaning of the word in the context of Luau (i.e.  the static type of a binding). An instance of the `type` userdata is a type runtime representation of a type within the program, and it provides a set of API calls that can be used to inspect and manipulate the type. As part of the _type runtime_, they are *only accessible within type functions* and are *not available during ordinary function runtimes*. Each type function can take an arbitrary number of `type` arguments, and can return exactly one `type` as a successful result during evaluation. Returning additional results will signal an error to the user for now, but a future RFC may revisit this to support user-defined type pack functions as well. We are currently excluding them since the scope of this RFC is already very large as-is, and even the built-in type function system has yet to find a compelling use for type pack functions.

Because the name `type` clashes with the global function `type(...)`, unlike `string` (whose library is named `string`), the name of the library for constructing `type`s will be `types`. So, a programmer will be able to access the boolean type by writing `types.boolean` or the number type by writing `types.number`. Evaluating `typeof(...)` on a `type` userdata will give the string `"type"`, and any possible extensions to support typechecking will likely name the type of types `type` as well. We considered an alternative where we used the name `type` and set the `__call` metamethod will to the original `type(...)` function. This would allow us to gracefully preserve the behavior of the built-in `type` function if it is necessary for developers, while still choosing the best possible name for the datatype itself, but it would also require the type runtime to provide a duplicate implementation of the `type(...)` function which would have to be maintained separately. We also considered alternative names like `luautype` `typelib` or `ltype`, but they seem to feel broadly worse when used in speech and writing, and have a similar disadvantage of being dissimilar from the existing library names in Luau.

Beyond `type`s, type functions will also reify any runtime errors that result from running their body into a type analysis error to report to the user. This means that developers can use the `error` and `assert` global functions to signal errors to consumers of their type function, and can provide detailed error messages that explain what went wrong. It also means that any "ordinary" runtime errors that arise from bad function calls or stack overflows will also become a user-facing error at the point that that type function is called.

### A note about syntax

Given this realization that type functions are, in fact, functions and their uses are function _calls_, one may wonder a couple things about the syntax that are helpful to clarify here. When programming types, all type functions (whether a user-defined type function, a type alias like `type Array<T> = {T}`, or a built-in type function like `add<T, U>`) are written with their parameters in angle brackets. At a call site, like `add<number, number>` or `rawget<Person, "name">` from above, the angle brackets indicate that the arguments to that call are _types_. By contrast, ordinary Luau functions are defined using parentheses for their parameters and function calls are similarly written with parentheses, e.g. `add(5, 6)`. For user-defined type functions, the arguments are types _serialized as runtime values for a Luau VM_, and so when we _declare_ a type function, we similarly use parentheses since they will be evaluated as functions in the _type runtime_.

### The environment of type functions

The type functions will run in a sandboxed VM instance with a limited scope of available libraries and globals. They will not be able to refer to statements defined in the scripts outer scope, nor to any runtime functions declared in the script. For the first iteration of this feature, we aim to be reasonably conservative in what we include to support writing type functions. The reason for this is that it is broadly much more feasible to _add_ new facilities to the language, than it is to take away features that are already in use. At the same time, we're attempting to balance this to ensure that programming type functions still feels like programming ordinary Luau code, so we wish to avoid having lots of "gotchas" or very unexpected exclusions. As such, we propose that the environment for type functions include only the following:

- `assert`, `error`, and `print`, which will be used by type functions to provide error feedback to consumers of a type function
- `next`, `ipairs`, `pairs`, `select`, and `unpack`, which support basic iteration and interaction with tables and multiple return
- `getmetatable`, `setmetable`, `rawget`, `rawset`, `rawlen`, `rawequal`, `tonumber`, `tostring`, `type`, and `typeof`, which are builtin functions that are essentially operators in Luau
- the `math` library, where we note that `math.randomseed` will be reset automatically on type function entry
- the `table` library
- the `string` library
- the `bit32` library
- the `utf8` library
- the `buffer` library

This list can be expanded in future RFCs if we learn of compelling usecases for additional libraries in type function contexts.

### The halting problem

In general, we'd like to be able to guarantee that type analysis will always terminate on all programs. Unfortunately, type inference for Luau's type system is already, in general, undecidable, and there are known edgecases where the type inference can be spun into exponential blowup or worse. Adding type functions seems like another space where this problem could be made worse, and so one might wish to try to detect when a type function will not terminate and error instead. Unfortunately, by [Rice's theorem](https://en.wikipedia.org/wiki/Rice%27s_theorem), all non-trivial semantic properties of programs are undecidable.

We could attempt to get around this either by heavily limiting the language in type functions to avoid Turing-completeness or by introducing heuristics to try to detect infinite loops while the type function is running. We propose doing neither. In the former case, the restrictions to the language would be significant, heavily limiting the use of while loops, function calls, etc. This would mean that learning to program type functions would feel _very_ different from learning to program Luau in general. In the latter case, the implementation work is considerable, and it's not clear what kind of impact it might have on the VM itself. 

Instead, this RFC proposes that we accept that type functions, like the rest of Luau's type system, are already not guaranteed to terminate in general, and we should employ the same mitigations that we've already implemented for addressing that problem --- namely, Luau's analysis supports global limits that include an overall runtime limit on analysis and a user-requestable (really, embedder-requestable) cancellation system for analysis. This means that editors, language services, and other software embedding Luau's type analysis will continue to be able to use one simple, unified system for imposing limits on analysis. In an editor tooling context, we expect that it might be useful for us to grow the feedback provided by the cancellation to support editors providing more detailed feedback to the users. It's not hard to imagine a developer experience where your editor informs you that a particular type function call in a particular spot in your program is taking a long time to evaluate. This would actually be a better experience than what is available today when builtin pieces of Luau's type system are unable to complete in an acceptable amount of time, and we are unable to give any actionable feedback to the user. Extending such a system to support identifying problem areas in built-in components of the type system is likely a much harder effort that is out of scope for this RFC.

### `types` API Reference

This section details the initial programming interface we propose for `type`s when they are reflected into type function bodies. Each section is separated by headers (e.g. Singleton) and describe the methods available to that category of type. The "`type` Instance" section, describes the elements of the interface common to every category of type. All properties of newly-created `type`s are initialized with empty tables / arrays and `nil`. All arguments are passed by references. 

### `types` Library

| Library Properties | Type | Description |
| ------------- | ------------- | ------------- |
| `unknown` | `type` | an immutable instance of the built-in type `unknown` |
| `never` | `type` | an immutable instance of the built-in type `never` |
| `any` | `type` | an immutable instance of the built-in type `any` |
| `boolean` | `type` | an immutable instance of the built-in type `boolean` |
| `number` | `type` | an immutable instance of the built-in type `number` |
| `string` | `type` | an immutable instance of the built-in type `string` |

| Library Functions | Return Type | Description |
| ------------- | ------------- | ------------- |
| `singleton(arg: string \| boolean \| nil)` | `type` | returns an immutable instance of the argument as a singleton type |
| `negationof(arg: type)` | `type` | returns an immutable instance of the parameter as a negated type; the parameter cannot be a table or function type |
| `unionof(...: type)` | `type` | returns an immutable instance of an union type of the arguments; requires at least 2 parameters to be provided |
| `intersectionof(...: type)` | `type` | returns an immutable instance of an intersection type of the arguments; requires at least 2 parameters to be provided |
| `newtable(props: {[type]: type \| { read: type?, write: type? } }?, indexer: { index: type, readresult: type, writeresult: type }?, metatable: type?)` | `type` | returns a mutable instance of a table type; the keys of the table's property must be a singleton type; sets the table's metatable if one is provided |
| `newfunction(parameters: { head: {type}?, tail: type? }, returns: { head: {type}?, tail: type? })` | `type` | returns a mutable instance of a `function` type |
| `copy(arg: type)` | `type` | returns a deep copy of the given `type` |

### `type` Instance

| Instance Properties | Type | Description |
| ------------- | ------------- | ------------- |
| `tag` | `"nil" \| "unknown" \| "never" \| "any" \| "boolean" \| "number" \| "string" \| "singleton" \| "negation" \| "union" \| "intersection" \| "table" \| "function" \| "class"` | an immutable property holding this type's tag |

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `__eq(arg: type)` | `boolean` | overrides the == operator to return true if self is syntactically equal to arg, note that semantically equivalent types like `true \| false` and `boolean` will _not_ compare equal |
| `is(arg: string)` | `boolean` | returns true if self has the same tag as the argument. List of available tags: "nil", "unknown", "never", "any", "boolean", "number", "string", "singleton", "negation", "union", "intersection", "table", "function", "class"  |

Depending on the particular `tag`, a `type` instance can have additional properties and methods available as described below.

#### Negation `type` instance

| Instance Methods | Type | Description |
| ------------- | ------------- | ------------- |
| `inner()` | `type` | returns the `type` being negated |

#### Singleton `type` instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `value()` | `string \| boolean \| nil` | returns the actual value of the singleton, i.e. a specific string, boolean, or `nil` |

#### Table `type` instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setproperty(key: type, value: type?)` | `()` | adds / overrides (if same key exists) a key, value pair to the table's properties; if value is nil, removes the key, value pair; if the key does not exist and the value is nil, nothing happens; equivalent to calling `setreadproperty` and `setwriteproperty` with the same parameters |
| `setreadproperty(key: type, value: type?)` | `()` | adds / overrides (if same key exists) a key, value pair to the table's properties; if value is nil, removes the key, value pair; if the key does not exist and the value is nil, nothing happens |
| `setwriteproperty(key: type, value: type?)` | `()` | adds / overrides (if same key exists) a key, value pair to the table's properties; if value is nil, removes the key, value pair; if the key does not exist and the value is nil, nothing happens |
| `readproperty(key: type)` | `type?` | returns the type used for _reading_ values to this property in the table; if the key does not exist, returns nil |
| `writeproperty(key: type)` | `type?` | returns the type used for _writing_ values to this property in the table; if the key does not exist, returns nil |
| `properties()` | `{[type]: { read: type?, write: type? } }` | returns a table mapping property keys to a table with their respective read and write types |
| `setindexer(index: type, result: type)` | `()` | sets the table's indexer with the index type as the first parameter, and the result as the second parameter; equivalent to calling `setreadindexer` and `setwriteindexer` with the same parameters |
| `setreadindexer(index: type, result: type)` | `()` | sets the table's indexer with the index type as the first parameter, and the resulting read type as the second parameter |
| `setwriteindexer(index: type, result: type)` | `()` | sets the table's indexer with the index type as the first parameter, and the resulting write type as the second parameter |
| `indexer()` | `{ index: type, readresult: type, writeresult: type }?` | returns the table's indexer as a table; if the indexer does not exist, returns nil |
| `readindexer()` | `{ index: type, result: type }?` | returns the table's indexer as a table using the read type of the result; if the indexer does not exist, returns nil |
| `writeindexer()` | `{ index: type, result: type }?` | returns the table's indexer as a table using the write type of the result; if the indexer does not exist, returns nil |
| `setmetatable(arg: type)` | `()` | sets the table's metatable to the argument |
| `metatable()` | `type?` | returns the table's metatable; if the metatable does not exist, returns nil |

#### Function `type` instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `setparameters(head: {type}?, tail: type?)` | `()` | sets the function's parameter types to the given arguments |
| `parameters()` | `{ head: {type}?, tail: type? }` | returns the function's parameter list as an array of ordered parameter types and optionally a variadic tail |
| `setreturns(head: {type}?, tail: type?)` | `()` | sets the function's return types to the given arguments |
| `returns()` | `{ head: {type}?, tail: type? }` | returns the function's return types as an array of types and optionally a variadic tail |

#### Union `type` instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `components()` | `{type}` | returns an array of `types` being unioned |

#### Intersection `type` instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `components()` | `{type}` | returns an array of `types` being intersected |

#### Class `type` instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `properties()` | `{[type]: { read: type, write: type } }` | returns a table mapping class' property keys to a table with their respective read and write types  |
| `readparent()` | `type?` | returns the read type of class' parent class; if the parent class does not exist, returns nil |
| `writeparent()` | `type?` | returns the write type class' parent class; if the parent class does not exist, returns nil |
| `metatable()` | `type?` | returns the class' metatable; if the metatable does not exists, returns nil |
| `indexer()` | `{ index: type, readresult: type, writeresult: type }?` | returns the class' indexer as a table; if the indexer does not exist, returns nil |
| `readindexer()` | `{ index: type, result: type }?` | returns the class' indexer as a table using the read type of the result; if the indexer does not exist, returns nil |
| `writeindexer()` | `{ index: type, result: type }?` | returns the class' indexer as a table using the write type of the result; if the indexer does not exist, returns nil |

## Drawback

The main drawback to the proposed design, particularly the use of the standard Luau VM for execution, is that it makes analysis explicitly depend on the Luau runtime in order to evaluate type functions. This has been historically discouraged for the purpose of maintaining a clear separation of concerns for different parts of Luau's implementation and to minimize side effects across different phases of the program execution. We believe this concern is mitigated since we still retaining a separation between the type runtime and its evaluation of type functions versus the overall runtime and its evaluation of ordinary functions. Further, the ability to use the same VM for both ensures that the runtime semantics of code in type function bodies is _always_ consistent with the runtime semantics of ordinary Luau code. This is not the case for some alternatives, like implementing a separate interpreter.

## Alternatives

### Table Runtime Representation

Instead of serializing types into `type`, we can serialize them into tables with a predefined structure. For instance, the representation for a string singleton `"abc"` could be `{type = "singleton", value = "abc"}`. So, instead of writing:
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

This is a reasonable design in its own right, and was used in the initial prototype of user-defined type functions because of its relative ease of implementation. We decided against continuing with it, however, because it makes it more difficult to apply basic well-formedness restrictions to the interface of type functions. For instance, we want all union types to have at least two elements to them (i.e. to be a non-trivial union), which would be harder to enforce with the table design since developers could just write `{type = "union", components = {{type = "string"}}}`. More generally, this design makes the creation of new types at runtime messier and more prone to errors since there's a greater surface for minor typos and misplaced braces to impact the program.

### Compile-Time Interpreter

Many languages implement an additional interpreter for evaluating expressions at compile-time. This approach has its benefits, such as allowing us to completely isolate analysis and runtime and allowing developers to write type functions in a syntax that potentially deviates from Luau's runtime syntax (e.g. supporting constructing intersection and union types using `&` and `|`). This has two main downsides: (1) the implementation required is higher complexity and carries a greater maintenance burden, and (2) the design against one of the stated goals for type functions. We want the experience of writing type functions to feel intrinsically familiar to developers and for the semantics to be the same as ordinary Luau code. Since Luau is already designed around being embeddable, we prefer to use the existing VM and add a new userdata instead.

### More Built-In Type Functions

Another alternative is to create more built-in type functions that cover a wider range of programming patterns. This approach clearly lacks the flexibility of fully user-defined type functions because type manipulation would still be limited to the predefined set of type functions designed by the Luau team. Furthermore, continuously expanding the set of built-in type functions leads to bloat and complexity within the language, making it harder to keep in your head, harder to maintain, and ultimately running against Luau's core philosophy of simple, general-purpose primitives and a small standard library.

## Future Work

In the future, we may consider adding type checking for user-defined type functions. In other advanced type systems, this is typically called _kind checking_ (where a _kind_ is what we'd call the type of a type). So, for instance, the type `number` has the kind `type`, but a type alias like `type Array<T> = {T}` has the kind `(T: type) -> type`, i.e. `Array` is a type function that when applied to a type `T` gives you a type. However, there's additional complexity in building this out in a way where we can reuse the existing type system machinery to implement typing. Since we expect type functions to largely be fairly small and the developer experience is inherently _very_ interactive since they will execute live in your IDE as you write code, we believe that working with dynamically-typed type functions will feel very pleasant, and so taking on the additional complexity is not yet justified.

Other areas of future work may include expanding the set of libraries available in type function environments to enable more expressive type functions, such as those manipulating generic types or named parameters for function types. We also want to investigate making type aliases (which are themselves a sort of _total_ type function) available as part of the type function environment to make type function programming feel even more like conventional Luau programming.
