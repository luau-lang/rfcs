
# Import Syntax

This RFC proposes the introduction of dedicated syntax for `import` - a feature that will enable Luau programmers to declare the set of modules their script depends on. This builds on the previous rfc proposed [here](https://github.com/luau-lang/rfcs/pull/103) by u/JackHexed.

## Motivation

Luau supports modularity via `require`, which is an embedder defined function that resolves the location of modules, evaluates them, and returns their exports (usually a table). However, we afford no syntactic conveniences for unqualified imports, requiring programmers to instead qualify their imports everywhere, either at the point of use, or at the top of the file.  

For example, here is some real code (abbreviated) from [lute](https://github.com/luau-lang/lute/blob/primary/lute/cli/commands/test/reporter.luau):

```luau
local rt = require("@batteries/richterm")
local path = require("@std/path")
local tys = require("@std/test/types")
local snapty = require("./snap/types")

local pass = rt.combine(rt.green, rt.bold)
local fail = rt.combine(rt.red, rt.bold)
local errlabel = rt.yellow
local separator = rt.dim  
  
local function printFailure(f : tys.FailedTest) end  
local function snapReporter(result: snapty.SnapResult)
```

Under the design proposed by this rfc, this might end up looking like:

```luau
import combine, 
       red, green, bold,  
       yellow as errlabel, dim as separator from "@batteries/richterm"  
import format from "@std/path"  
import type FailedTest from "@std/test/types"
import type SnapResult as SnapshotTestResult from "./snap/types"  
  
local pass = combine(green, bold)  
local fail = combine(red, bold)  
  
local function printFailure(f: FailedTest) end  
local function snapReporter(result : SnapshotTestResult) end 
```

## Design

This RFC proposes the introduction of three new keywords, `import` , `from`, and `as` .
- `import` is followed by the thing or list of things you want to import
- `from` is followed by a string literal, denoting the require-by-string path
- `as` is used to re-bind an imported item

### Syntax Summary
```luau
import * from "./A"
import * as B from "./A"

import type A from "./A"
import type A as from "./A"
import type A, type B, type C as D from "./A"

import f1 from "./A"
import f1 as f2 from "./A"
import f, g as h, u as v, type A, type B as C from "./A"
```

 Assume all the following examples can see this code snippet as a sibling module, colocated in the same directory:

```luau  
-- A.luau
export const foo = "foo"  
export function bar()  
     print("Hello World")
end
export local bim = true  
export type Wow = number | typeof(bim)  
export type Ok = string | typeof(bar)
```

### Unqualified Imports

```luau  
-- Module main.luau
import foo, bar from "./A" 
```

Will be equivalent to:

```luau
const foo = require("./A").foo  
const bar = require("./A").bar
```

### Rebinding Unqualified Imports

```luau
-- Module main.luau
import foo as constantFoo, bar as printer from "./A" 

printer() -- prints "Hello World"  
print(constantFoo) -- prints "foo"

print(foo) -- prints nil
print(bar) -- prints nil
```

Will be equivalent to:

```luau  
const A = require("./A")
const constantFoo = require("./A").foo  
const printer = require("./A").bar  
  
printer()  
print(constantFoo)  

print(foo)  
print(bar)
```

### Type Imports

Type imports can exist in the same statement as regular import statements and will support all the standard rebinding we expect.
```luau
-- Module main.luau  
import foo, type Wow, type Ok as AliasedOk, from "./A  
```

Will be equivalent too:

```luau  
const foo = require("./A")  
type Wow = require("./A").Wow  
type AliasedOk = require("./A").Ok
```

### Wildcard Imports

```luau
-- Module main.luau
import * from "./A"
```

Equivalent to:

```luau  
const foo = require("./A").foo  
const bar = require("./A").bar  
const bim = require("./A").bim  
  
type Wow = require("./A").Wow  
type Ok = require("./A").Ok
```



Finally, we'll also support requalifying the wildcard import via `as` syntax:

```luau
--- Module main.luau  
import * as B from "./A"
```

Equivalent to:

```luau
const B = require("./A")
```

This RFC proposes the following restrictions on imports:
1) The string passed after `from` must be a static string literal. We will explicitly not support:  
```luau  
local A = "./Mod3"  
local pathC
import * from A  
import * from if math.random() then "./Mod1" else "./Mod2"  
import * from (function() return "./Mod4" end) ()
```

2) Imports at the top level are hoisted, so their effects occur at the beginning of the file.
3) Non top level imports are treated like dynamic requires.

Finally, imports that contain only types can be trivially elided, and imports that mix types and values, will only elide the types.

## Drawbacks
This RFC is explicitly designed with the idea in mind that we cannot support reserving keywords and that we must maintain backwards compatibility. Luau's existing "cool-call" (parenless function call) syntax means that:
```luau
import "..."
import { ... }
```
are off the table, since they are valid today as syntax.

The main implementation related drawback here is the additional work the compiler must be augmented to perform. Specifically, for top level imports, we can perform static require tracing and inlining of modules or even conspire for imported modules and imported subsets of modules to be cached in registers for faster access.

This comes with a downside that compilation would be less parallelizable, although this could be solved with a separate multi-file compilation API.

Another possible weakness of the proposed syntax is that the autocomplete experience within a namespace might not be as good. In fact, adopting a pythonic style like:
```python
from x import ....
```
means that we can provide recommendations for imports since we'll know which module the user is choosing to import from.

## Alternatives
This RFC proposes imports as a solution to the issue of overly verbose requires + as a pathway to performing more static analysis at compile time. Table Destructuring syntax can solve the first problem, and formally 'blessing' top level require-by-string with the same static compilation guarantees would solve the latter. As always, we could also not do any of this work.

## Prior Art
This section is explicitly focused on other dynamic languages. For the most part, the state of the art does involve unqualified
and renaming of qualified imports. While all the examples listed support wildcard imports, it's typically considered bad practice to use them as they mutate the global namespace.


### Python
```python
import math # imports a module and binds it to the name `math`
from math import sqrt # only binds sqrt to exported value from the math module
import math as builtin_math # requalifies the math import
from math import s
```

### Javascript / Typescript
This syntax is more or less similar to ours. The main difference here is the presence of explicit table destructuring syntax, and additional details around how `module.exports` interacts with the wildcard operator.
