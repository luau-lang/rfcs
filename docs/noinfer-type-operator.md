# `noinfer` Type Operator

## Summary  

This RFC builds on an alternative listed in [#106](https://github.com/luau-lang/rfcs/pull/106).
<br>This RFC proposes the addition of a new type function, `noinfer`, which could be used to block type inference from various contexts.

## Motivation  

When a user is working with a function which contains a generic type, they may wish for one or more inputs of the function not to contribute to type inference. E.g., a potential version of `table.insert`:

```luau
local function insert<V>(tbl: { V }, value: V)
  -- ...
end
```

Take the following example in the current version of Luau's Type Inference Engine V2:

```luau
local some_table = { 1, 2, 3 }
insert(some_table, true)
```

The above doesn't produce a type error, although it would have in The V1 Type Inference Engine. This is "expected" of Luau's Type Inference Engine V2, but might not be what a user wants.
The purpose of this RFC is to allow users to annotate a type which will not contribute to type inference, such that a polymorphic type's value isn't inferred from an input to a `noinfer` type.

## Design  

For the purpose of this RFC, additions to Luau's type function runtime will not be considered in the main body. This is because relevant changes to the types runtime or `types` library could be investigated in a future RFC.

This RFC proposes a solution in the form of a type function, `noinfer`, which will block type inference on its input when a generic type is instantiated implicitly. This RFC allows, i.e., a `greedy_insert` function to be defined:

```luau
local function greedy_insert<V>(tbl: { V }, value: noinfer<V>)
  -- ...
end
local a: "a"
local b: "b"
local some_table = { a, b }
local c: "c"
greedy_insert(some_table, c) -- TypeError: Type '"c"' could not be converted into '"a" | "b"'
```

In the above code, `greedy_insert` is instantiated as type `'(tbl: { "a" | "b" }, value: "a" | "b") -> ()'`, which causes a type error.
This would behave similarly in a type alias:

```luau
type SomeAlias<T> = {
  foo: T,
  bar: noinfer<T>
}
local function test<Input>(input: SomeAlias<Input>)
  -- ...
end
test({
  foo = "hello",
  bar = false
}) -- TypeError: Type 'boolean' could not be converted to 'string' in an invariant context
```

If all provided generic inputs are represented by `noinfer` generic type(s), the generic should instantiate with `'unknown'`.

## Drawbacks  

- Introduces a new keyword in type contexts.
- Introducing a new "modifier" to types in luau could potentially increase the complexity of the type inference engine.
- The behavior proposed here could be 'icky' in the context of some types & type aliases. Consider if a user has an alias with multiple optional properties they wish to belong to identical types. Someone might try:

  ```luau
  type Foo<T> = {
    foo: T?,
    bar: noinfer<T>?,
    baz: noinfer<T>?
  }
  local test: <T>(Foo<T>): T
  ```

  The above code would instantiate the generic with parameter `T` as `'unknown'` if the user tries to pass values `{bar = 123}`, and would not produce an error given the input `{bar = 123, baz = true}`. This RFC does not remove existing behaviour and does not block future solutions to this problem.

## Alternatives

### Type Attributes

Luau could allow 'type attributes', consistent with current function attribute syntax. Instead of masking a `noinfer` type as a type function, you could use `@noinfer T` as a type with an attribute.

### Greedy Annotation for Generic Types

New syntax could be introduced to allow any inference of a generic type `T` to modify subsequent occurences to follow the behaviour of `noinfer<T>`, expecting a more "precise" type match. This could look something like `greedy T` or `eager T`:

```luau
local function test<greedy T>(foo: T?, bar: T?, baz: T?)

end
test("hi", 1) -- TypeError: Type 'number' could not be converted into 'string?'
```

Because this error is dependant on the "order" of generic inputs traversed during instantiation, this alternative exposes an implementation detail in a way that provides error messages to the user. However, this alternative could solve some drawbacks of this RFC, while following similar motivation. Such a modifier for generic types isn't *exclusive* to this RFC, and could be investigated at a later date.

### Do Nothing

Introduce no new type function or syntax, leave Luau's Type Inference Engine V2 as-is. Due to the reasoning in the motivation section, this doesn't seem desireable.

### Default Behaviour

By default, assume similar behavior to the greedy/eager behaviour mentioned before, and add an annotation to allow polymorphic types to be "non-greedy". This defeats a lot of the purpose of bi-directional type inference in Luau's Type Inference Engine V2, as new features predating this RFC would become essentially opt-in.
