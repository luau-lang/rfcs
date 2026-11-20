# Buffer type states

## Summary

Implement static semantics to validate buffers by their total size using generic's syntax.

## Motivation

As of now, you can only distinguish types, such as tables, by their indexer and key types.
However, having two objects that point to a single uniform type, say buffer or number, is too imprecise.
A number can be a

Take the following example with the buffer type :

```luau
type object1 = buffer -- (id: u16, p: (f32,f32), v: (f32,f32)), 18 bytes.
type object2 = buffer -- (p: (f32,f32), v: (f32,f32)), 16 bytes.

type table1 = { [number]: object1 }
type table2 = { [number]: object2 }
```

table1 and table2 have the same type definition in this scenario even if they are different.

## Implementation

A typed buffer will be instantiated using the following :

```luau
type b = buffer<16>
```

The value between the generic brackets will define the total size of said buffer's contents in bytes.

This specific syntax also has no way to break existing code.
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
The minimum size is for safe static reads whereas and the maximum size is for memory allocation.

```luau
type foo = buffer<1> | buffer<2> | buffer<3>
type bar = buffer<1>
```

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
    if buffer.len(b) >= 4 then error("size of buffer must be >4 bytes.") end
    return buffer.readu32(b, 0)
end

readu32(buffer.create(5)) -- Element size is inferred if created, or otherwise the generic is instantiated.
readu32<<7>>(buffer.create(5)) -- TypeError: cannot instantiate with 7 bytes; argument is buffer<5>.
```

As you can see, the issue is that there you cannot guarantee that a n-sized buffer respects a specific constraint without :
  > Using runtime checks as shown above.
  > Using branding, which is shown in the Alternatives section.
Otherwise, the type solver can infer what N's bounds are by looking at the function body.
Thus, a custom syntax could be implemented to describe buffers of >k or <k elements.

At the end, typed buffers help differentiate two *similar* types that point to the same underlying type.
Which in this case is *buffer*.

## Drawbacks

* The type solver cannot efficiently infer the minimum size of a generic at compile-time.
* Constraints for valid buffer size literals.
  > bigger or equal to zero, smaller or equal to than 1 GiB, no fractional numbers.
* Complications to ensure type checking with buffer library methods.
  > having a buffer of size n, and reading / writing at offset n+1.

## Alternatives

* Use a branding system to the type at the cost of having to use a table instead, as shown in the example below :
  > `type object3 = { b: buffer, brand: 8 }`

