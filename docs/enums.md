## Summary

This RFC introduces first-class enumeration datatypes into Luau, both at runtime and in the type system.

## Motivation

Luau currently supports basic enum-like fields in the type system by means of string singleton unions (`"foo" | "bar" | "buzz"`). While useful in many cases, this functionality cannot be used to express enumerations that have meaningful values associated with the enumeration. It also faces challenges regarding singleton to `string` demotion.

At current, it can be very challenging to express an enumeration that has both names and values in Luau in a way that is both type-safe and ergonomic to use.

First-class enumerations also present an opportunity for more efficient serialisation of some data structures in environments where this is desired.

## Design

A new contextual keyword `enum` is introduced, which is used as shown:

```luau
enum DemonstrationEnum
	Foo = 1
	Bar = 2
end
```

Type annotations can optionally be specified on an enum item, annotating the value of that item:

```luau
enum DemonstrationEnum
	Foo: number = 1
	Bar: number = 2
end
```

Not all members of an enum must share the same type. That is to say, the following code is legal:

```luau
enum DemonstrationEnum
	Foo = 1
	Bar = "two"
end
```

Further, multiple names may share the same value:

```luau
enum DemonstrationEnum
	Foo1 = 1
	Foo2 = 1
end
```

All names must be unique within a single enum. The following code is illegal:

```luau
enum DemonstrationEnum
	Foo = 1
	Foo = 2  -- syntax error: Duplicate enum name "Foo" (see previous definition on line 2)
end
```

An enum requires a minimum of one item. An enum with no items would have an  type of `enum<never>` and produce a type alias of `never`, which is meaningless.

```luau
enum DemonstrationEnum  -- syntax error: No enum items defined
```

An enum can be exported using `export enum Name`. Two scripts defining an enum with identical names define two distinct enums.



The value of an enum item must be a expression that the compiler can fully evaluate to a non-nil constant of type `number`, `string`, `boolean` or `integer`. For example, `Foo = math.rand()` would not be legal, but `Foo = 1 + 1` is known to be `Foo = 2` and is therefore legal. `nan` is explicitly disallowed as a legal value.

An enum is constant and its items cannot be altered at runtime.

The enum itself, `DemonstrationEnum` in these examples, is represented as a new language primitive: `enum`.

Individual members of an enum can be accessed using `DemonstrationEnum.Foo`. This introduces the second new language primitive, an `enumitem`.

An `enumitem` has a read-only field `name` containing its name, and a read-only field `value` containing its value.

### Library additions

A new `enum` library is introduced to the language:

```luau
function enum.fromName<T>(e: enum<T>, name: string): T?
```

Returns the item from enum `e` whose declared name is equal to `name`. Returns `nil` if no match is identified.

---

```luau
function enum.fromValue<T>(e: enum<T>, value: any): T?
```

Returns the first item from `e` whose associated value compares equal to `value`. If multiple members in the enumeration share a value, the first matching member in declaration order of the enum is returned. Returns `nil` if no match is identified.

---

```luau
function enum.items<T>(e: enum<T>): {T}
```

Returns a table containing all items, in order, that are members of enum `e`.

---

```luau
function enum.enumOf<T>(item: T): enum<T>
-- where T <: enumitem
```

Returns the enum the enum item `item` is a member of.

---

No function is currently proposed for dynamic creation of enums. This is due to the desire for optimal serialisation support.

### Type system interactions

`enum<T>` is the type of an enum object whose members are represented by the enum-item union `T`; the generic argument is a union of enum-item types, not a union of their associated value types. That is to say, `DemonstrationEnum` is of type `enum<DemonstrationEnum.Foo | DemonstrationEnum.Bar>`. `enum<unknown>` is the top type for enumerations.

The non-generic type `enumitem` is added as the top type for all enum members. Every nominal enum item type is a subtype of `enumitem`. The definition of an enum introduces a type alias of the same name which represents a union of all members of that enum.

As an example:

