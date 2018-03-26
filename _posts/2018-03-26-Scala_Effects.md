---
title: Language-Level Effects
---
# Ideas for Language-Level Effects in Scala 

The scala language may get an effects system, it is the last thing on the list on the [dotty]() page.  I am speculating, but I think this will be based on passing values around that represent the _capability_ to cause the effect.  

For example, `println(x: String)` would be given an extra argument `println(x: String, e: IOEffect): Unit` so that the caller must provide an `IOEffect` value.  The caller is not supposed to manufacture the `IOEffect` or pluck an `IOEffect` out of a global variable. That would defeat the purpose.   For reasons that will become apparent, the caller must require an `IOEffect` among its own arguments.  

## Thunks and Monads

In practice the `IOEffect` would be passed implicitly like so: `println(x: String)(implicit e: IOEffect): Unit`.  This allows us to write an expression like `println("Hello World")` which does no printing but produces a _thunk_ or suspended action.  

The thunk is an immutable value that can be incorporated into other thunks of the same type or into data structures.  Its type is `implicit IOEffect => Unit`.  The corresponding type would be `IO[Unit]` if we were using a monad to represent IO effects.

The `IOEffect` value is required but it does not affect the behaviour of `println`.  As an optimization, it can be marked for erasure  like so: `println(x: String)(erased implicit e: IOEffect): Unit`.  When that optimization is applied throughout, the thunks are eliminated during compilation and the runtime costs of reifying effects should be all but eliminated.  

## Capabilities and Scopes

Suppose `println` is called by a larger method `print_report`.  The `IOEffect` capability should also be an argument of `print_report` whose signature would be `print_report(d: Data)(erased implicit e: IOEffect): Result`.  The expression `print_report(d)` produces a thunk of type `implicit IOEffect => Result`. The corresponding monadic type would be `IO[Result]`. 

It is essential that capabilities are injected as arguments (implicit or not).  If a method took a capability from the static scope it could produce the corresponding side effect directly.  Equational reasoning is defeated, types no longer reflect effects and we are no longer in the realm of functional programming.

To enforce discipline, capabilities must be second class values.  They may be passed as arguments but may not be captured in data structures or closures. More about this later.

At some point a capability must be manufactured and the thunks executed.  This point marks the outer boundary of an island of functional code, the "end of the world".  

## Mutation Capability

Mutation could also be controlled with a capability.  We can introduce `MuEffect` which would be required for assignment to a `var` and update of an `Array` element.

Using this capability, the type `implicit MuEffect => (S, A) => B` would correspond to the State monad, where `S` is some mutable state.  This is useful for mutable state in the large, where the state is affected by many parts of a program. 

The opposite situation is commonplace in scala, where a mutable value is confined to a single method or constructor.  For example, an Array might be used in the construction of an immutable collection.  Or an externally pure function might use a `var` internally.  Entirely local mutation should be allowed without exposing `MuEffect` in the method or constructor signature.  

The next section explores an alternative approach.

## Immutable Values

Which values are immutable in scala?  Every value of type `AnyVal` is immutable. `String`s are immutable. Every `Array` is _mutable_.  Every other value of type `AnyRef` is immutable if:

1. it has no `var` members
2. all of its `val` members are immutable
3. its methods do not close over any `var`s or mutable `val`'s

There are still more immutable values, such as `ImmutableArray` and `List` instances, which have `Array` and `var` members respectively but don't mutate them after construction.

This could be formalized using a marker trait, `Immutable`.  If a type `T` is a subtype  of `AnyVal` or `Immutable` then it is an immutable type.  A class or trait inheriting `Immutable` must conform with the three rules above or it must be annotated `@uncheckedImmutable`. 

The standard library type `collection.immutable.Iterable` should be a subtype of `Immutable`. Concrete types such as `ImmutableArray` and `::` (the `List` cons cell) should be annotated as `@uncheckedImmutable`.  Many other standard types would become subtypes of `Immutable`. 

## Pure Methods, Second Class Values and Functional Domains

The three rules above don't take into account side effects other than mutation.  For example, if a method reads a character and returns it then the object it belongs to cannot be considered immutable.  Another rule is needed for an immutable type:

 4. its methods do not close over any capabilities

This requires us to formalize a type for capabilities.  A capability has the marker type `Capability`. 

With that in place we can make some strong statements about a method of an immutable type:

- The type signature of the method determines if it is pure or not.  If the arguments are immutable the method implements a pure function.

- Mutable values and capabilities are second class values within the method.  They can be passed as arguments but they cannot be captured in the static scope.

A method of an immutable type that accepts a mutable value serves the same purpose as a State monad.  It captures a state-affecting operation which can be composed with others.  It confines the mutable state and can be executed within a pure function. 

A module, or global `object` that is marked `Immutable` creates a functional domain within which mutation and side-effects are controlled and can be reasoned about.

## Conclusion

The concept of immutable types outlined here is sufficient to provide an effects system and pure functions without any other change to the scala language.  Implicit functions are unnecessary for this but they make it easy to lift expressions into effects and to compose effects. 
