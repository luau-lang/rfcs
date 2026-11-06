# Do Not Implement JSX

## Summary

We will not be adding JSX to Luau.

Luau is a much more flexible language for building tiny, useful domain-specific languages than JavaScript and does not share the problems that make JSX necessary there.

## Motivation

JSX has been proposed for Luau a number of times.  It's a very useful feature for React developers using JavaScript and Roblox itself uses a port of React to Luau, so there's a natural inclination to port JSX over as well.

In JS, a JSX expression like `<Component tag=value>child</Component>` desugars to something like `createElement(Component, {tag: value}, ["child"])`.  A typical React application needs to create a lot of components and arrange them in a complex tree, so the XML-like syntax does a lot to make things easier to read.

Things are different in Luau, however, for two reasons:

First, Luau allows a function call to omit the enclosing parentheses if the argument is a string literal or a table literal expression.  For example, the following two statements are completely equivalent:

```luau
foo({a=1, b=2, c=3})
foo {a=1, b=2, c=3}
```

Secondly, and more importantly, a Luau table can simultaneously hold indexed (array-like) elements and values mapped to keys.  This makes it very easy for a Luau API to combine properties and children into the same construct.

```luau
function render()
    paragraph {
        width="100%",
        text_size="24px",

        "Lorem Ipsum ",
        italics {"dolor sit "},
        "amet"
    }
end
```

If mixed tables are not desired, an API author could also choose to offer an interface atop curried functions.  In the following example, the `paragraph` function itself returns a function which is passed a child array:

```luau
function render()
    paragraph { width = "100%", text_size = "24px" } {
        "Lorem ipsum ",
        italics { "dolor sit " },
        "amet"
    }
end
```

We think this is good enough that a second syntax would be mostly redundant and not worth the maintenance burden.

## Conditions to Reconsider

The conditions that might cause us to reconsider this position are a longshot: paren-less function calls and mixed array-and-dictionary tables are very old features that have been part of Lua for decades.  We'd never consider breaking these features and so it seems unlikely that Luau would ever grow less capable for these sorts of domain-specific languages.

There would have to be some use case where actual XML syntax shows some clear benefit that could never be realized any other way.
