# Short Statements

## Summary
A simpler, smaller, statements for one-liners.

## Motivation

The current syntax for statements makes one-liners cluttered.

```js
while(true) break;
for(true) break;
if(true) console.log('Hello World!');
// C-Like languages allow the usage of semicolons to replace brackes.
// It's short and simple.
```

```lua
while true do break end
for true do break end
if true then print('Hello World!') end
-- these is very large, the (end) isn't really needed.
```

## Design

Statements accept a newline as an end statement.

```lua
while true do break
for true do break
if true then print('Hello World!')
```

## Drawbacks

None that are obvious.

## Alternatives

Not doing one-liners.

## Prior Art

As said earlier, C-like languages have this behavior.
```c
if(true) printf("Hello World!\n");
```

Pascal (inspiration for Lua's syntax) also allows this.
```pascal
if true then writeln('Hello World!');
```