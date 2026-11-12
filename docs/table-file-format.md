# Luau Table file format

## Summary

The Luau Table format (`.table`) is a well-defined subset of Luau which can be used to store and transfer Luau data without allowing expression evaluation or code execution.

`.table` is to `.luau` what `.json` is to `.js` - a format that preserves the ergonomics and mechanics of writing Luau, but simplifies it to a statically resolvable subset suitable for the domain of data recording, authoring and transfer.

## Motivation

Luau is a natural choice for storing data such as configuration: it's familiar to Luau authors, supports comments, and can represent any Luau value. However, Luau files are Turing complete, allowing arbitrary control flow and evaluation, making Luau unsuitable for machine editing and splicing.

For example, consider a tool that wants to change `retries` in this file:

```luau
local base = 3

return {
    retries = base * 2,
    enabled = if DEBUG then true else false,
}
```

The value of `retries` can only be found by evaluating the file, and there is no single place in the source where a new value can be written. In general, a tool can't safely edit a Luau file without understanding everything that leads up to the value it wants to change, and evaluating the file may depend on its environment, cause side effects, or never finish.

The Luau Table format restricts Luau to a subset where every value is written out literally, so the file can be read without evaluation and edited in place by machines, while remaining ordinary Luau to the humans writing it.

## Design

### Values

The Luau Table format contains a single acyclic table definition at its root.

Four primitive types are allowed: strings, booleans, numbers and vectors. All are represented identically in Luau Table as in Luau and permit their full range of values; numbers and vectors have well-defined float precision, strings are formed of bytes, and vectors are 3-wide. These four primitive types are allowed to appear as keys and values, so long as there are no duplicates. Duplicates are detected by value the same way they are in Luau, so `1` and `1.0` are the same key, as are `0` and `-0`. NaN can't be used as a key, just as in Luau.

On top of those four primitive types, nested tables are allowed to appear as values only, never keys, as they have referential identity. Tables may have a contiguous array part with implicit keys, and a map part with explicit keys. To prevent blowup and match Luau table declaration behaviour, nesting is capped at a depth of 255.

Nil is specifically excluded from Luau Table as `[a] = nil` is identical to not writing a key for `[a]`, and `{1, nil, 3}` gives an array with a hole, which we choose not to support with implicit keys - arrays with holes are best represented as maps.

### Syntax

#### Literals

All literal string and number forms are allowed, specifically: single and double quote strings, raw strings of arbitrary level `[==[]==]`, every escape code including unknown escapes (`\n`, `\x`, `\u{}`, `\z`, `\ddd`, etc.), decimal numbers, exponents, hex, binary, and underscore separators. Backtick strings are allowed only without interpolation `{}`, so they behave as plain constant strings. The boolean literals `true` and `false` are allowed.

Note that, while escaped NULs in strings still work (`"\0"`), raw NUL bytes in files are always rejected no matter where they appear in a `.table` file.

#### Blessed names

Number, string and boolean literals are not enough to express the full range of representable Luau values. These are augmented by certain "blessed" identifiers that are allowed to show up in Luau Table, and which line up with standard library constants and functions:

- `math.nan`
- `math.huge`
- `-math.huge`
- `vector.create(x, y, [z])`

The NaN payload bits and exact representation is undefined, as it is for `math.nan`. `vector.create` accepts any number form, identically to Luau.

Unary minus is allowed on number literals and `math.huge` to allow negative numbers and negative infinity to be represented. Unary minus is not permitted as a prefix on `vector.create` or `math.nan`.

#### Tables

Table constructors are written identically to Luau:

- `name = value`, where `name` is an identifier, as sugar for a string key
- `[key] = value`, where `key` is any allowed primitive form, including `[-1]`, `[math.huge]` and `[vector.create(1, 2, 3)]`
- positional entries, which form the array part
- `,` and `;` as separators, with a trailing separator allowed

Implicit array keys and explicit keys share one key space, so `{ "a", "b", [2] = "c" }` has a duplicate key.

#### Comments

Comments are allowed, identically to Luau.

#### Top level

For backwards compatibility with existing Luau configuration, a single `return` at the start of the file is ignored if present, and a single `;` at the end of the file is also ignored if present. We suggest implementors don't write these by default and output just the top-level table declaration.

#### Type syntax

Luau Table format does not support type annotations or casting; all type-related syntax is reserved and may be reintroduced at a later date.

