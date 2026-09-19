# Multiline String Interpolation

## Summary

Allow to interpolate multiline strings similarly to single line strings.

## Motivation

Luau currently supports string interpolation by surrounding the string with backticks (`` ` ``). However, this syntax only works for single-line strings. When attempting to create a multiline string with interpolation, users must resort less readable methods, for example escaping newlines:

```
local value = 42
local interpolated = `first line: {value}\
second line: {value +1}`

print(interpolated)
--> first line: 42
--  second line: 43
```

or by interpolating each line separately and concatenating them:

```
local value = 42
local interpolated = `first line: {value}\n` ..
                     `second line: {value + 1}`
print(interpolated)
--> first line: 42
--  second line: 43
```

## Design

To make multiline string interpolation more ergonomic, we propose new syntax that directly supports multiline interpolation:

1. Add an opening and cloisng backtick charaters to the multiline string separator:
    ```
    local value = 42

    local interpolated_1 = `[[first line: {value}
    second line: {value + 1}
    ]]`

    -- or separting the double brackets with any number of equal signs:
    local interpolated_2 = `[=[first line: {value}
    second line: {value + 1}
    ]=]`
    ```
2. Once the multiline string interpolation syntax is recognized, the existing interpolation rules apply within the string body:
3. New lines within `{}` are not considered part of the multiline string.
    ```
    local value = 42

    -- this will still generate a 2-line string
    `[[first line: {value}
    second line: {
        value +1
    }]]`
    --> first line: 42
    --  second line: 43
    ```

## Drawbacks

This syntax may conflict with existing programs that interpolate strings of the form:

```
local value = 42
local interpolated_1 = `[[one line: {value}]]`
print(interpolated_1)
--> [[one line: 42]]

-- or any number of equal signs:
local interpolated_2 = `[==[one line: {value}]==]`
print(interpolated_2)
--> [==[one line: 42]==]
```

Notice that once a line break is introduced within the backticks, the parser raises an error:

```local value = 42
local interpolated = `[[one line: {value}
]]`
print(interpolated)
--> Error:2: Malformed interpolated string; did you forget to add a '`'?
```

## Alternatives

Instead of breaking current programs, we could explore other syntaxes for multiline string interpolation, such as:

* Use `` [[` `` and `` `]] ``: Still has potential conflicts with existing programs that use multiline strings with opening and closing backticks.
* Include the backticks within the square brackets: `` [`[ `` and `` ]`] ``: This syntax is less intuitive and may confuse users who expect backticks to denote interpolation. For the case users choose to include equal signs, it could lead to visually cluttered code, e.g., `` [`==[ `` and `` ]==`] ``.
