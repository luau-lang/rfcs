# eprint global function

## Summary

A new global function `eprint` will be added to the standard library. The behavior will be identical to `print` but emit to `stderr` by rather than `stdout`.

## Motivation

It is desirable to separate the legitimate output of a program from incidental output like errors and logs. This is supported by most terminals having two separate output streams: `stdout` and `stderr`. Major terminal shells (Bash, Windows Command Prompt, Powershell) support piping `stdout` and `stderr` separately, allowing users to treat them differently.

Many programming languages also support writing to both `stdout` and `stderr` separately. `eprint` would add this functionality to Luau, bringing it more in line with other languages.

This would be very helpful for logging purposes as well as writing errors for CLI tools.

## Design

A global `eprint` would be added to the global environment. It would have the same type as `print` (`<T...>(...: T...)`), and behave identically but instead of writing to `stdout` it would write to `stderr`.

### Prior Art

- JavaScript has `console.log` as an equivalent to `print` and `console.error` as an equivalent to `eprint`.
- Rust has `println!` and `eprintln!` macros that perform equivalently to `print` and the proposed `eprint`
- Python has `print` designed around this: `print(...)` prints to `stdout` and `print(..., file=sys.stderr)` prints to `stderr`
- C has `fprintf` that works similiar to Python's `print`: `printf(stdout, ...)` and `printf(stderr, ...)`

These are just examples. Based on further research on this that it seems roughly split between functions that accept a stream handle to write in (like C or Python) and those that use globals (like JavaScript or Rust). Given Luau currently has no concept of stream handles and `print` already exists, it seems more reasonable to propose the latter.

By default, `stderr` is generally not buffered in the same way as `stdout`. This RFC acknowledges this, but does not specify whether `stderr` should be changed to buffer or not: this is likely better left to the embedder rather than decided by Luau itself.

## Drawbacks

If a normal standard library edition starts with a point against it, a global starts with five points against it. It may be worth not adding this function simply because it is another global. However, this function does something very predictable and it's unlikely that people have variables with the same name that don't do something similar. A search on GitHub says that this is the case.

## Alternatives

A more feature-complete `stdio` library could be added in lieu of `eprint`. It would be more complicated, and carry with it many more design considerations, but it may be a better option. However, this is not incompatible with this function: globals for printing to `stdout` and `stderr` are more ergonomic than putting them in a library, and both `print` and `eprint` could exist alongside a hypothetical `stdio` library.

Rather than making another function that behaves like `print`, a function could be designed that allowed printing to _either_ `stdout` and `stderr` (and potentially do other things, like format input). In the interest of simplicity and ergonomics, this was not proposed because the design was either complicated or annoying.
