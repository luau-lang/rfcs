# Use `'` for Singleton Strings

## Summary

Adjust type inference to assume that string literals written using single quotes (eg `'string'`) should always be inferred to have a singleton type, and that strings written using `"double quotes"` should always be typed `string`.

## Motivation

It's quite tricky for us to consistently infer a singleton type in exactly all of the places where it's what the developer intended, and for us to always infer `string` where it's what the developer intended.

Today, our inference strategy is to assume that a string literal should have type `string` if possible, but that type can be constrained to a singleton if necessary.

This strategy is broadly successful, but fails in a few ways.

### Bugs

We have bugs that prevent us from deducing the proper type in certain cases.  For example: https://github.com/luau-lang/luau/issues/1483

```lua
type Form = "do-not-register" | (() -> ())

local function observer(register: () -> Form) end

observer(function()
	if math.random() > 0.5 then
		return "do-not-register" -- TypeError: Type 'string' could not be converted into '"do-not-register" | (() -> ())' Luau(1000)
	end
	return function() end
end)
```

### Singleton Strings and Generics

The currently implemented inference strategy doesn't always do what the developer intended in the presence of generics.  Whenever a string literal is passed to a generic, it is possible for it to be typed `string`, and so that's what Luau infers.

In the following snippet using the [luaup](https://github.com/jackdotink/luaup) project, a developer clearly wanted `Kind` to be bound to a string singleton and not `string`.  They were instead forced to write a bunch of casts:

```lua
local function new_token<Kind>(kind: Kind, text: string?): cst.TokenKind<Kind>
    return { kind = kind, text = text or kind :: any, span = span, trivia = trivias }
end

return table.freeze {
    semicolon = new_token(';' :: any) :: cst.TokenKind<';'>,
    equals = new_token('=' :: any) :: cst.TokenKind<'='>,
    colon = new_token(':' :: any) :: cst.TokenKind<':'>,
    comma = new_token(',' :: any) :: cst.TokenKind<','>,
    dot = new_token('.' :: any) :: cst.TokenKind<'.'>,
    endd = new_token('end' :: any) :: cst.TokenKind<'end'>,
}
```

## Design

Languages like Lisp and Erlang offer a concept called an _atom_, which is essentially an interned string.  Atoms and Luau string singletons are largely used for the same purpose: People use them to tag their unions and to build enumerations.

Given that this is the most common use case, it makes quite a lot of sense to offer separate syntax to allow developers to be precise about what they want.

In this RFC, we propose the syntax `"foo"` for a string of type `string`, and `'bar'` for a string with the singleton type `"bar"`.

This makes a whole bunch of inference scenarios completely trivial.

```lua
-- fruits : {string}
local fruits = {}
table.insert(fruits, "avocado") -- These are just strings
table.insert(fruits, "hazelnut")
table.insert(fruits, "carolina reaper")
```

```lua
type Ok<T> = {
    tag: 'ok', -- the tag of a tagged union should use single quotes
    data: T
}
type Err<E> = {
    tag: 'error',
    error: E
}
type Result<T, E> = Ok<T> | Err<E>

function try_very_hard()
    local n = math.random()
    if n > 0 then
        return {tag='ok', data=n}
    else
        return {tag='err', error="I failed"} -- no ambiguity: The tag is a singleton and the error message is not.
    end
end

local r: Result<number, string> = try_very_hard()
```

And

```lua
local function new_token<Kind>(kind: Kind, text: string?): cst.TokenKind<Kind>
    return { kind = kind, text = text or kind :: any, span = span, trivia = trivias }
end

return table.freeze {
    semicolon = new_token(';'), -- The string here uses single quotes, and so the generic Kind can only be ';'
    equals = new_token('='),
    colon = new_token(':'),
    comma = new_token(','),
    dot = new_token('.'),
    endd = new_token('end'),
}
```

### Implementation

This is a very simple adjustment to the constraint generation phase of type checking.  Most of the actual work involves disabling the code to infer when a particular string literal must be a singleton.

It is so easy to implement, in fact, that we've already done it as an experiment.  As of this writing, you can try it out by setting the flag `DebugLuauStringSingletonBasedOnQuotes`.

## Drawbacks

There are a number of pretty serious drawbacks to this proposal:

First, and arguably most importantly: The inference we already have is performing quite well!  I could only find one ticket that points out a bug in the system: https://github.com/luau-lang/luau/issues/1483.  This bug is a little bit esoteric and can probably be fixed in a reasonable timeframe, so it is not itself strong evidence that we should change the language.

Secondly, most code formatters in modern use will automatically change all string literals to use double quotes because that is considered good style.  If this RFC is implemented, those tools will need to be updated so they do not break type inference.

This is also a change to type inference, and not type checking.  It will therefore also affect nonstrict mode.  We probably won't report any spurious errors in this case, but autocomplete quality might suffer for code that hits this.

Lastly, this is probably going to cause some developer confusion unless we are very clear in our communication.  We are going to have to document and explain this to developers and we are going to have to announce it fairly loudly so that people know about this, and the necessary changes (if any) to their code.

## Alternatives

Put simply, we could do nothing and fix the remaining bugs with our singleton inference.  It's almost there.

The interaction between singletons and generics is unfortunate, but could be resolved another way.  One idea is for bounded generics to afford a "strict subtype" constraint.  Developers could use this to write a function with a generic that must be some string singleton.
