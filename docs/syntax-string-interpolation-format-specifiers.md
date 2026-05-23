# Format Specification in String Interpolation 
## Summary

## Motivation 
Most languages have some form of template strings that easily allow number formatting. Python has f-strings and C has `printf`. Lua has `string.format`, but in terms of readability, string interpolation is superior, but lacks the ability to effectively format floats.
## Design
String interpolation is internally implemented using `string.format` with `%*`. Selecting an additional format specifier should be relatively straightforward.
```luau
local balance = 100.204
print(`You have ${balance;.02f} in your account.`)
```
## Drawbacks
String interpolation is no longer safe and can error if the format specifier doesn't match the type.
## Alternatives
Keep on using `string.format` for format specification.
### Delimiters
This RFC was originally denied due to the delimiter.
`,` was denied as it creates a false impression of adding an argument into the interpolated string. Some users might expect a function that returns `2, ".2f"` to use the second parameter as the format specifier.
`:` is used for namecalls.
`;` is already used in Luau to separate statements or to separate indexes. It's never required to be used in Luau, but seems to be the least evil option as the delimiter.