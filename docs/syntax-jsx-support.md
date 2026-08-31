# JSX syntax for Luau via `.luax` (experimental)

## Summary

This RFC proposes adding an **optional JSX-like syntax** to Luau, enabled only for files with a `.luax` extension and gated behind a feature flag, that desugars directly into existing Luau AST constructs (no separate transpile step).

## Motivation

JSX is a concise, widely understood syntax for describing UI trees. In Roblox’s ecosystem, UI is commonly built using React-like libraries (e.g. Roact/React), but Luau today requires verbose `createElement(...)` calls or helper builders.

This proposal aims to:

- Make UI code **more readable** and **more ergonomic**.
- Preserve Luau’s “no mandatory build step” workflow by implementing parsing in the language front-end.
- Avoid hard-coding any specific UI library into the language by desugaring to a **user-provided factory function**.

### Why JSX belongs at the language level (and is not “React syntax”)

A common concern is that JSX is “too specific” to React or UI frameworks to warrant native syntax. This RFC argues the opposite: **JSX is a general-purpose notation for declarative, structured function calls**, and the runtime meaning is entirely controlled by the `jsx` factory function chosen by the host/project.

In other words, JSX is not “React in the language”; it is:

- A compact syntax for constructing nested call trees (`jsx(tag, props, ...children)`).
- A convenient way to bind named parameters via record-like props (`{ key = value }`).
- A structured form that is easier for humans and tools to read than deeply nested function calls.

This makes JSX useful beyond UI, including data-driven scene graphs, entity graphs, or declarative configuration trees.

## Design

### Goals

- **No transpilation step**: JSX is parsed directly by Luau’s parser when enabled.
- **Opt-in**: no behavior change for `.lua` / `.luau` unless explicitly enabled.
- **Library-agnostic**: JSX lowers to an ordinary Luau call (`jsx(...)`) so projects can bind it to Roact/React/etc.
- **Minimize grammar ambiguity** with existing and future Luau syntax.

### Non-goals (initially)

These are intentionally excluded from the initial proposal to reduce ambiguity and implementation risk:

- Raw text children (e.g. `<Foo>hello</Foo>`)
- Fragments (e.g. `<>...</>`)
- Spread attributes (e.g. `<Foo {...props} />`)
- Namespaced attributes (e.g. `on:click=...`)
- JSX comments syntax

These can be added via follow-on RFCs once the core parsing approach proves robust.

### Opt-in surface area

**File extension**: `.luax` indicates JSX may appear in the file.

**Feature flag**: The syntax must be gated by a flag (e.g. `FFlag::LuauJsx`) so it can ship experimentally and be rolled out carefully.

### Syntax

When enabled, an additional `simpleexp` form is introduced:

- `jsxElement ::= '<' jsxTagName jsxAttributes ( '/>' | '>' jsxChildren '</' jsxTagName '>' )`
- `jsxTagName ::= Name { '.' Name }`
- `jsxAttributes ::= { jsxAttribute }`
- `jsxAttribute ::= Name [ '=' ( String | '{' expr '}' ) ]`
- `jsxChildren ::= { jsxChild }`
- `jsxChild ::= jsxElement | '{' expr '}'`

Notes:

- `{ expr }` is used for embedding arbitrary Luau expressions in children and attribute values.
- Attribute shorthand (`<Foo disabled />`) is allowed and is equivalent to `disabled={true}`.

### Desugaring

JSX constructs are desugared in the parser into an ordinary Luau function call:

#### Tag lowering

- If the tag begins with a lowercase identifier and has no dots (e.g. `<frame />`), it lowers to a string tag: `"frame"`.
- Otherwise it lowers to an identifier/dotted expression (e.g. `<Foo.Bar />` lowers to `Foo.Bar`).

#### Props lowering

Attributes lower into a table literal (record fields). If there are no attributes, props lower to `nil`.

#### Children lowering

Children lower into additional positional arguments after `(tag, props)`.

#### Examples

Self-closing:

```lua
return <Foo bar={x} />
```

