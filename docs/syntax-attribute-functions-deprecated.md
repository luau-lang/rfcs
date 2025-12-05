# Deprecated Attribute for Functions

**Status**: Implemented

## Summary

This RFC proposes a `@deprecated` attribute to annotate Luau functions for deprecation. Using these functions will result in a deprecation warning from the linter. The Luau LSP can indicate the deprecated status of these functions in its completion list by showing them in a visually distinct style.

## Motivation

Roblox already has numerous deprecated Luau APIs. LSP strikes out the names of these APIs in the autocompletion popup in Roblox Studio, their use issues deprecation warnings, and their deprecation status is reflected in Roblox's Creator Hub documentation. Unfortunately, Luau lacks support to leverage this tooling for deprecating custom creator-defined functions. Even for core Luau deprecated functions, `getfenv` and `setfenv`, the linter has hardcoded name comparisons to identify their uses and report deprecation messages. Deprecating new APIs requires ad-hoc modifications to the tooling which can only be performed by the language designers.

This RFC proposes the introduction of a `@deprecated` attribute to mark deprecated Luau APIs. This has multiple benefits:
1. It provides an explicit syntactic marker to identify deprecated functions in Luau code.
2. The LSP interface can display the deprecated API in a visually distinct style in the autocompletion popup.
3. The linter can issue a warning message when the deprecated function is used.

## Design

The `@deprecated` attribute can be used on named functions and properties. It does not apply recursively to the functions defined within the lexical scope of the attributed function. It can take up to two named string parameters. The parameters affect the warning message issued by the linter. The `reason` parameter gives the reason for deprecation and the `use` parameter gives the name of the function that should be used in lieu of the deprecated function. The following table shows the warning message issued by the different styles of this attribute when used on a function named `foo`.

| Style                                                 | Message                                                                |
| ----------------------------------------------------- | -----------------------------------------------------------------------|
| `@deprecated`                                         | `"Function 'foo' is deprecated."`                                      |
| `@[deprecated {reason = "my reason"}]`                | `"Function 'foo' is deprecated. <reason>"`                             |
| `@[deprecated {use = "bar()"}]`                       | `"Function 'foo' is deprecated, use '<use>' instead."`                 |
| `@[deprecated {reason = "my reason", use = "bar()"}]` | `"Function 'foo' is deprecated, use '<use>' instead. <reason>"`        |

If `foo` is a member of class `bar`, the warning messages above will start as `"Member 'bar.foo' is deprecated ...`.

Consider the following example. Function `printTable` is marked deprecated.

```lua
@[deprecated {
    use = "table.foreach(table, print)", 
    reason = "`printTable` forces tables to be printed in one style; `table.foreach` allows configurable printing.",
}]
local function printTable(t)
    for k,v in pairs(t) do
        print(k, v)
    end
end

printTable({"a", "b", "c", "d"})
```

The call to `printTable` causes the linter to issue the following warning message:

```
Function `printTable` is deprecated, use `table.foreach(table, print)` instead. `printTable` forces tables to be printed in one style; `table.foreach` allows configurable printing.
```

Table and class properties also support deprecated members. In the following example property `m` of table `t` is deprecated.

```lua
local t = {}

@deprecated
function t:m() end
```

## Related Work

Many mainstream languages provide a similar mechanism for deprecating language features. They differ in the kind of program elements that can be deprecated, the ability to convert deprecation warnings to errors, and the ability to specify the version which deprecated the API. 

1. **C++** since C++14 provides a `[[deprecated(msg)]]` attribute that can be applied to declaration of a class, a typedef-name, a variable, a nonstatic data member, a function, a namespace, an enumeration, an enumerator, or a template specialization. The compiler emits a deprecation message, `msg`, when the deprecated program element is used.

2. **C#** provides a `[ObsoleteAttribute(msg, error)]` which can be applied to all program elements except assemblies, modules, parameters, and return values. The `msg` string is emitted by the compiler when the deprecated program element is used in code. The `error` flag converts the warning to an error.

3. **Java** provides a `@Deprecated` annotation to deprecate a class, method, or field. The Java compiler issues warnings when these program elements are used. A detailed deprecation message can be supplied by a corresponding javadoc `@deprecated` tag.

4. **Rust** provides a `#[deprecated(since, note)]` attribute. `since` specifies a version number when the item was deprecated and `note` specifies a string that should be included in the deprecation message. The deprecated attribute can be applied to many program elements such as macros, modules, functions, enums, structs, traits, enum variants, struct fields, etc. When applied to an item containing other items, such as a module, all child items inherit the deprecation attribute.

5. **Scala** provides a `@deprecated(msg, version)` annotation for deprecating definitions. The annotation takes a message and a string specifying the first version since the definition has been deprecated.

6. **Kotlin** provides a `@Deprecated(message, replaceWith, level)` annotation. The `message` and `replaceWith` fields translate respectively to `msg` and `alt` fields of the `@deprecated` attribute proposed in this RFC. The `level` field can be `WARNING`: to report use of the API as a warning, `ERROR`: to report use of the API as an error, and `HIDDEN`: to make the API inaccessible from code.

7. **Dart** provides a `Deprecated(msg)` annotation that applies to libraries, top-level declarations (variables, getters, setters, functions, classes, mixins, extension and typedefs), class-level declarations (variables, getters, setters, methods, operators or constructors, whether static or not), named optional parameters and trailing optional positional parameters. This annotation applies transitively. Deprecating a library or a class deprecates all its members. Deprecating a variable deprecates its implicit getter and setter. Deprecating a superclass does not deprecate its subclass. Deprecation warning is issued when the code imports a deprecated library or any deprecated member.

8. **Python** decorators provide a mechanism for developers to roll their own deprecated decoration. A popular implementation is provided by the `deprecation` library which provides `@deprecation.deprecated(deprecated_in, removed_in, current_version, details)` decorator that updates the docstring of the deprecated method and issues warning when it is used. `deprecated_in` is the version at which the decorated method is considered deprecated, `removed_in` is the version when the decorated method will be removed, and `current_version` is the version information for the currently running code, and `details` string provides extra details added to the method docstring and warning.

The description above shows the full-form of the deprecation annotation. In most cases, it is valid to leave the message parameter in favor of a generic deprecation message. Compared to most of these languages, the design proposed for deprecating Luau APIs is simpler. 
1. Deprecation is not supported for non-function program elements because Luau only supports attributes on functions. 
2. There is no way to convert deprecation warnings to errors. 
3. There is no explicit attribute parameter for specifying version string. They can be accommodated in the `reason` parameter. If desired, a future RFC could introduce a `version` parameter to the `@deprecated` attribute.

## Drawbacks

Adding this attribute increases complexity of code. Once the attribute is released, we cannot deprecate it.

## Alternatives

The alternative would be to not provide this attribute; core Luau APIs will rely on custom enhancements to the Linter for deprecation but creators will have no way to deprecate their Luau functions.