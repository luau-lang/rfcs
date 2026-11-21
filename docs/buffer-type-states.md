# Buffer type states

## Summary

Implement static semantics to validate buffers by their total size using generic's syntax.

## Motivation

As of now, you can only distinguish types, such as tables, by their indexer and key types.
However, having two objects that point to a single uniform type, say buffer or number, is too imprecise.

Take the following example with the buffer type :

```luau
type object1 = buffer -- (id: u16, p: (f32,f32), v: (f32,f32)), 18 bytes.
type object2 = buffer -- (p: (f32,f32), v: (f32,f32)), 16 bytes.

type table1 = { [number]: object1 }
type table2 = { [number]: object2 }
```

table1 and table2 are the same type alias in this scenario even if they are different.

## Implementation

### Syntax and semantics

A typed buffer will be instantiated using the following :

```luau
type foo = buffer<16>
```

The value between the generic brackets will define the total size of said buffer's contents in bytes.

This specific syntax is backwards compatible with previous code.
Perhaps, the type errors for said issues would be in the following format :
  > TypeError: expected buffer of size <n>, but got size <n2>.

Here is a basic segment of code that utilizes buffer type states :

```luau
type foo = buffer<4>

const function read(b: foo): number
    return buffer.readu16(b, 3) -- Flags a TypeError, readu16 is not possible on a 4 byte sized buffer at offset 3. 
end
```

You can also unionize them, so only the maximum size is taken into play.
The minimum size is for safe static reads whereas the maximum size is for memory allocation
(Reading u32 requires that the minimum union byte size is 4, and the maximum byte size is for allocation).

```luau
type foo = buffer<4> | buffer<8>
```

### Generics

Interaction with generics is possible. There are 2 rules.
  > The generic in place must be constrained to a valid buffer size literal.
  > buffer<N> is a subtype of an unsized buffer, but the vice-versa is not true.
    Any input that expects a sized buffer requires a type refinement.

```luau
type Packet<T> = {
  header: buffer<4>,
  payload: buffer<T>,
}

type foo = Packet<14>
```

```luau
const function readu32<N>(b: buffer<N>): number
    if buffer.len(b) < 4 then error("size of buffer must be >4 bytes.") end
    return buffer.readu32(b, 0)
end

readu32(buffer.create(5)) -- Element size is inferred if created, or otherwise the generic is instantiated.
readu32<<7>>(buffer.create(5)) -- TypeError: cannot instantiate with 7 bytes; argument is buffer<5>.
```

As you can see, the issue is that there you cannot guarantee a n-sized buffer respects a specific constraint without :
  > Using runtime checks as shown above.
  > Using branding, which is shown in the ``Alternatives`` section.

Thus, a syntax could be implemented to describe buffers of >k or <k elements.
**buffer<n> represents a buffer of size n, but I propose the following semantics (syntax is up-to-change).
Where ! is the clamped range bound symbol, not the negation symbol.**

```luau
type foo<T> = buffer<T!> -- T! represents T sized and over, clamped between T and 1 GiB.
type bar<T> = buffer<!T> -- !T represents T sized and under, clamped between 0 and T.
```

At the end, typed buffers help differentiate two *similar* types that point to the same underlying type.
Which in this case is *buffer*.

### Parser & type solver

The size of a typed buffer during its instantiation will be treated as a singleton values.
Similar to constant values, the parser will directly replace the number with its direct number literal.
Bounds validation will be done during the type solver stage rather than the parsing stage, returning TypeError outputs.

## Drawbacks

The type solver cannot efficiently infer the minimum size of a generic at compile-time.
There must be constraints for valid buffer size literals, such as the size being more than or equal to zero, less than or equal to 1 GiB, no fractional numbers.
There will have to be updates to the current ``buffer`` library, updating the type definition for each method.

## Alternatives

People can use a branding system to the type at the cost of having to use a table instead, as shown in the example below :
```luau
type object3 = { b: buffer, brand: 8 }
```

Otherwise, no designs can be implemented and ambiguity will remain across type aliases with buffer.
