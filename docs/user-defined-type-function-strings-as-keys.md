# Implicitly convert strings in type userdata methods that accept a `key` arg and allow strings as keys in the `props` table of `types.newtable`

## Summary

This RFC proposes allowing strings as keys for methods on the `table` type userdata in type functions, alongside allowing strings as keys instead of types in the `types.newtable` method of the types library.

## Motivation

Currently having to always wrap string literal keys in `types.singleton` is annoying, and becomes especially annoying when using `types.newtable` due to having to write something like the following:

```luau
types.newtable({
	[types.singleton("meow")] = types.unknown,
	[types.singleton("mrrp")] = types.unknown,
})
```

Instead of:

```luau
types.newtable({
	meow = types.unknown,
	mrrp = types.unknown,
})
```

## Design

As a solution, methods that accept a `key` arg on the table `type` instance, and keys in `props` table passed to `types.newtable` will allow strings. With luau implicitly converting these strings to type userdatas.

### Changed methods

### `types` Library

| Name | Current Type | New Type |
| ------------- | ------------- | ------------- |
| `newtable` | (props: {[type]: type \| { read: type, write: type } }?, indexer: { index: type, readresult: type, writeresult: type }?, metatable: type?) -> type | (props: {[type \| string]: type \| { read: type, write: type } }?, indexer: { index: type, readresult: type, writeresult: type }?, metatable: type?) -> type  |

#### Table `type` instance

| Name | Current Type | New Type |
| ------------- | ------------- | ------------- |
| `setproperty` | (key: type, value: type?) -> () | (key: type \| string, value: type?) -> () |
| `setreadproperty` | (key: type, value: type?) -> () | (key: type \| string, value: type?) -> () |
| `setwriteproperty` | (key: type, value: type?) -> () | (key: type \| string, value: type?) -> () |
| `readproperty` | (key: type) -> type? | (key: type \| string) -> type? |
| `writeproperty` | (key: type) -> type? | (key: type \| string) -> type? |

## Drawbacks

Adds implcit behavior, although the behavior is fairly straightforward to understand.

## Alternatives

Add new methods to the types library and on the table type instance for accepting strings as keys, or do nothing.
