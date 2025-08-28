# Function Parameter Names in User-Defined Type Functions

## Summary

This RFC proposes adding a method to include names for function parameters created in user-defined type functions.

## Motivation

Being able to use parameter names in user-defined type functions increases the expressiveness of the type system, which is the primary motivation of the original [user-defined type functions RFC](https://github.com/luau-lang/rfcs/pull/45).

One could postulate that anything that can be written by hand in Luau should also be able to be written programmatically in a user-defined type function, and vice versa. Function parameter names currently don't meet this criteria.

Here is an example and use case:

```luau
type function Fire(Event: type): type
    if not Event:is("function") then
        return types.any
    end
    local Parameters = Event:parameters()
    local Self = EventHandler(types.any)
    if Parameters.head then
        table.insert(Parameters.head, 1, Self)
    else
        Parameters.head = {Self}
    end
    local SelfName = types.singleton("self")
    if Parameters.names then
        table.insert(Parameters.names, 1, SelfName)
    else
        Parameters.names = {SelfName}
    end
    return types.newfunction(Parameters, Event:returns(), Event:generics())
end

export type EventHandler<Event = () -> ()> = {
    -- .. snip ..
    read fire: Fire<Event>,
    -- .. snip ..
}

local my_event: EventHandler<(my_message: string) -> ()>

my_event:fire("Hello, world!") -- function EventHandler:fire(self: { read fire: any }, my_message: string): ()
```

## Design

The following changes need to be made to the user-defined type functions API:

### `types.newfunction`

The `parameters` argument gains an optional `names` field, which acts as a map from parameter position (in the `head` array) to a string singleton type, or `nil`.

- A string singleton type is used instead of just a string for consistency with the table key API and to support future extensions (for example, doc comments).
- Names that are `nil`, not string singleton types, or invalid Luau identifiers should be ignored.

```luau
function types.newfunction(
    parameters: { 
        head: {type}?, 
        names: { [number]: type }?, -- Doesn't need to be a contiguous array
        tail: type? 
    }, 
    returns: { head: {type}?, tail: type? }, -- Return values don't have names (yet? 🤫)
    generics: {type}? 
)
    -- .. snip ..
end
```

### `functiontype:setparameters`

This method gains an optional `names` argument, following the same rules as above.

```luau
function functiontype:setparameters(
    head: {type}?, 
    tail: type?,
    names: { [number]: type }?,
) -> ()
    -- .. snip ..
end
```

### `functiontype:parameters`

The returned table gains an optional `names` field, following the same rules as above. Furthermore, missing/invalid parameter names are always `nil` (and not some other representation, such as string singleton types with string length 0).

```luau
function functiontype:parameters(): {
    head: {type}?, 
    names: { [number]: type }?,
    tail: type? 
}, 
    -- .. snip ..
end
```

## Drawbacks

It's unclear if there are any drawbacks to this. Everything should be backwards compatible, and the motivation seems to align well enough with the goals of the language.
