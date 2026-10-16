# Table Destructuring
This RFC proposes (yet another) table destructuring syntax for Luau to make it convenient for Luau programmers to extract table keys.

## Summary
Table destructuring is a common ask for Luau because of how often programmers interact with tables. It is a convenience feature, intended to make it syntactically easy to write programs that unpack table values in a single line of code. Prior RFC's have suggested similar syntax, but these have often suffered from excess verbosity or would break backwards compatibility. This RFC requires us to perform unbounded lookahead in the lexer, but in return, provides us a symmetric syntax for constructing and destructuring tables.

## Motivation
Broadly, this feature makes it easier to work with tables. 

### Example 1
Abbreviated from [lute](https://github.com/luau-lang/lute/blob/8965cd90c1a23bc56f017eb0fc973a0e0c5e8291/lute/std/libs/fs.luau#L224)

```luau
function fslib.removeDirectory(path: Pathlike, options: RemoveDirectoryOptions?): ()
	...
    for _, entry in entries do
				local entryPath = pathlib.join(path, entry.name)
				if entry.type == "dir" then
					fslib.removeDirectory(entryPath, { recursive = true })
				else
					fslib.remove(entryPath)
				end
    end
    ...
end
```

would become:
```luau
function fslib.removeDirectory(path: Pathlike, options: RemoveDirectoryOptions?): ()
	...
    for _, {name = name, type = entryType} in entries do
				local entryPath = pathlib.join(path, name)
				if entryType == "dir" then
					fslib.removeDirectory(entryPath, { recursive = true })
				else
					fslib.remove(entryPath)
				end
    end
    ...
end
```

### Example 2
Abbreviated from [lute](https://github.com/luau-lang/lute/blob/8965cd90c1a23bc56f017eb0fc973a0e0c5e8291/lute/std/libs/json.luau#L115)
```luau
local buf = state.buf
local cursor = state.cursor
buffer.writeu8(buf, cursor, 0x22)
buffer.writestring(buf, cursor + 1, escaped)
buffer.writeu8(buf, cursor + 1 + sl, 0x22)
```

would become:
```luau
local {buf = buf, cursor = cursor} = state
buffer.writeu8(buf, cursor, 0x22)
buffer.writestring(buf, cursor + 1, escaped)
buffer.writeu8(buf, cursor + 1 + sl, 0x22)
```

## Design
This RFC proposes the following syntax to support table and array destructuring.

```luau
local {fieldA = fieldA, fieldB = fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
print(fieldA) -- prints 1
print(fieldB) -- prints 2
```
as well as `const`:
```luau
const {fieldA = fieldB, field = fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
print(fieldA) -- prints 1
print(fieldB) -- prints 2
```

This syntax is symmetric with table construction, so we get renaming of fields for free:
```luau
const {fieldA = bing, fieldB = bong} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
print(fieldA) -- nil
print(bing) -- 1
print(bong) -- 2
```

We should also support arbitrary levels of nesting:
```luau
const {fieldA = renamed, fieldB = {bing = bing, bong = bong}} = {"fieldA" = 1, "fieldB" = {"bing" = false, "bong" = true}, "fieldC" = 3}
print(renamed) -- 1
print(bing) -- false
print(bong) -- true
```
Destructuring expressions are processed from left to right, and outside in, so a field that introduces a fresh binding may get shadowed by a later destructuring expression.

Finally, one would expect to see table destructuring at other points where bindings get introduced too:
```luau
for k, {x = x, y = y} in {{x = "1", y = "1"}, {x = "2", y = "3"}, {x = "5", y = "8"}} do
    print(x, y) -- prints 1,1\n2,3\n,5,8\n
end

if local {x = x, y = y} = {x = 1, y = 2, z = 3} then
    print(x, y)
end

for _, {x, y} in {{"a", "b"}, {"c", "d"}, {"e", "f"}} do
	print(x,y)
end

```

We will also explicitly support array destructuring for symmetry with the above feature.

For example:
```luau
local {x, y, z} = {1,2,3,4,5}
print(x, y, z) -- prints 1, 2, 3

```

This feature will explicitly use lookahead in the lexer to disambiguate in this situation whether we are trying to parse a paren-less call to a function named `const` or a const destructuring. There is a possible performance penalty here, and we must pay special care to avoid providing poor error messages. In practice, calling a function named `const` is so unlikely that we should consider whether this regression is acceptable.

```luau
local const = function(x) end
const {x, y} -- equivalent to const({x, y})

const {x, y} = ... -- destructuring
```
In Luau, function calls may omit parenthesis, and are valid as both expressions and statements.
Without a glyph, we would have to pay for an explicit sweep forward until the end of the array, before deciding if we
are evaluating a function call or a structured binding. A glyph makes this explicit, and this RFC proposes the use of `.`, since this is the operator used for table accesses. The benefit of the glyph is that it a) nests extremely nicely, and b) can be given whatever grammar we want for it!


