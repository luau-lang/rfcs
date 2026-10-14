# Unfreeze Cloned Table Types

## Summary

`table.clone` will return an un-frozen version of the table.

## Motivation

calling `table.clone` unfreezes frozen tables. This can be a useful
pattern when treating tables immutably:

```luau
const value = table.clone(get())
value.counter += 1
set(table.freeze(value))
```

However, this behavior is not currently reflected in the type system;
a table with `read` only properties and/or indexers will still be
`read` only after cloning, so you cannot update the counter.

## Design

`table.clone` will be given a magic type function. If it were formed
as a UDTF, it could look something like this:

```luau
type function unfreeze(tbl: type)
	for field, prop in tbl:properties() do
		if not prop.write then
			tbl:setwriteproperty(field, prop.read)
		end
	end

	const write_indexer = tbl:writeindexer()
	if not write_indexer then
		const read_indexer = tbl:readindexer()
		if read_indexer then
			tbl:setwriteindexer(read_indexer.index, read_indexer.result)
		end
	end
end
```

## Drawbacks

- It is possible to promote the type of a table incidentally with
`table.clone` in a way which causes type errors with other code,
but the cases where this happen seem vaguely rare at this time.
Assuming users are annotating their functions sufficiently, the
table would simply demote back to `read` only once it is passed
into a function expecting its type, or an explicit `table.freeze`
could be used.

- Tables with vaguely derranged metatables could gain unsafe `write`
types. There appears to be no precedent in the type system today to
accomodate such heretical constructions.

## Alternatives

- **Do Nothing.** Users could unfreeze tables with the user-defined type
function already shown.

- **Introduce `freeze<T>` and `unfreeze<T>` or similar built-in type
functions.** Doing so is not exclusive with this RFC, and also exists
as a reasonable path to migrate away from a magic type function for
`table.clone`. This was not selected because grouping those here
would make for a very scattered RFC design & motivation.

- **Only unfreeze properties, leave indexers `read` only.** This
decreases the likelihood of mistakenly editing what was not meant to
be edited, but would exasperate existing grievous inconsistencies
between table indexers and table properties.

## Prior Art

In typescript it is possible to construct an inverse to the
`readonly` utility type:

```ts
type removereadonly<T> = {
	-readonly [P in keyof T]: T[P]
};
```

Typescript has `readonly<T>` as a utility type, which Luau does not.
However, Luau could (and in my opinion should) introduce one. Doing so
is not exclusive with this RFC.
