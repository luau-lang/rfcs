# Type Constraints on Generic Functions

## Summary

Modify the current generics implementation to allow generic types to be constrained by a type.

## Motivation

When writing generic functions currently, it is as if you were simply using any unless you make your own type, being that the generic type has no constraints.

Take this snippet for example:

```luau
type Entity = {
    -- ...
}

type Gun = {
    IsEmpty: (self: Gun) -> boolean,
    FireGun: (self: Gun) -> (),
    -- ...
}

type WeaponResult = {
    HittedEntity: Entity,
    DamageDealt: number,
}

type ShotgunWeaponResult =  {
    PelletCount: number,
    AmountOfPelletsHit: number,
} & WeaponResult

type Shotgun = {
    -- ...
} & Gun

local function fireGun<T: Gun, U: WeaponResult>(self: T): U?
    if not self:IsEmpty() then        -- No error
        return self:FireGun()         -- No error
    end
    
    return nil    
end 

local gun: Gun
local shotgun: Shotgun

local fireResult = fireGun(gun) :: WeaponResult                 -- No error
local shotgunFireResult = fireGun(shotgun) :: ShotgunWeaponResult             -- No error

fireGun("Hello, world!")     -- Type Error: type 'string' and 'Gun' are not related to one another.
```

On this snippet, we are trying to implement a theoretical Gun system, we may have many implementations for the Gun, like a Shotgun or a Pistol. In this implementation, we want to make a generic function that can take a gun of type `T` constrained by the base `Gun` type. But we also want to be able to return a result which is constrained by the type `WeaponResult`, however, the current system for generics does not allow for this without getting type errors.

This is the main reasoning behind this modification. Many other languages have a feature to limit what types does a generic allow as a baseline. Languages like C# have this implemented on the form of the `where T : BaseClass` syntax, which delimits the type that `T` can be or inherit from. It supports cases in which you may want to use a strucutre, but not know its implementation details, like an interface.

However, we can use this pattern to improve the reliability of code, a function such as the following that takes a string and returns a type, without having to type cast into `any` before type casting into `T`.


```luau
type GameService = {
    -- ...
}

type GameManager = {
    Disconnect: (self: GameManager) -> (),
    -- ...
} & GameService

type ServiceContainer = {
    ServiceList: { [string]: GameService },
    Require: <T: GameService>(self: ServiceContainer, serviceName: string) -> T
}

local ServiceContainer = {
    Require = function<T: GameService>(self: ServiceContainer, serviceName: string): T 
        return self.ServiceList[serviceName] :: T
    end
}

-- ...

local GameManager = ServiceContainer:Require<GameManager>("@game/GameManager")
GameManager:Disconnect()
```

## Design

The design of this feature will reuse the already existing generics syntax, with little modification.

```luau
type Entity = {
    -- ...
}

type Gun = {
    IsEmpty: (self: Gun) -> boolean,
    FireGun: (self: Gun) -> (),
    -- ...
}

type WeaponResult = {
    HittedEntity: Entity,
    DamageDealt: number,
}

type ShotgunWeaponResult =  {
    PelletCount: number,
    AmountOfPelletsHit: number,
} & WeaponResult

type Shotgun = {
    -- ...
} & Gun

local function fireGun<T: Gun, U: WeaponResult>(self: T): U?
    if not self:IsEmpty() then        -- No error
        return self:FireGun()         -- No error
    end
    
    return nil    
end 

local gun: Gun
local shotgun: Shotgun

local fireResult = fireGun(gun) :: WeaponResult                     -- No error
local shotgunFireResult = fireGun(shotgun) :: ShotgunWeaponResult   -- No error

fireGun("Hello, world!")     -- Type Error: type 'string' and 'Gun' are not related to one another.
```

The syntax for defining a generic function would remain the same, except, instead of simply taking a `T`, we would take a `T : TypeConstraint`, the syntax remains similar to other features of the language, and it would work by checking if the type that `T` is inferred to be has any kind of relationship with `TypeConstraint`. However, we may also want to be explicit when we are doing generics on this way, therefore a syntax to properly and explicitly specify which types the function takes in its current call would be preferred, this way, we no longer have to use turbofish to type cast into the correct return type `U` constrained by `WeaponResult`.