The semantics of structured bindings are as follows:
```luau
local {fieldA = fieldA, fieldB = fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
```
will effectively be:
```luau
local _rhs = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
local fieldA = rhs.fieldA
local fieldB = rhs.fieldB
_rhs = nil
```
Fields that do not exist will be bound to nil.

Nested accesses, such as:
```luau
local {fieldA = {A = A, C = C}} = {fieldA = {A = 1, B = 2, C = 3}}
```
will effectively be:
```luau
local _rhs = {fieldA = {A = 1, B = 2, C = 3}}
local A = rhs.fieldA.A
local C = rhs.fieldA.C
_rhs = nil
```

Finally, the semantics for destructuring arrays:
```luau
local {x, y, z} = foo()
```
will effectively be:
```luau
local _rhs = foo()
local x = _rhs[1]
local y = _rhs[2]
local z = _rhs[3]
_rhs = nil
```




## Drawbacks
The main drawback of this feature is that when examining a const binding, we will have to do lookahead to determine whether or not we are using an identifier named `const` as a paren-less function call:
```luau
const = function() end
const { x = 1, y = 2}
```
or as a structured binding:
```luau
const {x = x, y = y} = ...
```
This requires potentially unbounded lookahead, but once we balance out braces, once we see the first `equals` sign, we can disambiguate which of the above cases we are trying to parse. The main issue is here is increased lexing complexity, a more complicated parser, and a possible regression in error message quality. This will require more work to get right, and may cause users of `const` as an identifier to incur a performance penalty during parsing. However, we have determined that this is rather unlikely, and so by predictively parsing as a structured binding, we are likely to not have to roll back lexer state.


## Alternatives
One possible option is to use a glyph instead of introducing a new parsing rule that requires us to use unbounded lookahead:
```luau
const .{x = reboundX, y = reboundY, z = .{reboundU, reboundV}} = makeATable()
```

## Non-goals
This RFC explicitly does not propose extracting types from a table. This is because most tables in the language (aside from require, which is treated specially)
do not have any implicitly associated types. Thus, we will not support:
```luau
local {type x = x, ...} = require("./someModule")
```

## Prior Art
Table destructuring is a common feature in many languages - Javascript, Python, OCaml, Zig all support this.

### Javascript
[Syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring)
```JS
const [a, b] = array;
const [a, , b] = array;
const [a = aDefault, b] = array;
const [a, b, ...rest] = array;
const [a, , b, ...rest] = array;
const [a, b, ...{ pop, push }] = array;
const [a, b, ...[c, d]] = array;

const { a, b } = obj;
const { a: a1, b: b1 } = obj;
const { a: a1 = aDefault, b = bDefault } = obj;
const { a, b, ...rest } = obj;
const { a: a1, b: b1, ...rest } = obj;
const { [key]: a } = obj;
```

### Python
Python is slightly different in that destructuring and unpacking is only supported on lists/tuples, but not directly on
tables.
```python
head, *tail = [1, 2, 3, 4, 5]

print(head)  # 1
print(tail)  # [2, 3, 4, 5]


x,y = function () return 1,2,3 end
print(x,y) # 1,2
```

Python objects typically support a .values() method that allows you to get all of the values in a dictionary as a list.

### Zig
Zig uses list destructuring too. The `.` glyph is used for literals:
```zig
const tuple = .{ 1, 2, 3 };
x, var y : u32, const z = tuple;
```

### OCaml
```OCaml
let x, y = (1, 2)
type point = { x: int; y: int }
let p = { x = 10; y = 20 }
let { x; y } = p (* x is 10, y is 20 *)

(* Nested destructuring with renaming *)
type person = {
  name : string;
  street : string;
  city : string;
  zip : int;
  contact : string * int;
}
let bim = { name = "Bim", street = "Mission St", city = "San Francisco", zip = 1, contact = ("bim@bim.gmail.com", 1)}
let { name; street; city; contact = (email, phone) } = bim;;
```