Desugars to (conceptually):

```lua
return jsx(Foo, { bar = x })
```

Lowercase tag:

```lua
return <frame />
```

Desugars to:

```lua
return jsx("frame", nil)
```

Nested child:

```lua
return <Foo><Bar /></Foo>
```

Desugars to:

```lua
return jsx(Foo, nil, jsx(Bar, nil))
```

Boolean attribute shorthand:

```lua
return <Foo disabled />
```

Desugars to:

```lua
return jsx(Foo, { disabled = true })
```

### Runtime / library integration

Luau does **not** include React/Roact. JSX desugars to a call to a function named `jsx`. Projects can provide this in whichever way fits their runtime:

- `jsx = React.createElement`
- `jsx = Roact.createElement`
- `jsx = MyCustomElementFactory`

This keeps the language neutral and allows different runtimes to adopt the syntax without coupling Luau to any one library.

#### Intrinsics mapping for lowercase tags

Because lowercase tags lower to string values (e.g. `<frame />` → `jsx("frame", ...)`), most runtimes will want a small **lookup table** that maps these “intrinsic” names to the runtime’s native element types (for Roblox, typically Instance class names such as `"Frame"`).

For example:

```lua
local Intrinsics = {
    frame = "Frame",
    textlabel = "TextLabel",
    uilistlayout = "UIListLayout",
    -- ... (many more)
}

function jsx(tag, props, ...)
    if type(tag) == "string" then
        local className = Intrinsics[tag]
        assert(className, ("Unknown intrinsic tag %q"):format(tag))
        -- dispatch to your runtime using className
    else
        -- tag is a component (function/table/etc.) -> dispatch accordingly
    end
end
```

This approach is explicit and predictable, avoids ambiguous casing rules, and allows aliases (`div = "Frame"`) if desired.

### Example: binding JSX to Fusion (community framework)

Fusion is a popular community framework for Roblox that uses a declarative style for Instances and reactive state. Without changing the JSX proposal, a project can adopt JSX by binding `jsx` to a Fusion element factory.

In practice, Fusion code commonly looks like `Fusion.New("Frame"){ ... }` and uses special keys like `Fusion.Children` for child lists. The exact adapter shape may vary by project, but a representative sketch is:

```lua
local Fusion = require(Packages.Fusion)
local New = Fusion.New
local Children = Fusion.Children

local Intrinsics = {
    textlabel = "TextLabel",
    frame = "Frame",
    -- ...
}

-- One possible adapter shape:
--   jsx("frame", props, ...children) -> Fusion.New("Frame")(props)
--   jsx(FooComponent, props, ...children) -> FooComponent(props, ...children) (project-defined)
function jsx(tag, props, ...)
    props = props or {}
    props[Children] = { ... }

    if type(tag) == "string" then
        return New(Intrinsics[tag])(props)
    else
        -- component case is project-defined; this is just a sketch
        return tag(props)
    end
end

local State = Fusion.Value("Hello")

local ui =
    <textlabel
        Text={State}
        Size={UDim2.fromScale(1, 1)}
    />
```

This example illustrates the key point: **JSX is a “call tree” surface syntax**, and Fusion (not Luau) defines the semantics.

### Example: non-UI usage (scene graph / “fiber-like” tree)

JSX can also be used for non-UI declarative graphs, e.g. building a 3D scene tree out of Roblox Instances (a “fiber-like” idea: a declarative tree that produces runtime objects).

```lua
local Intrinsics = {
    model = "Model",
    part = "Part",
    pointlight = "PointLight",
    attachment = "Attachment",
    -- ...
}

-- A minimal Instance-producing factory:
function jsx(tag, props, ...)
    assert(type(tag) == "string", "This example expects intrinsic (lowercase) tags")
    local className = assert(Intrinsics[tag], ("Unknown intrinsic tag %q"):format(tag))

    local inst = Instance.new(className)
    props = props or {}

    -- Apply properties
    for k, v in pairs(props) do
        inst[k] = v
    end

    -- Parent children to this instance
    for _, child in ipairs({ ... }) do
        if typeof(child) == "Instance" then
            child.Parent = inst
        end
    end

    return inst
end

local scene =
    <model Name="Root">
        <part Name="Ground" Anchored={true} Size={Vector3.new(50, 1, 50)} />
        <part Name="Lamp" Position={Vector3.new(0, 5, 0)}>
            <pointlight Brightness={3.5} Range={24} />
        </part>
    </model>
```