```luau
local fireResult = fireGun<Gun, WeaponResult>(gun)
```

This would allow us to be explicit with what types our generic function takes when calling it, improving overall reliability for the types we write, as they would no longer be an inferred `any` that cannot be type-casted using turbofish. This also allows for more flexible coding styles, and to implement custom require-like functions without them returning `any` and requiring an explicit type cast. This would also be enforced by the type checker, making it so if `GameManager` is not related to `GameService` it warns or errors at the programmer for it, like any other type safe language.

```luau
type GameService = {
    -- ...
}

type GameManager = {
    Disconnect: (self: GameManager) -> (),
    -- ...
} & GameService

type ServiceContainer = {
    ServiceList: { [string]: GameService },
    Require: <T: GameService>(self: ServiceContainer, serviceName: string) -> T
}

local ServiceContainer = {
    Require = function<T: GameService>(self: ServiceContainer, serviceName: string): T 
        return self.ServiceList[serviceName] :: T
    end
}

-- ...

local GameManager = ServiceContainer:Require<GameManager>("@game/GameManager")
GameManager:Disconnect()
```

This also allows the return of `ServiceContainer:Require<T>("...")` to accidentally not be casted into something unrelated.


## Drawbacks

The main deterrant to implementing type constraints on generics is type intersection. Type intesection allows us to implement this snippet without fighting much of anything relating to the type system.

```luau
type Gun = {
	IsEmpty: (self: Gun) -> boolean,
	FireGun: (self: Gun) -> (),
	-- ...
}

type Shotgun = {
	-- ...
} & Gun

local function fireGun(self: Gun)
	if not self:IsEmpty() then
		self:FireGun()
	end
end

local gun: Gun = nil
local shotgun: Shotgun = nil

fireGun(gun) 		        -- Works as intended.
fireGun(shotgun) 	        -- Works as intended.
fireGun("Hello, world!")    -- Type Error: type 'string' could not be converted into 'Gun'

```

Using type intersection would not require any changes to generics, and makes the function take an input that is of any type, as long as the type is intersected with, this solution is type safe and does not require any modification to the current way generics work.

This is one of the cases in which type constraints on generic parameters would have no significant benefit to the language, and may complicate some aspects, as it is another way of achieving the same end goal: Enforcing a certain type as a parameter.

## Alternatives

The unspoken of alternative to this, is simple type intersection, yet another reason as to why this should be implemented is that the future of `Luau` is bleeding more into a safer language in what types respect to, by adding this feature we are improving an existing one which was added with such idea in mind, improving it by adding something that was originally not thought of and that could overall improve the current implementation for generics, and allow for more flexible patterns when programming with Luau.

The design given on the Drawbacks section is indicative of another way of designing the system so that it plays nicely with the current type checker, however, this leads generics in a place where they have nowhere to belong to.

However an implementation that could benefit of the enforcing type constraints on generic functions is that of the require-like function, a possible implementation without type constraints would be marginally less type safe, as we would need to first cast into any to do any conversion into the type `T`. In the case this can be avoided, there could be room for improvement on Luau internals in what relates to native code generation, allowing it to generate performant native versions for functions that fit the type constraint, which could be benefitial overall.

With it aside, it could also allow for factories such as `Instance.new("ProximityPrompt")` to be improved and be more explicit, into a syntax like `Instance.new<ProximityPrompt>()`, and instead of using a string, it could use a type, enforced by the type checker, and not allowing users to make silly mistakes such as adding an incorrect `string` and at runtime causing a problem.
However, a modification as massive as this one not make it, as it would essentially modify this specific function deeply, and break a lots of code, and the the C API would need to be modified to allow types to be read at runtime for fields, methods and names, essentially adding reflection, something that even though can be highly useful, would increase the complexity of the VM, the compiler and every aspect of the language as well as potentially decreasing performance.