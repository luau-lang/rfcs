# Format Specification for String Interpolation 

## Summary

Allow string interpolation to be able to use all format specifiers supported by `string.format`. It is delimited by `;` in interpolated strings.

## Motivation 

Most languages have some form of template strings that easily allow number formatting. Python has f-strings, C has `printf` and C# has native string interpolation with specifiers. Lua has `string.format`, but in terms of readability, string interpolation is superior, but lacks the ability to effectively format floats and integers.

## Design

String interpolation is internally implemented using `string.format` with `%*`. Selecting a different format specifier should be relatively straightforward.
Format specifiers must be added after a `;` onto interpolated values. This will support most future additions to `string.format`.

Some examples:

```luau
local balance = 100.2035
print(`You have ${balance;.2f} in your bank account.`) 
```
`You have $100.20 in your bank account.`

```luau
local digit = 12345
print(`{digit} is {digit;#x} in hex!`)
```
`12345 is 0x3039 in hex!`

```luau
local score = 1030
print(`Score:\
{score;05i}`)
```
`Score`\
`01030`

## Drawbacks

String interpolation is no longer safe for primitives and can error if the format specifier does not match the type.

## Alternatives

- Keep on using `string.format` for format specification. This can be achieved by using `string.format` in place of string interpolation or nesting it within one.

### Delimiters

This RFC was originally denied due to the delimiter.

- `,` was denied as it creates a false impression of adding an argument into the interpolated string. Some users might expect a function that returns `2, ".2f"` to use the second parameter as the format specifier.
- `:` was denied as is used for namecalls, and conflating the symbol with string interpolation could cause confusion.
- `|`, `&`, `#`, `=`, `~`, and `@` are all used in the language already. These would clash with how the symbols are already used in the language.
- `$` is reserved to not be used as a symbol in Luau.
- `!` does not seem like a good choice for Luau as it would introduce a new symbol to normal code when its only use currently is for comment directives. This is the second best option.
- `;` is already used in Luau to separate statements or to separate keys in dictionaries. It is never required to be used in Luau, and seems to be the least evil option as the delimiter. This feels like an acceptable extension to how `;`s are used today.

Other alternative delimiters (like `%`) were denied due to issues with backwards compatibility.