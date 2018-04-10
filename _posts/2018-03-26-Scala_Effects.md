---
title: Idea for Immutable Types and Language-Level Effects in Scala 
---

Rewritten 2018/4/9. Earlier version [here](https://github.com/arnolddevos/arnolddevos.github.io/blob/38538969525310d39a2e786f80707c86266f067d/_posts/2018-03-26-Scala_Effects.md).

## TL;DR 

If immutability could be represented in the scala type system it would also be possible to define pure function types and effect capabilities. No other changes to the language would be needed. 

The agenda is to put up a definition of _immutable_ that can be verified at the type level.  This leads to restrictions on references to mutable terms from the  body of an immutable type.  That restriction is extended in a natural way to capabilities which are values required to generate effects. 

It is seen that all methods of an immutable type must be pure or declare their effects in their type signatures. Methods of immutable types with effects can be lifted into immutable types, reifying their effects.  Implicit functions are the most convenient and efficient lifted types. It is seen that these lifted types are monadic. 

Finally, a module as a whole can be declared immutable.  The set of immutable modules and classes forms a functional domain separate from the rest of the program. Within this domain effects can be reasoned about at the type level. Outside this domain, effects can be executed.

## Immutable Values

Which values are immutable in scala?  Every value of type `AnyVal` is immutable. `String`s are immutable. But every `Array` is _mutable_.  Is an instance of a scala.collection.immutable class really immutable?  We assume it is iff its elements are immutable.  

A rough intuition is that a value is immutable if its behaviour observed through its public methods never changes.   Here is a definition: 

> a value is immutable if the result of each of its nullary methods is constant and the result of each method with immutable arguments is a function of those arguments.

That definition assumes "constant" and "function" have the usual functional programming interpretations. It also leaves open the possibility that a method of an immutable value could accept a mutable argument. The properties of these methods will be discussed later. 

Intuitively, a value that does not contain any mutable state will satisfy the definition (putting aside side effects such as I/O for now).  In other words: 

1. the value has no `var` members and all of its `val` members are immutable
2. the value does not close over any `var`s or mutable `val`'s

While those conditions are sufficient for immutability, they are not necessary.  For example each `ImmutableArray[Int]` contains a (mutable) `Array[Int]` and each non-empty `List[Int]` (as defined in the standard library) has a `var` member. These values are immutable by the proposed definition nevertheless. 

## Immutable Types

> A type is immutable if every value of that type is immutable (in the sense defined above).   

Not many types can be shown to be immutable by this standard.  For example, a class can only be immutable if it is `final`.  Otherwise a subclass could add mutable features.  

To make this more tractable, a marker trait, `scala.Immutable` is proposed.  If a type `T` is a subtype `Immutable` then it is asserted to be an immutable type.  That assertion must be proved during compilation or `T` must be annotated `@uncheckedImmutable`. 

A suggested sketch of this proof starts from the two conditions above.  Consider each class or trait, `C`, where `C <: Immutable`. Show that:  

1. `Bi <: Immutable` for each class or trait, `Bi` where `C <: Bi`.
2. `C` defines no `var` members.
3. `Mi <: Immutable` or `Mi <: AnyVal` for each member of `C`, `val mi: Mi`.
4. `Ti <: Immutable` or `Ti < AnyVal` or `Pi < Immutable` for each free term `ti` in the body of `C` where `(pi: Pi).ti: Ti`. 
5. `Ni <: Immutable` for each constructor application `new ... Ni(...) ...` in the body of `C`.

By way of explanation, 

1. ensures all ancestor classes are separately proved immutable. 
2. prevents direct mutation.
3. ensures all value members are separately proved immutable.
4. prevents closing over a mutable value directly or by reference to its methods.  Note that the terms may be implicit or explicit. 
5. prevents closing over a mutable value indirectly through a constructor application.

## Introducing an `IOEffect`

The scala language may get an effects system, it is the last thing on the list on the [dotty]() page.  This could be based on passing values around that represent the _capability_ to cause the effect.  

For example, `println(x: String)` would be given an extra argument `println(x: String, e: IOEffect): Unit` so that the caller must provide an `IOEffect` value.  

The caller should not manufacture the `IOEffect` or pluck an `IOEffect` out of a global variable. That would defeat the purpose.   The caller should have an `IOEffect` among its own arguments.  

A base trait for capabilities, `scala.Capability`, will enable them to be recognized in type signatures. For example, `println(x: String, e: IOEffect)` is recognized as having side-effects because `IOEffect <: Capability`.

## The Interplay of `Capability` and `Immutable`

The test for immutable types sketched out previously ignored side-effects.  We can now close this gap by preventing an immutable class or trait, `C` from closing over a capability or creating one.  Rules (3), (4) and (5) become:

3(a) (`Mi <: Immutable` and not `Mi <: Capability`) or `Mi <: AnyVal` for each member of `C`, `val mi: Mi`

4(a) (`Ti <: Immutable` and not `Ti <: Capability`) or `Ti < AnyVal` or `Pi < Immutable` for each free term `ti` in the body of `C` where `(pi: Pi).ti: Ti`. 

5(a) `Ni <: Immutable` and not `Ni <: Capability` for each constructor application `new ... Ni(...) ...` in the body of `C`.

Purity is now evident from the types alone.  A method of of `C` is pure if its type signature does not involve any capability or mutable type.  

Just as interesting, a method of `C` that accepts a capability or mutable value cannot capture its argument or any other mutable value or capability.  It can therefore be lifted into an immutable value that represents its effect.

## Thunks and IO Monads

Consider

```scala
object PrintOps extends Immutable {
    def println(x: String, e: IOEffect): Unit = { ... }
    // ...
}

import PrintOps._

object Example1 extends Immutable {
    val message: IOEffect => Unit = println("hello world", _)
}
```
 The value `message` is a thunk that will print "hello world" when it is executed.   It captures the  `PrintOps` object and a `String` in the body of a `Function1`.  To make the example valid we have to suppose those are subtypes of `Immutable`.  In that case `message` is a provably immutable value representing an I/O operation.  

Is there a monad here? 

```scala
type IO[T] = IOEffect => T

implicit object ioInstance extends scalaz.Monad[IO] {
    def bind[A, B](fa: IO[A])(fb: A => IO[B]): IO[B] = e => fb(fa(e))(e)
    def point[A](a: A): IO[A] = e => a
}
```

This is a well known monad, the function monad. Left and right identity and associativity are easily shown.

## Implicit Functions

In practice the `IOEffect` would be passed implicitly like so: `println(x: String)(implicit e: IOEffect): Unit`.  This allows us to write an expression like `println("Hello World")` which is automatically lifted into `IO[Unit]` which is now defined as:

```scala
type IO[A] = implicit IOEffect => A
```

Implicit function types should be marked `Immutable` so they can be used this way. 

The various implicit function shortcuts apply.  A `;` works something like the monadic `>>` operator in this example:

```scala
def printTwice(x: String): IO[Unit] = { println(x); println(x) }
``` 

Note that the `IOEffect` value is required but it does not affect the behaviour of `println`.  As an optimization, it can be marked for erasure  like so: `println(x: String)(erased implicit e: IOEffect): Unit`.  When that optimization is applied throughout, the thunks are eliminated during compilation and the runtime costs of reifying effects should be all but eliminated.  

## Allowing Local Mutation

It is reasonable to create and use local mutable state within a pure method or in the construction of an immutable value.  Mutable _builders_ in the standard library construct immutable collections, for example.  The penalty is the loss of equational reasoning in the local scope.  That can be mitigated by keeping the scope small or lifting the mutations into a state monad.  

Rule (5) prevents us constructing a mutable value directly because its constructor might close over some global mutable state.  But we can bless constructors that don't using the `@uncheckedImmutable` annotation.

```scala
@uncheckedImmutable object LocalState extends Immutable {
    import scala.collection.mutable
    def Buffer[A]() = new mutable.ArrayBuffer[A]()
    def Map[A, B]() = new mutable.HashMap[A, B]() 
    def Set[A]()    = new mutable.HashSet[A]()
    def Queue[A]()  = new mutable.Queue[A]()
}
```

Now we can use these in pure methods. For example, as the intermediate data structures in an (imperative) topological sort:

```scala
object Example2 extends Immutable {
    def topologicalSort(g: Graph): Vector[Node] = {
        import LocalState._
        val ordered = Buffer[Node]()
        val degree  = Map[Node, Int]()
        val pending = Queue[Node]()

        // Khan's algorithm elided for brevity
        // ...

        ordered.toVector
    }
}
```

## A State Monad

Similar to `IO` above, we can capture a mutation in an implicit function.   

```scala
type State[S, A] = implicit S => A
```
  
An `S` is a mutable value but could be seen as the capability to read and write this particular data structure.  A `State[S, A]` is a read/write operation that requires a capability of type `S` and produces an `A`. 

To show that `State` is a monad we again write out a version of the function monad, this time with some extra operations specific to state monads.  The comments to the right show the equivalent of each expression, written out in full.

```scala
class MonadState[S] extends scalaz.Monad[[A] => State[S, A]] {
    type F[A] = State[S, A]
    def bind[A, B](fa: F[A])(fb: A => F[B]): F[B] = fb(fa) // implicit s => fb(fa(s))(s)
    def point[A](a: A): F[A] = a                           // implicit s => a
    def get: F[S] = implicitly[S]                          // implicit s => s
    def gets[T](f: S => T): F[T] = f(get)                  // implicit s => f(get(s))
    def modify(f: S => Unit): F[Unit] = f(get)             // implicit s => f(get(s))
}
implicit def instance[S]: MonadState[S] = new MonadState[S]
```

Continuing the earlier graph example, here are some `State` values that operate on a map of nodes to inbound degree.  These rely on implicit function composition, rather than the foregoing typeclass.

```scala
object Example3 extends Immutable {
    // the mutable type
    type Degree = scala.collection.mutable.Map[Node, Int]
    // makes no change and returns the current state
    def degree: State[Degree, Degree] = implicitly[Degree]
    // reduces the degree of the given Node
    def decrement(n: Node): State[Degree, Int] = { val i = degree.get(n).get-1; degree.update(n, i); i }
    // discounts the outbound edges of of a Node and returns its successors
    def discount(n: Node, g: Graph): State[Degree, Seq[Node]] = for( m <- g.outEdge(n); i = decrement(m) if i == 0) yield m
}
```

### Sidebar: Isn't Implicit Mutable State Insane?

No, not when used in `Immutable` modules and classes.  A method cannot pick up a mutable value, implicit or not, from its surrounding static scope. The implicit mutable state we are dealing with is always local.  

## The End of The World

The set of modules, or global objects, and classes that are marked `Immutable` form a _functional domain_.  

- Values, methods and constructors outside the functional domain cannot be directly referenced from within.   
- Inside the functional domain all methods are pure or their effects including _externally visible_ mutation are declared in their type signatures. 
- A method with effects can be lifted into a monadic value, reifying its effects.
- The boundary of the functional domain, or "the end of the world", is where capabilities are supplied and reified effects are executed.  

These properties all stem from a concept of immutable types and don't rely on any other change to the scala language.  Implicit functions are unnecessary for any of this, but they make it easy to lift expressions into effects and to compose effects. The impact of this scheme on the standard library is not discussed.

## Further Reading

- [LACASA: Lightweight Affinity and Object Capabilities in Scala](http://www.csc.kth.se/~phaller/doc/haller16-oopsla.pdf). Philipp Haller and Alex Loiko. 

- [Gentrification Gone too Far?
Affordable 2nd-Class Values for Fun and (Co-)Effect](https://www.cs.purdue.edu/homes/rompf/papers/osvald-oopsla16.pdf). Leo Osvald, Grégory Essertel, Xilun Wu, Lilliam I. González-Alayón, Tiark Rompf.

- [A Study of Capability-Based Effect Systems (Thesis)](https://infoscience.epfl.ch/record/219173/files/thesis-2017-1-update_1.pdf). Fengyun Liu.

