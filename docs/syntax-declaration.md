# Declaration syntax

## Summary

Formalizing the luau declaration syntax used in the luau parser.

## Motivation

RIght now the declaration syntax used by the luau parser is undocumented and an unstable implementation detail. By defining the syntax, it allows for embedders of luau to have typechecking for their apis without having to resort to empty modules. This should allow tools like luau-lsp to be able to provide easier ways to use these declarations.

## Design

Declaration syntax should only be usable in definition files. Definition files

The declaration syntax mirrors the current syntax used by the luau parser. All declarations are prefixed with `declare`, and global variables, global functions, and classes are able to be declared.

## Global variables
After `declare`, an identifier representing the name of the global variable is expected, followed by a colon and a type.
For example:

```luau
declare myglobal: number
declare anotherglobal: {
    add: (number, number) -> number,
    sub: (number, number) -> number,
}
```

## Global functions
Global functions can have the attributes which comes before declare. After `declare`, the `function` keyword is expected, followed by an identifier representing the name of the global function, optional generic type list, binding list, and finally. The body of the function is omitted.

Every parameter must be annotated with a type and if there is a vararg, it also must be annotated with a type. The return type is optional, but if it is not specified, the function will be treated as returning `nil`.

For example:

```luau
declare function warn<T...>(...: T...)
declare function wait(seconds: number?): (number, number)
declare function delay<T...>(delayTime: number?, callback: (T...) -> ())
```

## Classes
After `declare`, `class` is expected, followed by an identifier representing the name of the class. If the class inherits from another class, `extends` is expected, followed by the name of the class to inherit from.

Class properties and methods are now expected.

To declare a property, it can be specified with `["identifier"]`, `[ [[identifier]] ]`, or `identifier` followed by a colon and a type.

To declare an indexer, a type is used within `[` and `]` followed by a colon and a type. More than one class indexer is invalid.

To declare a method, the `function` keyword is expected followed by an identifier for the function name, binding list with varargs allowed, and an optional return type preceded by a colon.

There must always be at least one parameter in the binding list, and the first parameter in the binding list must be called `self` and have no type annotation. The rest of the parameters must be annotated with a type, and if there is a vararg, it also must be annotated with a type.

The class is closed with the `end` keyword.

Example:
```luau
declare class MyClass extends BaseClass
    -- properties
    ["Property1"]: number,
    [ [[Propery2]] ]: string,
    -- note this is not a method
    Property3: (number, number) -> number,
    -- indexer
    [number | string]: string,
    -- methods
    function Method1(self, a: number, b: string): (number, string)
    function Clone(self): MyClass
    function Empty(self)
end
```

More examples can found in [luau-lsp's globalTypes.d.luau](https://github.com/JohnnyMorganz/luau-lsp/blob/main/scripts/globalTypes.d.luau) and [luau's parser tests](https://github.com/luau-lang/luau/blob/master/tests/Parser.test.cpp#L1908)

## Drawbacks

A drawback would be that this is an unstable implementation detail, which allows for the parser to freely change the syntax without worry for backwards compatibility. By formalizing the syntax, it would mean more consideration would be needed when changing how declarations are parsed in the future.

## Alternatives

THe alternative to this is to leave this as an implementation detail. However this makes it inconvenient for embedders of luau to use the declaration syntax.