# Type Guards
## Summary

This RFC proposes adding type guards to Luau. Type guards are a function that returns a single boolean predicate, narrowing the type of one of their arguments. They can be used exclusively within control flow statements, at the top level of expressions.

```lua
function isFoo(x): x is Foo
    return x.type == "foo"
end
```

## Motivation
Luau currently has a powerful inference engine capable of type narrowing. This allows for `if pet.type == "dog" then pet.meow() end` statements to correctly raise type errors.

Type narrowing is however currently limited to inline statements. More complex narrowing conditions must be duplicated everywhere a narrowing is required.

Type guards allow for the narrowing logic for type to be encapsulated within a single reusable function.

In some instances, the logic being performed for narrowing may not be immediately obvious. Consider the following example:

```lua
type LegacyAction = { id: number, metadata: {} }
type ModernAction = { id: string, data: {} }
type Action = LegacyAction | ModernAction

local action: Action = getAction()

if typeof(action.id) == "number" then
    -- Use action.metadata
else
    -- Use action.data
end
```

A type guard instead allows us to write this code as:

```lua
function isLegacy(x: Action): x is LegacyAction
    return typeof(x.id) == "number"
end

if isLegacy(action) then
    -- Use action.metadata
else
    -- Use action.data
end
```

Additionally, if the behaviour required to discriminate legacy actions changes in the future (for example, a third type is added that returns to using numbers as the ID), only the single guard function needs amended.

Type guards also serve to simplify code when operating with more complex types. For example:

```lua
type Tree = { value: number, left: Tree?, right: Tree? } | nil

function isLeaf(x: Tree): x is { left: nil, right: nil }
    return x ~= nil and x.left == nil and x.right == nil
end

local t: Tree = getTree()
if isLeaf(t) then
    print("Leaf:", t.value)
    print(t.left, t.right)  -- Both known to be nil by the type solver
end
```

## Design
### Syntax
The proposed syntax is for functions to optionally have a return type of `: x is T`, where `x` must be one of the arguments to the function, and `T` is the type the function narrows to. This amends the grammar to:

```
ReturnType ::= Type | TypePack | GenericTypePack | VariadicTypePack | TypePredicate
TypePredicate ::= NAME 'is' Type
```

Type guards are allowed to be defined a members of tables, and the implicit `self` variable on methods is allowed to be used on self-call functions. The previous tree example could utilise this as:

```lua
local Tree = {}
Tree.__index = Tree
type Tree = setmetatable<{ value: number, left: Tree?, right: Tree? }, Tree>

function Tree.new(value: number): Tree
    return setmetatable({ value = value }, Tree)
end
function Tree:isLeaf(): self is { left: nil, right: nil }
    return self.left == nil and self.right == nil
end
```

### Semantics
When type guards evaluate as true, the type at the call site should be intersected with the restriction present in the predicate. The above `isLeaf` function called as `if tree:isLeaf() then` results in a new type of `tree: typeof(tree) & { left: nil, right: nil }`, which will simplify to `{ @metatable Tree, { value: number, left: nil, right: nil } }`. (In reality this example currently ends up with some pretty nasty types, but it serves as illustration.)

### Restrictions
Type guards may only be used in the control flow statements `if`, `elseif` and `while`, the condition of an `if ... then ... else ...` expression, and assert statements. Retaining the value of a guard in a variable may lead to stale predicates no longer holding true.

Multiple type guards may be combined using boolean operators. `and` performs intersection, `or` performs union, and `not` performs a compliment. This does not introduce new compliment logic to luau types, rather performing the same basic compliments that can be found in a statement such as `if (not (typeof(foo.x) == "number")) and foo.y == nil then`.

Assigning the value of a type guard to a variable (`local foo = isCat(x)`) or its use in a more complex expression (`foo(isCat(x))`) is not disallowed, though the predicate returned by the type guard is demoted to a simple boolean value and no longer serves the narrow the type of the subject variable. It is suggested that lint rules may be used to warn about these cases.

Type guard functions are not permitted to have multiple return values. `function foo(x): (x is number, string)` is disallowed, as is `function foo(x): (x is number, x is string)`.

### Additional type checks
A type guard returning any value other than a boolean is considered a type error. Within the type function, standard narrowing will be occurring. If the type solver identifies a `return true` but the narrowed subject variable cannot inhabit the type being asserted, a type error is raised. On the contrary, a `return false` has no additional checks performed.

If the annotated type of the subject variable in a type guard's function definition is not assignable to the type in the predicate, a type error is immediately raised.

### Runtime semantics
At runtime, the type information is not retained. The predicates returned by type guards are treated as simple booleans.

## Drawbacks
The restriction to exclusively control flow only prevents some more powerful patterns from being utilised. For example, the following pattern would not work:

```lua
local pet: Pet = getPet()
local wasDog = isDog(pet)
pet:mutateIntoCat()  -- Adds meow(), changes the discriminator used by isDog, but retains bark()
if isCat(pet) and wasDog then
    print(`My pet can {pet.meow()} and also {pet.bark()}!`)
end
```

This is an acceptable compromise, as situations like these are uncommon. We also have no guarantee that `pet` was not further mutated to remove `bark()`, so `wasDog` can no longer be relied upon. `isDog()` could instead be replaced with a `canBark()` type guard, giving `if isCat(pet) and canBark(pet) then`.

## Alternatives
Not adding type guards, retaining the existing status quo, still allows for narrowing of types using inline statements instead.

When more complex logic is desired, programmers could write their own type guard-esque functions, returning booleans, then manually perform casts on types based on the return value.
