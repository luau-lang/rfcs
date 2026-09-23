# Table Destructuring
This RFC proposes a table destructuring syntax for Luau to make it convenient for Luau programmers to extract table keys.

## Summary
This RFC proposes a new syntax to support table destructuring.

Table Destructuring will support local bindings to a table.
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

Moreover, we will also support punning bound fields via `as` syntax:
```luau
const .{fieldA as bing, .fieldB as bong} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
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

Tables destructuring syntax will be explicitly disambiguated by a new glyph. We must do so, otherwise we risk breaking backwards compatibility. For example:
```luau
const = function(x) end
const {x, y} -- equivalent to const({x, y})
```

Without a glyph, we would have to pay for an explicit sweep forward until the end of the array, before being able to decide if we
are evaluating a function call or a binding. A glyph makes this explicit, and this RFC proposes the use of `.`, since this is the operator used for table accesses.


The semantics of structured bindings are as follows:
```luau
local .{fieldA, fieldB} = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
```
will effectively be:
```luau
local rhs = {"fieldA" = 1, "fieldB" = 2, "fieldC" = 3}
local fieldA = rhs.fieldA
local fieldB = rhs.fieldB
```
fields that do not exist will be bound to nil.

Nested accesses, such as:
```luau
local .{fieldA as .{A, C}} = {fieldA = {A = 1, B = 2, C = 3}}
```
will effectively be:
```luau
local rhs = {fieldA = {A = 1, B = 2, C = 3}}
local A = rhs.fieldA.A
local C = rhs.fieldA.C
```

Finally, destructured tables will compose neatly with the `if local` feature:
```luau
if local .{x , y} = foo() then
end
```

In this case, the runtime here will evaluate `foo()` and bind the values in the return table to the identifiers `x` and `y` in the then block.


## Non-goals
This RFC explicitly does not propose a destructuring syntax for 'arrays'. Luau notionally supports arrays - these are effectively
tables with numeric keys:
```
local x = {"a", "b", "c"} -- { [1] = "a", [2] = "b", [3] = "c"}
```
The runtime provides `table.unpack`, which lets you destructure arrays as:
```luau
local a,b,c = table.unpack(x)
```

### Motivation
An extremely common use case here is namespacing requires

```luau
local .{foo, bar, baz as bim} = require("../A")
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
Zig uses list destructuring too:
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


