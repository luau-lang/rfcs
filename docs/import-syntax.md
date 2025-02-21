# Import Syntax

## Summary

This proposal introduces new syntax to import exported values and types from modules. This syntax pairs well with the existing `export type name = type` syntax and the proposed `export name = value` syntax. In a simple example: `import name, name2, type type1, type type2 from "module"`.

## Motivation

The last RFC to propose import syntax was rejected with the primary reason being that destructuring solved all of the same problems, and was a more expansive feature. At this point in time destructuring is pretty much dead in the water, with a destructuring RFC being rejected, and current RFC(s) in limbo.

The proposed import syntax enables programmers to control precisely what they bring into scope from a module. This is particularly useful with larger libraries, like react.

```luau
-- what was once this:
local React = require("@React")
local createElement, useState, useEffect, useContext, useRef = React.createElement, React.useState, React.useEffect, React.useContext, React.useRef
-- can now be this:
import createElement, useState, useEffect, useContext, useRef from "@React"
```

## Design

In EBNF:

```ebnf
chunk = imports block

imports = {import [';']}
import = 'import' {importname ','} [importname] 'from' STRING
importname = ['type'] NAME
```

Imports would only exist at the top of files, shebang and comments are ignored, and so would be allowed above and inside of imports. 

This import:

```luau
import createElement, useState, useEffect, useContext, useRef from "@React"
```

Would be equivalent to:

```luau
local React = require("@React")
local createElement, useState, useEffect, useContext, useRef = React.createElement, React.useState, React.useEffect, React.useContext, React.useRef
```

Types could also be imported, which was previously impossible:

```luau
import myvalue1, type mytype1, type mytype2 from "MyModule"
```

At runtime, type imports would be ignored.

The string after the `from` contextual keyword would be a string literal, and would follow the same rules for module resolution as the `require` function.

## Drawbacks

This gives users a second way to import modules, which could be confusing to new users and could lead to inconsistent codebases. This would already be the case with the proposed export-value syntax.

## Alternatives

The primary alternative is to continue using the existing `require` function.

Some other alternatives are:

* Change the contextual keyword `from` to the existing reserved keyword `in`. This may reduce parsing complexity, but would be less intuitive.
* Allow programmers to rebind imported values to other names, perhaps something like `import createElement as e from "@React"`. This would be more flexible, but would also be more complex. This pattern could also not be done with autocomplete, as the autocomplete would not know the new name. It is also a fairly uncommon pattern, existing in usage of only a few libraries. Those users could continue to rebind with a `local` statement.
* Figure out destructuring syntax, and use that instead. This would be a more expansive feature, and would be more powerful, but would also be more complex. This would also be a more common pattern, and would be more intuitive to users coming from other languages.
