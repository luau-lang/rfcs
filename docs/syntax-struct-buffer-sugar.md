# Struct-buffer sugar

## Summary

This proposal adds typed `struct` syntax as pure compiler sugar over Luau buffers. Struct declarations describe how a typed buffer should be read from and written to, but do not introduce a new runtime type, VM behavior, or buffer metadata. At runtime, struct values are ordinary `buffer`s. The compiler only allows struct field syntax in contexts where the buffer's struct type is known.

## Motivation

Luau buffers already provide the required runtime representation for compact structured data. However, writing buffer logic manually is verbose and error-prone, especially when repeatedly encoding and decoding the same layout.

This proposal does not add a new feature at runtime. Instead, it gives the compiler a way to understand a buffer as a typed structure in specific contexts, allowing high-level field syntax to compile into ordinary buffer operations. The expected result is less boilerplate and clearer intent, while preserving the exact same runtime behavior and representation.

## Design

### Type declaration

```luau
struct type Cat = {
	Name: U8_String;
	Age: U8;
}
```
This defines compile-time information only.

It tells the compiler how fields of a Cat are serialized into a buffer. It does not create a new runtime object kind.

### Construction
```luau
local cat: Cat = struct {
	Name = "Orange";
	Age = 5;
}
```
This compiles to ordinary buffer code:
```luau
local _tmp_name = "Orange"
local _tmp_name_len = #_tmp_name

local cat = buffer.create(1 + _tmp_name_len + 1)
buffer.writeu8(cat, 0, _tmp_name_len)
buffer.writestring(cat, 1, _tmp_name)
buffer.writeu8(cat, 1 + _tmp_name_len, 5)
```
### Field access
When the compiler knows a value is Cat, field syntax is allowed:
```luau
print(cat.Age)
cat.Age = 7
```
This compiles to ordinary buffer operations using the known struct serialization logic.

For example:
```luau
print(cat.Age)
```
compiles approximately to:
```luau
local _nameLen = buffer.readu8(cat, 0)
print(buffer.readu8(cat, 1 + _nameLen))
```
### Loss of type context

If a struct-typed value is used where its type is not known, it becomes just a normal buffer.
```luau
local raw: buffer = cat
```
After that, struct field syntax no longer works:
```luau
print(raw.Age) -- attempt to index buffer with 'Age'
```
The value must be cast back to a struct type for the compiler to use struct sugar again:
```luau
local cat2 = raw :: Cat
print(cat2.Age)
```
### Runtime behavior

At runtime, Cat values are ordinary buffers:
```luau
print(type(cat)) --> "buffer"
```
There is no VM support for structs, no runtime tag, and no new buffer behavior.

### Scope

This proposal is intentionally compiler-only:

- no VM modifications
- no runtime metadata
- no new built-in type at runtime
- no methods
- no inheritance
- no dynamic object behavior

It only changes parsing, type checking, and code generation.

## Drawbacks

This adds syntax and compiler complexity for functionality already expressible with manual buffer code. It also introduces a concept named `struct` that exists only at compile time, which may confuse users expecting a new runtime feature.

Another drawback is that field syntax depends entirely on type context. A value can behave like a struct in one place and like an ordinary buffer in another, depending on whether the compiler knows its struct type.

## Alternatives

One alternative is to continue writing manual buffer code. This keeps the language smaller, but requires more boilerplate and manual offset logic.

Another alternative is to use external code generation tools or plugins. This avoids language changes, but loses standard syntax and compiler awareness.

A third alternative is to add real runtime structs. This proposal explicitly does not do that as since it risks VM overhead.
