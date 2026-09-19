# Amend Global Type Functions to Accomodate Read/Write Indexing

This RFC supersedes [PR 162](https://github.com/hardlyardi/rfcs/blob/newindex-and-rawset-type-functions/docs/type-functions-newindex-and-rawset.md)

## Summary

Amend some global type functions (`keyof`, `rawkeyof`, `index`, `rawget`) to take in an optional trailing argument,
limiting their use to the "read" or "write" types of a table.

## Motivation

Currently, there are no built-in type functions with functionality to access the read/write variants of keys in a
table-like type. Additionally, `rawget` and `rawset` are currently typed unsafely, so this RFC improves the soundness
of existing globals.

## Design

The following existing type functions:

- `keyof<TableLike>`
- `rawkeyof<TableLike>`
- `index<TableLike, Key>`
- `rawget<TableLike, Key>`

Do not support table-like types with differing read and write types; including, but not limited to
[read-only properties](https://rfcs.luau.org/property-readonly.html). This RFC proposes adding an optional trailing
argument, a string literal type, to each. The current type expression `keyof<TableLike>` will become equivalent to the
expression `keyof<TableLike, "read"> | keyof<TableLike, "write">`.

Similarly for other global type functions:

```luau
type Meow = {
    read purr: true,
}
-- Write Key 'purr' does not exist in type
type Mrrp = index<Meow, "purr", "write">
-- 'true'
type Mrrp = index<Meow, "purr", "read">
```

Accessing a field which does not exist in a table should emit a type error to prevent mistakes as always.

Since global type functions can be called within user-defined type functions as globals, it might be helpful to ensure
that the global can accept strings as well as canon stringletons.

## Drawbacks

- This proposal relies on an optional stringleton in a type function. Generally, type functions should probably operate
on types, and in this case we are treating one as a runtime value in an API.

## Alternatives

- Do nothing. User-defined type functions can compute this behavior - but this isn't desired due to human error,
fragmentation of common behavior, and time to implement. We should strive to make the language easy to use.
- Do not error when indexing a field which does not exist. This would silently send out an 'unknown' in a place where a
user is likely doing something wrong. However, this alternative makes things easier to work with unions in table-like
types & keys.
- Add new global type functions (`newindex`, `rawset`, `newkeyof`, `rawsetkeyof`), for a total collection of:
`index`,
`rawget`,
`newindex`,
`rawset`,
`keyof`,
`rawkeyof`,
`newkeyof`,
`rawsetkeyof`.
Horrifying. **No.**
