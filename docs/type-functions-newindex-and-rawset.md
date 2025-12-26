# Add `newindex` and `rawset` type functions.

## Summary

Implement two new builtin indexing type functions for `newindex` and `rawset` types.

## Motivation

Currently, there is no built-in type function for accessing the write variants of keys in a table-like type. This means
users have to write it themselves. Additionally, `rawget` and `rawset` are currently typed unsafely, so this RFC
improves the soundness of existing globals.

## Design

### type function: `newindex`

`newindex` provides the table's **write** index result by taking in arguments for a table-like type, and a key. I.e.:

```luau
type Meow = {
    write purr: true,
}
-- 'true'
type Mrrp = newindex<Meow, "purr">
```

Accessing a field which does not exist in a table should emit a type error.

For table-like types with an `__newindex` metamethod, behavior will need to be specialcased to match runtime behavior.
If the base table's index result is nilable (I.e., nil is a subtype of the index result), the output type should reflect
the expression `(base_table_result & ~nil) | __newindex_result` where '~' is the type negation symbol.

For convenience and consistency with the `index` type function, both the table-like type and the key can be unions, in
which case it will result in a union between every index result present in components of the union(s).

### type function: `rawset`

Similarly to `index` and `newindex`, this type function is passed two arguments which may be type unions. `rawset` will
behave identically to `newindex` - however, it will completely ignore the metatable portion of the type.

## Drawbacks

- This change will increase the number of type functions present in the global scope. Over time, large numbers of
utility globals can inflate autocomplete, conflict with user defined aliases & type functions, and make the language
more complicated to "learn".
- Supporting unions fully without insanity could have high runtime complexity.

## Alternatives

- Do nothing. User-defined type functions can compute this behavior - but this isn't desired due to human error,
fragmentation of common behavior, and time to implement. We should strive to make the language easy to use.
- Do not error when indexing a field which does not exist. This requires amendments to the original indexing type
functions, and would silently send out an 'unknown' in a place where a user is likely to be doing something wrong. In
contrast, it makes things easier to work with unions in table-like types & keys.
- Do not support passing unioned table-like types and/or keys. This sidesteps concerns about runtime complexity &
indexing fields which do not exist - however, it ultimately makes the utility less flexible. This would require
amendments to previous RFCs, and people are likely to "hack" the behavior in with inadequate safety or correctness.