This keeps the same core idea: the tag is a string, props are a table, and children are a list. The meaning is defined by the chosen runtime factory (`jsx`), and JSX is simply a concise way to express the tree.

### Compatibility & ambiguity analysis

#### Backwards compatibility

With `.luax` + flag gating, existing `.lua`/`.luau` source is unaffected.

Within `.luax`, code that uses `<` for comparisons (e.g. `a < b`) remains valid; JSX is only recognized when the parser expects a `simpleexp` and sees `<` as the next token in that position.

#### Grammar ambiguity concerns

Key ambiguity sources:

- `<` as binary operator vs JSX element start
- `</` sequences inside other contexts
- Interaction with existing type syntax, including explicit type instantiation (`<<T>>()`)

Mitigations:

- Restrict JSX recognition to positions where a `simpleexp` is parsed.
- Keep JSX syntax small initially (no raw text children, fragments, or spreads).
- Continue gating behind a flag to allow collecting real-world compatibility data.

### Editor integration & tooling considerations

Adding JSX affects:

- Tokenization/highlighting for `.luax`
- Parsing for autocomplete, formatter/pretty-printer, and incremental parse
- Error recovery expectations (JSX introduces new “paired delimiter” structures)

Mitigations:

- Keep JSX AST lowering to existing nodes so downstream tools operate on familiar AST shapes.
- Ensure error recovery always makes progress to avoid hangs in partial code states.
- Treat `.luax` as a distinct language mode/extension for editors.

### Security / sandboxing considerations

The syntax itself is not inherently unsafe, but it makes calling a factory function extremely easy. Sandboxing concerns are the same as for ordinary function calls:

- The host controls which globals exist (including `jsx`).
- Environments can restrict or replace `jsx` to safe implementations.

### Performance considerations

- Parsing adds new branches and lookahead, but is expected to be negligible compared to overall parsing cost.
- Desugaring into existing AST nodes avoids adding new runtime node kinds and keeps later stages (typechecking, codegen) unchanged.

### Status

**Implemented (flagged / experimental)** in this repository as a spike:

- `FFlag::LuauJsx` gating
- `.luax` extension used to enable `ParseOptions::allowJsx`
- Parser desugars JSX to `jsx(tag, props, ...children)`
- Parser tests cover collision edge cases and error recovery

This RFC is still required before any user-facing rollout.

## Drawbacks

- Adds a significant new surface area to the language syntax, which increases learning and maintenance cost.
- JSX has a history of subtle grammar ambiguities; even with careful gating, future syntax additions may conflict.
- Editor/tooling support must be updated (highlighting, formatting, autocomplete), and mismatches can degrade UX.
- The “factory function” convention (`jsx`) is another community contract to standardize (or allow configuring).

## Alternatives

1. **Do nothing**: keep using `createElement(...)` calls and helper builders.
2. **Library-level DSL**: implement an embedded DSL using Lua syntax only (functions/tables/metatables). This avoids parser changes but is typically more verbose and harder to read.
3. **Transpile step**: introduce a `.luax -> .luau` transformer. This conflicts with the “interpreted/no build step” constraint and complicates tooling.
4. **New AST node kinds**: keep JSX nodes in the AST and handle them in later passes. This can improve tooling fidelity but increases implementation complexity across compiler/typechecker/codegen and makes sandboxing harder to reason about.
5. **Configurable factory** (future): allow `--!jsxFactory=...` or similar to avoid requiring `jsx` to be global; still requires a syntax proposal, and introduces additional complexity in binding resolution.
