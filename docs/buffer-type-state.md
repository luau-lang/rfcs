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

The value between the generic brackets will define the total size of said buffer's contents.
This definition can also be further expanded with numbers, tables.
Take a look at this example :

```luau
type t<T,K> = { T }<K>
```
```luau
type n<T> = number<T>
```

This specific syntax also has no way to break existing code.
Perhaps, the type errors for said issues would be in the following format :
  > TypeError: expected <variable> to be of type <t> and of size <n>, but got type <t2> and size <n2>

## Drawbacks

* Complex for the type solver to check if a buffer adheres to the specific restrictions including padding for memory alignment.
* Complex for the type solver to ensure the following prerequisites are followed:
  > bigger or equal to zero, smaller or equal to than 1 GiB, no fractional numbers.
  > for numbers, implementing exponential steps (2^0 byte, 2^1 bytes, 2^2 bytes).

## Alternatives

* Use a branding system to the type at the cost of having to use a table instead, as shown in the example below :
  > `type object3 = { b: buffer, brand: 8 }`

