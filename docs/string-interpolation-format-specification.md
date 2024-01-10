# Format Specification for String Interpolation

## Summary

This feature allows us to give format specifiers when printing interpolated strings such as `.2f` for a float rounded to two decimal places or `x` for printing in hex. It has all of the format specifiers available in `string.format`. It is delimited by a `,` in interpolated strings. 

## Motivation

Most languages allow for format specifiers such as python f-strings or printf in C. Lua also supports `string.format` but we want to make it easier to use with interpolated strings. 

## Design

Under the hood, this will just be syntactic sugar in addition to the current implementation of interpolated strings. Before, interpolated strings were passed to `string.format` with no format was speified, but now we will also pass the optional format specifier. We decided on `,` to act as the delimiter for format specification.

To give some examples, this is how it would look like in code:

```lua
balance = 100.2035
print(`You have ${balance,.2f} in your bank account`)
```
`You have $100.20 in your bank account`

```lua
number = 12345
print(`12345 is 0x{number,x} in hex!`)
```
`12345 is 0x3039 in hex!`

This will support most additions that could be made to `string.format` in the future as well. 

## Drawbacks

There are no clear drawbacks to this.

## Alternatives

We have also considered allowing arbitrary format specifiers and not just ones supported by `string.format`. We would allow `__tostring` to take a second format argument, and that specifier get passed into it. We determined that we won't do this for now because it could mess up backwards compatibility on existing `__tostring` calls and a performance regression since we can't assume `tostring` will only have one argument anymore. Also for specifiers that work with `string.format` will be slower than just using `string.format`