```luau
enum DemonstrationEnum
	Foo = 1
	Bar = 2
end
local de1 = DemonstrationEnum  -- type: enum<DemonstrationEnum.Foo | DemonstrationEnum.Bar>
local de2: DemonstrationEnum  -- type: DemonstrationEnum.Foo | DemonstrationEnum.Bar
local foo1 = DemonstrationEnum.Foo  -- type: DemonstrationEnum.Foo
local foo2: DemonstrationEnum.Foo  -- type: DemonstrationEnum.Foo
DemonstrationEnum.Foo.name  -- type: "Foo" (singleton string, not just `string`)
DemonstrationEnum.Foo.value  -- type: number

-- An example for when `enumitem` might be used instead of the nominal types
local function getEnumItemName(ei: enumitem): string
	return ei.nameend
```

Enumeration members are nominal. The following code is a type error, despite both the name and the value matching:

```luau
enum DemonstrationEnumOne
	Foo = 1
end
enum DemonstrationEnumTwo
	Foo = 1
end
-- This line is a type error:
local foo: DemonstrationEnumOne = DemonstrationEnumTwo.Foo
```

Refinements on an enum item can be performed with direct equality, or by equality of the `name` field (as shown below). This refinement logic comes as part of general union refinement, and `name` being a singleton string. Refinement on `value` is possible by the same logic, but is unlikely to narrow an unknown enum member particularly strongly (for example, `.Foo.value` and `.Bar.value` could identical `number` types).

```luau
function foo(bar: DemonstrationEnum)
	if bar == DemonstrationEnum.Foo then
    	-- Narrows to `bar: DemonstrationEnum.Foo`
	end
	if bar.name == "Foo" then
	    -- Narrows to `bar: DemonstrationEnum.Foo`
	end
end
```

Intersections between two nominal enum items, such as `DemonstrationEnum.Foo & DemonstrationEnum.Bar`, reduces to `never`. Intersection between a nominal enum item type and `enumitem`, such as `DemonstrationEnum.Foo & enumitem`, reduces to the nominal enum item type (`DemonstrationEnum.Foo`).

### Interactions with existing language features

```luau
type(DemonstrationEnum) == "enum"
typeof(DemonstrationEnum) == "enum"

type(DemonstrationEnum.Foo) == "enumitem"
typeof(DemonstrationEnum.Foo) == "enumitem"

-- It may be preferred to have a "nicer" string representation
-- Currently this RFC proposes a less "nice" representation following existing convention
tostring(DemonstrationEnum) == "enum 0x..."
tostring(DemonstrationEnum.Foo) == "enumitem 0x..."

DemonstrationEnum.Foo == DemonstrationEnum.Foo
DemonstrationEnum.Foo ~= DemonstrationEnum.Foo.value
DemonstrationEnum.Foo ~= OtherEnum.Foo

-- runtime error: attempted to assign to immutable enum object
-- type error: cannot assign to readonly property 'Foo'
DemonstrationEnum.Foo = OtherEnum.Foo

DemonstrationEnum["Foo"] == DemonstrationEnum.Foo  -- Can be considered an alternative to enum.fromName

-- runtime error: attempted to access enum item "DoesNotExist" which does not exist
-- type error: Item "DoesNotExist" not found in enum "DemonstrationEnum"
DemonstrationEnum.DoesNotExist
DemonstrationEnum["DoesNotExist"]  -- errors as above

#DemonstrationEnum == 2  -- Number of items in the enum

-- Iteration order is declaration order
for index, item in DemonstrationEnum do
	print(index, item)
	-- 1, enumitem 0x... (ie Foo)
	-- 2, enumitem 0x... (ie Bar)
end
```

### Restrictions

All enums must be defined at the top level of a script.

While this restricts flexibility, this is currently a strict requirement for efficient serialisation. This restriction may be relaxed in the future.

## Drawbacks

This RFC increases the complexity of the language, and introduces two new language primitives.
