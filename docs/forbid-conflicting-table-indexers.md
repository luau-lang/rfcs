# Forbid Conflicting Table Indexers

## Summary

Let's forbid table types from simultaneously having named properties and an indexer that covers the same string.

## Motivation

Tables are complicated!  One of the particular problems we run into frequently is the case where a table type has both named properties and an indexer.  Our rules in this situation aren't super consistent or sound.

```luau
function print_them(t: {[string]: string, n: number})
    for k, v in t do
        -- k : number
        -- v : string!

        -- Crashes
        print(`{k} = {string.lower(v)}`)
    end
end

print_them({
    one="two",
    two="three",
    n=2
})
```

These clashing indexers also make inference of expression indexing fickle and surprising:

```luau
function get_key()
    return "n"
end

function foo(t: {[string]: string, n: number})
    local a = t[get_key()] -- string
    local b = t["n"]       -- number because it happens that the type inference engine can see that this is equivalent to t.n
end
```

## Design

We impose a new restriction on Luau table types: Tables with indexers that cover strings cannot also have named properties.

If the indexer does not cover strings, then named properties are fine.  For instance, we allow `{[number]: string, n: number}` but not `{[any]: string, n: number}`.

We'll emit a type error when we spot a table type annotation like this.

In the type algebra, we define an intersection type like `{n: number} & {[string]: boolean}` to be equivalent to `never`.  There is no other sound approximation.

We change the unsealed table inference logic: If an unsealed table needs a string indexer, that indexer will also be widened to encompass any named properties that are added to it.  For example:

```luau
function mkTable(key: string)
    local t = {}
    t[key] = "hello" -- t : {[string]: string}
    t.n = 1          -- t : {[string]: number | string}

    return t
end
```

## Drawbacks

The most prominent drawback is that there is existing Luau code out there that successfully does work using types like this.  The behaviour we offer right now is a little bit inconsistent, but it's close enough to allow people to express some useful things.  Roblox's own React port, for instance, takes advantage of this.  We lose some precision and expressiveness in exchange for predictability.

## Alternatives

We could decline to fix this if we think it's useful enough to suffer the `t[get_key()]` inaccuracy described above.

This change doesn't actually affect the soundness of table iteration that much: We should already be inferring `(unknown, unknown)` for the key and value types of any inexact table and take the union of any indexer and all named properties for an exact table.  These would improve the soundness of the type system whether or not we restrict string indexers.
