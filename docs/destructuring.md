# Table Destructuring
This RFC proposes (yet another) table destructuring syntax for Luau to make it convenient for Luau programmers to extract table keys.

## Summary
Table destructuring is a common ask for Luau of how often programmers interact with tables. It is a convenience feature, intended to make it syntactically easy to write programs that unpack table values in a single line of code. Prior RFC's have suggested similar syntax, but these have often suffered from excess verbosity or would break backwards compatibility. This RFC
suggests a third way forward.

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
    for _, .{name, type as entryType} in entries do
				local entryPath = pathlib.join(path, entry)
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
local .{buf, cursor} = state
buffer.writeu8(buf, cursor, 0x22)
buffer.writestring(buf, cursor + 1, escaped)
buffer.writeu8(buf, cursor + 1 + sl, 0x22)
```

## Design
This RFC proposes the following syntax to support table destructuring.
```luau
local .{fieldA, fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
print(fieldA) -- prints 1
print(fieldB) -- prints 2
```
as well as `const`:
```luau
const .{fieldA, fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
print(fieldA) -- prints 1
print(fieldB) -- prints 2
```

Moreover, we will also support punning fields via `as` syntax:
```luau
const .{fieldA as bing, fieldB as bong} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
print(fieldA) -- nil
print(bing) -- 1
print(bong) -- 2
```

We should also support arbitrary levels of nesting:
```luau
const .{fieldA as bing, fieldB as .{bing, bong}} = {"fieldA" = 1, "fieldB" = {"bing" = false, "bong" = true}, "fieldC" = 3}
print(fieldA) -- nil
print(bing) -- false
print(bong) -- true
```
Finally, one would expect to see table destructuring at other points where bindings get introduced too:
```luau
function foo(.{x, y} : ...)
    print(x,y)
end

for k, .{x, y} in {{x = "1", y = "1"}, {x = "2", y = "3"}, {x = "5", y = "8"}} do
    print(x, y) -- prints 1,1\n2,3\n,5,8\n
end
```

Tables destructuring syntax will be explicitly disambiguated by a new glyph. We must do so, otherwise we risk incurring excess backtracking and ambiguity in the grammar. The classic example here is a function named const:
```luau
local local = function(x) end
const {x, y} -- equivalent to const({x, y})
```
In Luau, function calls may omit parenthesis, and are valid as both expressions and statements.
Without a glyph, we would have to pay for an explicit sweep forward until the end of the array, before deciding if we
are evaluating a function call or a structured binding. A glyph makes this explicit, and this RFC proposes the use of `.`, since this is the operator used for table accesses. The benefit of the glyph is that it a) nests extremely nicely, and b) can be given whatever grammar we want for it!


The semantics of structured bindings are as follows:
```luau
local .{fieldA, fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
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
local .{fieldA as .{A, C}} = {fieldA = {A = 1, B = 2, C = 3}}
```
will effectively be:
```luau
local _rhs = {fieldA = {A = 1, B = 2, C = 3}}
local A = rhs.fieldA.A
local C = rhs.fieldA.C
_rhs = nil
```

Finally, destructured tables will compose neatly with the `if local` feature:
```luau
if local .{x , y} = foo() then
end
```
In this case, the runtime here will evaluate `foo()` and bind the values in the return table to the identifiers `x` and `y` in the then block.

This RFC also proposes that the grammar for structured imports reserves `type ident` as a possible assignee. This is to make it convenient to work with associated table types / requires:
```luau
const .{type X, type Y as Z} = require("./A")
const .{foo, type X, type Y as Z} = require("./A")
```
Type bindings can also be elided, but will desugar to the equivalent table access used to define types:
```luau
local _rhs = require("./A")
type X = rhs.X
_rhs = nil
```

## Drawbacks
Most features take on implementation drawbacks, but in this case, the implementation is quite straightforward. The main drawback here is the syntax - it might be that the glyph is just too ugly and it would make the language worse.


## Alternatives
One possible option is to co-opt the same syntax used by tables instead of introducing a new `as` keyword:
```luau
const .{x = reboundX, y = reboundY, z = .{reboundU, reboundV}} = makeATable()
```
OCaml uses this approach too, but a reasonable objection might be that assignment *to* usually occcurs on the lhs. 
Moreover, the glyph means we might be able to avoid nested `.` inside of this destructuring statement, if that is also
too verbose.
If we end up not hating the glyph approach, we could also consider extending this syntax to `[]` for sugar over unpack:
```luau
const .[x, y] = {1, 2}
```

This RFC is written assuming that incurring the backtracking is untenable and that we'd need to design around this. As always,
we could just do nothing.


## Non-goals
This RFC explicitly does not propose a destructuring syntax for 'arrays'. Luau notionally supports arrays - these are effectively
tables with numeric keys:
```luau
local x = {"a", "b", "c"} -- { [1] = "a", [2] = "b", [3] = "c"}
```
The runtime provides `table.unpack`, which lets you destructure arrays as:
```luau
local a,b,c = table.unpack(x)
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


