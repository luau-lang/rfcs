# Scoped lint silencing

## Summary

We will add the ability to silence lints across scopes smaller than the entire file.

## Motivation

Currently, the only way to disable lints is adding `--!nolint` in the beginning of a file. This will disable either all lints or a specific lint for the entire file. However, this is often far too encompassing. 

Take for example deprecated APIs. Suppose we have the following code...
```luau
foo()
bar()
baz()
```

Later, we want to deprecate `bar`, so we use the `@deprecated` attribute...
```luau
@deprecated
local function bar()
```

We are deprecating this API because we can't remove everything that uses `bar()` all at once, or maybe we're a public library. That means we will want to silence the warning for `bar()`. The only way to do this is with `--!nolint`, and thus...

```luau
--!nolint DeprecatedApi
foo()
bar() -- No more deprecated warning
baz()
```

There are a few consequences of this. First, we only know that this file uses a deprecated API *somewhere*. From the code alone, we have no idea what. This also means that if we do replace our usage of `bar()`, we might not know we can now remove the `--!nolint`. Second, we added this consciously to allow us to keep using `bar()`. However, by adding the lint ignore to the entire file, we now will no longer get warnings about *other* deprecated APIs too, which was not our intent.
## Design

We will add the ability to disable lints on a specific line or range of lines through the use of special comments.

One method will be `--!nolint-next-line LINTNAME`. This will silence the specific lint for the next line of source code (not comments/whitespace)

```luau
foo()
--!nolint-next-line DeprecatedApi
bar()
baz()

--!nolint-next-line DeprecatedApi
-- Even though these next lines are comments,
-- that is okay because it does not apply to trivia.
bar()
```

The comment must be alone on its line, as its not obvious what `x() --[[!nolint-next-line]] y()` should do. If it is not, it will create a static warning.

Another method will be disabling across ranges with `--!nolint-begin LINTNAME` and `--!nolint-end LINTNAME`.

```luau
foo()

-- I'm about to use a lot of bad APIs
--!nolint-begin DeprecatedApi
bar()
bar()
bar()
--!nolint-end DeprecatedApi
```

This will disable the specified lints across everything in between the range. It is expected that every `nolint-begin` will be paired with a `nolint-end` to avoid accidentally removing an end line without its beginning, otherwise a static warning will be thrown. This is based on raw source code, and not the AST, which will be discussed in Alternatives.

For both of these, you can leave off the lint name just as you can `--!nolint` in order to ignore *all* lints. There will also not be lints for if a line *doesn't* have the specified error, similar to `--!nolint`.

## Drawbacks

Nothing obvious.

## Alternatives

There is no shortage of prior design to look at for linting, and we are for choice on alternatives.

### AST Aware Scopes
Rust and selene operate on a code's AST rather than its text. In other words...
```luau
-- selene: allow(undefined_variable)
if f() then
	a()
	b()
	c()
end
```
```rust
// Rust
#[allow(some_lint_name)]
if f() {
	a();
	b();
	c();
}
```
This will operate on the if statement as a whole, including its condition and contents. If you want to narrow down what you're focusing on, you put the comment in a more precise spot. For instance, if we just wanted to say that `f()` is okay not to lint, we'd write:
```luau
if
	-- selene: allow(undefined_variable)
	f()
then
	a()
	b()
	c()
end
```

There's a few reasons why a linter might want to do this.

In Rust's case, its lints do not use special comments, but instead the language's built in attribute syntax, which must be attached to an actual syntax node.

It is also generally immune to auto formatters. Suppose you have this code, where `longName` is linted as undefined:
```luau
-- selene: allow(undefined_variable)
f({ longName(), longName(), longName(), longName(), longName(), longName(), longName() })
```

An auto formatter is likely to turn this into something like:
```luau
-- selene: allow(undefined_variable)
f({
	longName(),
	longName(),
	longName(),
	longName(),
	longName(),
	longName(),
	longName(),
})
```

This does not change the meaning of the code, so this lint still works. However, the same case of `--!nolint-next-line` will not work.

```luau
-- Before formatting, this works.
--!nolint-next-line DeprecatedApi
f({ longName(), longName(), longName(), longName(), longName(), longName(), longName() })

-- After formatting, this now only applies to f().
--!nolint-next-line DeprecatedApi
f({
	longName(),
	longName(),
	longName(),
	longName(),
	longName(),
	longName(),
	longName(),
})
```

This is unfortunate as we either have to have everything inside `--!nolint-begin`, or we need to expect auto formatters to not make changes when `nolint-same-line` is in effect. In the above case, the latter is undesirable--we *do* want it to format, we just don't want it to lint.

This RFC opts not to do this, though the names of the comments are intentionally specific as to what exactly they are doing to allow us to do something like `--!nolint-syntax` (or something). However, the styling issue may be good enough reason to not have `--!nolint-next-line` at all and instead just opt for `--!nolint-begin` and `--!nolint-end`.

For similar reasons, we also do not have `--!nolint-same-line`.

### `--!` as a syntax
Currently, we use `--!` for file-wide configuration such as the aforementioned `--!nolint`, but also things like `--!strict`. Luau does not have any configuration through comments that does not apply to the whole file.

It is possible that `--!` should be kept just for file-wide configuration. In that case, we can choose a different sigil.
