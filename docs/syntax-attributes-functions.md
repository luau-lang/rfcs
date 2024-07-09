# Attributes (for Functions)

**Status**: Implemented

## Summary

Add a syntax for attributes to function declarations in the style of `@name` that can be placed before their declarations to adjust the behavior of the compiler, analyzer, and runtime.

These attributes would be valid in both the normal syntax for declaring functions and the syntax for anonymous functions, with or without locally scoped names.

Due to attributes being tied to the language's behavior, this proposal does not allow for them to be implemented by users. A mechanism to do so would be either very complex or wildly unsafe and is thus not considered by this proposal.

## Motivation

It's desirable to have control over various parts of Luau. Consider three potential attributes:

- `@native`, to indicate that a function should be compiled using native codegen
- `@inline`, to indicate that a function should be inlined where called
- `@deprecated`, to indicate that a function should be marked as deprecated by the analyzer

Right now, `@native` has an equivalent directive at the top of scripts in the form of `--!native`, albeit for the entire chunk being processed. In the future however, this may be guided by heuristics on a per-function basis to determine if doing so is 'profitable'. While cost models of this nature can be extremely well tuned, they tend to err on the side of caution so are not always correct. Without some way to indicate that a specific function should be natively compiled, users are at the whim of a cost model that might change unexpectedly and negatively impact performance.

Functions are sometimes inlined too, based on a heuristic cost model. There is currently no way to tell one way or another whether the model has determined if a function is a good candidate for inlining without inspecting bytecode however, which is often not possible (as in the case of Roblox). An `@inline` attribute would allow a user to heavily suggest the compiler inline a function, trusting that they know the potential downsides and have decided it's worth it.

There is currently no way to discourage the use of functions in user code. Luau has existing lints for this named `DeprecatedGlobal` and `DeprecatedApi`, which flag the use of deprecated globals and API members in code, but they do not have a user-facing API. A `@deprecated` attribute could be used to let users indicate that a piece of code is deprecated, which is an obvious boost to the user experience, and would do so without introducing any new syntax beyond that of attributes.

While all of these attributes could be supported on a case-by-case basis, a unified syntax for them would simplify the language's grammar and ease their implementation as well as any other features of this sort in the future.

## Design

The following syntax is proposed for attributes:

```ebnf
attribute = '@' NAME
```

Attributes would attach to the function as a form of metadata to adjust the behavior of Luau, whether it be in the compiler, analyzer, or otherwise.

Attributes would be valid before function declarations, both anonymous and named:
```luau
@example
function foo()
end

foo = @example function() end
```

Whether the name of a function is a local variable would have no impact on the use of attributes.

When declaring a function as a member of a table, any attributes should be for the function:
```luau
@example -- @example applies to `bar`, not `foo`
function foo:bar()
end
```

This is consistent with other uses, as it applies the attribute to what is being declared.

Multiple attributes are supported inline, so that a function with multiple attributes does not have several lines of boilerplate:
```luau
@attribute1 @attribute2 @attribute3 @attribute4 @attribute5
local function example()
end
```

Attributes with parameters are not included in this proposal because it's unclear what the syntax and limitations should be. However, the syntax proposed would not conflict with a future addition of parameters provided they used proper delimiters.

User-defined attributes are not proposed because they are considered incompatible with Luau's goals of being a simple but safe-to-embed language. Since the compiler would at least in part be controlled by attributes, allowing them to be implemented by users would be either prohibitively complex or unsafe.

## Drawbacks

Attributes are a potentially very broad-reaching syntax extension. While there are tangible benefits to users, it may not be worth the complexity they bring to the table. Though it is not currently proposed, attributes may eventually become attached to other language constructs like loops and variable declarations, which would heighten this problem.

Allowing manual control over language features like inlining and native codegen may be a bad idea, since it means that obvious flaws in the heuristics used for doing this normally may go unreported by power users that care about performance. It may be better to not include them simply for the sake of refining the cost models used for these things.

Adding yet another symbol (`@`) contributes to the complexity of the language, both in the literal implementation and to read, write, and learn.

## Alternatives

Most of the features attributes can support could be implemented by refining the type system (in the case of `@deprecated` and attributes of that nature) and improving heuristics for the compiler (in the case of `@native` and `@inline`). This is very much an option, but in the case of the type system is probably not sustainable since every new feature adds more complexity.

Attributes are distinct from decorators in that they are compiler defined and thus cannot be defined during runtime. An alternative use for this syntax may be user-defined decorators as in JavaScript. If the language provided special built-in decorators, this could serve a very similar purpose. This is not rejected on merit, it's simply that attributes have a more obvious path to implementation.

A more complete syntax for attributes with parameters could be proposed instead, but it would require quite a bit more consideration than this proposal. Since it's not currently clear what the needs of attribute parameters are, any proposed syntax would be speculative and might be overly or underly ambitious. Many of the 'obvious' attributes don't require parameters to be useful, so in the current state of this proposal is designed to be straightforward to facilitate their implementation.

Other potential syntaxes that were considered include a Rust-style syntax (`#[attribute]`) and comment directives (`--!attribute`). These were rejected because:

- `#` is used in other places in the language, where it means something totally unrelated

- Comments being used to control language features and flow means that tooling and users must care about them, which is antithetical to how comments are traditionally used

- Attribute-as-comments would naturally conflict with the language in a lot of weird ways and would just cause a lot of problems:
```luau
local f = --!native function() end
print(f)
```

As proposed, attributes would only be applicable on functions at first. They could instead be added to the entire language at once, which would facilitate widespread adoption immediately. However, that is a lot more work, and most of the potential uses for attributes are on functions, so it makes sense to begin with them.