#### Example

```luau
-- Settings for the example service.
return {
    name = "example",
    enabled = true,
    retries = 3,
    timeout = 2.5e3,
    flags = 0b1010_0001,
    threshold = -math.huge,
    origin = vector.create(0, 10, -5),
    tags = { "fast", "beta", },
    [1] = "first numeric key";
    ["display name"] = `Example Service`,
    limits = {
        [true] = math.huge,
        [false] = 0,
    },
}
```

### Well-formedness

Anything not explicitly allowlisted in the Luau Table format is excluded. The allowlist is carefully curated so that all data has value semantics, represents the whole domain of possible values, and has exact storage and precision rules. Files containing non-allowed features or syntax, or duplicate keys, are malformed.

Malformed files _must_ be rejected by all implementors by default in order to be conformant, though a non-rejecting alternative can still be made available as the non-primary way of parsing the Luau Table format if malformed files need to be handled optimistically (e.g. for an editor extension providing syntax highlighting mid-edit).

### Relationship to Luau

Any well-formed `.table` file evaluates in standard Luau, whether as a table expression, or as a complete module if `return` is present (assuming `math` and `vector` exist and have the relevant members).

`.table` files should be read by parsing, not by evaluating them in a Luau VM. Luau Table makes a best effort to yield identical results for well-formed files. However, this is subject to Luau's own limits for parsing, including register counts. Furthermore, running a file in a Luau VM won't reject malformed files, so it is non-conformant, in the same way as the non-rejecting alternative above: fine for trusted files, but not how a conformant implementation reads the Luau Table format.

### Machine editing

Every value in a `.table` file sits at one fixed place in the syntax, and nothing is computed, so a tool can locate a value and replace, insert or delete it in place without understanding the rest of the file. How tools do this, including whether they preserve comments, formatting and ordering, is up to them.

### Scope

This is an ecosystem RFC: the Luau Table format is opt-in, and requires no changes to Luau itself. Tools and runtimes that want to read or write `.table` files implement it themselves.

## Drawbacks

### Another format

JSON and similar formats already exist and are widely supported, so `.table` is one more format for users to learn and for tools to support. We think the benefits of staying within Luau's syntax and value model justify this (see Alternatives), but tools that need to interoperate with other ecosystems will likely still need JSON.

### Loading through a VM is tempting

Because every well-formed `.table` file is also valid Luau, it's tempting to load one with `require`, `loadstring` or similar, and skip validation. Doing so accepts malformed files, including ones containing arbitrary code, so files read this way may not be portable to conformant implementations, and untrusted files must never be read this way.

### Stricter than Luau

Some tables that are valid in Luau are malformed in `.table`. Duplicate keys, which Luau silently overwrites, and `nil` values are both rejected. Someone pasting a table from Luau code into a `.table` file may be surprised when it's rejected.

### NaN isn't bit-exact

Since the NaN payload bits and exact representation are undefined, a NaN read from a `.table` file and written back out isn't guaranteed to have the same bits. Round-tripping is exact for every other value.

## Alternatives

### JSON, TOML, YAML and similar formats

Existing data formats could be used instead. However:

- They can't represent Luau data faithfully. JSON only allows string keys, so number, boolean and vector keys can't be expressed; it has no representation for NaN or infinities, or for vectors; and its split between arrays and objects doesn't map cleanly onto Luau tables, which can have both an array part and a map part. Other formats have similar mismatches.
- JSON doesn't allow comments, which matters for files that humans author by hand.
- They use a different syntax from Luau, so Luau authors have to switch between the two, and Luau-native tools need a separate parser.

### Plain Luau in a sandbox

Data could be stored in ordinary Luau files, evaluated in a sandbox that prevents side effects. However, a sandbox stops side effects but not computation: the file can still contain locals, arithmetic, `if` statements and loops, so a machine still can't tell where to write a new value, and the editing problem remains. A sandboxed file may also never halt, so reading it needs a VM and a timeout.

## Future possibilities

### Type syntax

Type annotations and casts are reserved, so they may be reintroduced later, for example to let a file declare the schema it follows, without breaking existing files.

### A lines format

A companion format, similar to JSON Lines, could store a sequence of records as newline-separated `.table` expressions, each on one line. That would allow appending and streaming without rewriting the whole file. It would need to settle how newlines inside long strings and `\z` escapes are handled, and what a single record may contain.
