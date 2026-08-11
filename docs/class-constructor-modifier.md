# Constructor Modifier

## Summary

To provide constructors that are compatible with existing Luau conventions, this RFC proposes the following syntax:

```luau
class Car
	public IsDestroyed: boolean
	public Speed: number

	constructor function new(speed: number)
		self.IsDestroyed = false
		self.Speed = speed
	end
end
```

This proposal introduces a `constructor` modifier, similar to the `public` modifier.

## Motivation

Using the `__init` metamethod as a constructor may be difficult for beginners, since the connection between `__init` and `new` is not immediately intuitive.

On the other hand, abandoning the long-standing Luau convention of constructors such as `Class.new()` and only allowing classes to be instantiated through function-call syntax like `Class { prop = a }` would also be inconsistent with existing conventions.

Furthermore, `Class.new()` is not the only constructor naming convention in Luau. 
Native objects also use names such as `create`, as seen in `buffer.create()` and `vector.create()`.

For this reason, forcing all constructors to be named `new` would not be appropriate either.

Since Luau classes are being designed with inheritance in mind, constructors of superclasses must also be callable.

Therefore, each class should have exactly **one** constructor, while the **name** of that constructor should be customizable.
In most cases, `new` would be the preferred constructor name, but it should be possible to use another name, such as `create`, when appropriate.

## Design

Inside a constructor function, `self` is implicitly available, and the constructed instance is returned even without an explicit `return self`.

```luau
class Car
	public IsDestroyed: boolean
	public Speed: number

	constructor function new(speed: number)
		self.IsDestroyed = false
		self.Speed = speed
	end
end

const myCar = Car.new(50)
```

If inheritance is added in the future, `super` could also be implicitly available as the superclass constructor and could be called directly.
`super` refers to the superclass's constructor regardless of that constructor's declared name.

```luau
class Car
	public IsDestroyed: boolean
	public Speed: number

	constructor function new(speed: number)
		self.IsDestroyed = false
		self.Speed = speed
	end
end

class ElectricCar extends Car
	public Capacity: number

	constructor function new(capacity: number)
		super(math.max(50 - capacity * 2, 0))
		self.Capacity = capacity
	end
end
```

If a different constructor name is desired, it can be specified using `constructor function <name>`:

```luau
-- When following the flatcase naming convention used by Luau's native objects,
-- constructors are commonly named `create`
class nativecar
	public isdestroyed: boolean
	public speed: number

	constructor function create(speed: number)
		self.isdestroyed = false
		self.speed = speed
	end
end
```

Each class may have only one constructor.

If the superclass and subclass use different constructor names, only the subclass's constructor should be available on the subclass.

## Drawbacks

Although there are several constructors that do not use the name `new`, such as `buffer.create` and `vector.create`, these are relatively uncommon cases. 
Most classes are instantiated using either `Class { prop = a }` or `Class.new(a)`.

Allowing the constructor name to vary also means that consumers must know the constructor name of a particular class in order to instantiate it. 
As a result, constructors would not provide a common naming contract across classes.

This design also differs from the convention inherited from Lua where special behavior is defined using names such as `function __*()`. Unlike `__init`, a `constructor` modifier may therefore introduce some ambiguity.

The `self` and `super` values inside a `constructor function` are implicit. 
Having undeclared variables implicitly injected into a function may make it harder for beginners to understand where they come from.

Additionally, Luau classes currently favor explicitly declaring `self`, so making it implicit only for `constructor function` may feel inconsistent.

## Alternatives

If `private` is introduced, constructors could instead follow a rule based on `__init`, with `new` being the default constructor, while still allowing classes to expose differently named constructors or prevent callers from invoking `new` directly.

Using `buffer` as an example, `__init` could be declared as a private function to prevent external consumers from accessing the default `new` constructor. 
A differently named constructor such as `buffer.create` could then be implemented as a static function, and additional constructors such as `fromstring` could be provided in the same way.

```luau
class buffer
	size: number
	data: string

	private function __init(size) -- Indicates that `new`, backed by `__init`, cannot be called externally.
		self.size = size
		self.data = ""
	end

	function create(size)
		return buffer.new(size)
	end

	function fromstring(str: string)
		const self = buffer.new(#str)

		self.data = str

		return self
	end
end

buffer.create(10) --> ok
buffer.fromstring("hello") --> ok
buffer.new(5) --> error: this class cannot be constructed externally using `new`
```

If the distinction in the design above is not clear enough, a `static` modifier could be introduced.

Another alternative is to continue defining constructors separately as static functions, as they are now, while adding a way to prevent external code from using `class {}` to instantiate the class.
