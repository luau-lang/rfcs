# Type Validation and Structural Comparison Utility

## Summary

Introduce a built-in method for `Type` objects in Type Function library (e.g., "Type:verifyFields" or "Type:matches") that validates whether a given object conforms to a target type definition, returning detailed error metadata.

## Motivation

Currently, implementing structural validation in Luau Type Functions requires manually iterating through properties using `Type:properties()`, checking existence via `readproperty`, and comparing types using `Type:is()`.

As shown in my example, validating a simple Interface requires over 20 lines of repetitive imperative code. This is:
1. Error-prone: Developers must manually handle edge cases like optional fields and indexers.
2. Verbose: Common patterns (like ensuring a table matches an interface) are rewritten constantly.
3. Inconsistent: Different developers will return different error formats, making library interoperability difficult.
4. Needs a Luau table to mimic a type definiton and it's fields to check against.
5. Becomes more complex the bigger your type becomes.
6. Having the type to be an actual type, not a mimic can still be crucial to have exported, it can be bad when you end up having a mimic and a real type.

A built-in utility would allow for "Type-Driven Validation" where the type system itself acts as the schema validator for runtime-adjacent logic.

## Design

I propose adding a method to the `Type` object accessible within type functions.

API Signature

```lua
type TypeValidatorErrorKind = 'type-mismatch' | 'field-missing' | 'unexpected'
type ValidationError = {
    kind: TypeValidatorErrorKind,
    field: string,
    targetType: Type,
}

type ValidationOptions = {
    strict: boolean
}

-- Within a type function context:
function Type:matches(target: table, options: ValidationOptions): (boolean, ValidationError?)
```

Behavior

**target**: The object to be validated against `self` type
**options.strict**: If `true`, the presence of fields in `target` that are not defined in `self` (the schema) will trigger an `unexpected` error. If `false` (default), extra fields are ignored.

Error Mapping
1. `field-missing`: A property exists in the type definition but is absent in the target.
2. `type-mismatch`: A property exists in both, but `:is()` returns false.
3. `unexpected`: (Strict mode only) A property exists in the target but not in the definition.

Examples

Current Approach
```lua
type function Person(obj)
    if not obj:is("table") then
        error("Object must be a table")
    end

    local properties = obj:properties()
    
    local PersonType = {
        Name = "string"
    }

    for Field in PersonType do
        Field = obj:readproperty(types.singleton(Field))

        if not Field then
            error(`Person is missing {Field}`)
        end
    end

    for k, v in properties do
        local value = v.read
        local key = k:value()

        local targetType = PersonType[key]

        if targetType and not value:is(targetType) then
            error(`Person {key} must be {targetType}`)
        end
    end

    return obj
end

local function CreatePerson<T>(person: T): Person<T>
    return person
end

local Peter = CreatePerson({
    Name = true --// Person Name must be string
})
```

Proposed Approach

```lua
local Person = {
    name = "john"
}

type PersonType = {
    Name: string
}

local Success, Error = PersonType:matches(Person, { strict = false })

if not Success then
    if Error.kind == Enum.TypeValidatorErrorKind.Mismatch then
       error(`Person field {Error.field} expected to be {Error.targetType}`)
    elseif Error.kind == Enum.TypeValidatorErrorKind.FieldMissing then
        error(`Required field {Error.field} is missing!`)
    end
end
```

## Drawbacks

Must handle recursive types and unions carefullly to avoid infinite loops or performance degradation during analysis.

## Alternatives

Write a runtime object schema validator.

## Unresolved Questions

1. How should this behave with `indexers` (dictionary types), especially with `strict` mode?
