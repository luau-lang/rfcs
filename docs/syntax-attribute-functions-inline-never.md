# Inline Attribute for Functions

## Summary

This RFC proposes a `@[inline("never")]` function attribute. This attribute's presence will restrict the compiler from inlining a function.

## Motivation

When optimizing Luau code it is common to place cold-path code into a function that cannot be inlined to reduce both bytecode size and cpu time usage.

```luau
local function cold_path()
	clean_up_operation()
	error(some_error_value())
end

local function do_something()
	if error_state() then
		cold_path()
	else
		return something()
	end
end
```

In the above example, if `cold_path` were able to be inlined, it might make `do_something` unable to be inlined, even though `do_something` being inlined could be much more beneficial. This is already possible today by turning `local function cold_path()` into `local cold_path; function cold_path()`.

This RFC proposes the introduction of an `@[inline("never")]` attribute to replace the hacky method and give users more power to optimize their code.

## Design

The `@[inline("never")]` attribute can be added to all functions, both named and anonymous.	When the attribute is present, the function should never be inlined, even if the Luau compiler's cost model evaluates that it should be inlined.

No argument to `@[inline]` other than `"never"` should be allowed. Other possibilities may be added in the future, but that is outside the scope of this RFC.

## Drawbacks

Users may be confused that they can only use `"never"`. This may also increase the amount of time that users spend optimizing their code.

## Alternatives

* Do nothing. Users will still be able to use hacky methods to work around the lack of this attribute.
* Add a `@noinline` attribute. This makes sense if the intent is to never add any attribute to force or encourage inlining, but what this RFC proposes leaves those possibilities open as an extension to the attribute's parameters.
