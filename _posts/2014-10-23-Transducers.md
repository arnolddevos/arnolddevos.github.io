---
---
# Homage to the Transducer

Clojure transducers and reducers in scala. [With minor updates - 24/10/14]

![Homage to the Square: Apparition, Josef Albers 1959](Transducers/homage.jpg)

Empirical learning might be characterized as repeated application of a function `(S, A) => S`, assuming `S` is existing knowledge and `A` is a new observation.

In clojure terms this signature is the basis of something called a _reducer_.

A _transducer_ is something that transforms data that will ultimately be reduced.  It is formulated as a function taking a reducer and producing a new reducer that accepts different input or treats the input differently.

Here is Rich Hickey's post [introducing transducers](http://blog.cognitect.com/blog/2014/8/6/transducers-are-coming) and here is a post about [Haskel types for transducers](http://conscientiousprogrammer.com/blog/2014/08/07/understanding-cloure-transducers-through-types/). The latter has some discussion between Rich Hickey and Franklin Chen, the author, in the comments.

I thought I would have a crack at formulating this idea in scala. Admittedly, this may impress neither clojure nor scala programmers, for different reasons.

Stream and iteratee libraries are plentiful in the scala world already. But the pitch for this particular formulation, from Rich Hickey, is: 

> "...for core.async, I became more and more convinced of the superiority of reducing function transformers over channel->channel functions..."
 
I've found it works well to define asynchronous processes in terms of foldables and fold functions, abstracting away messaging and other machinery.  So I would like to see if that can be taken further.  

My scala transducer code so far is on [github](https://github.com/arnolddevos/FlowLib/blob/master/src/main/scala/flowlib/Transducers.scala).

## Reducer

A `Reducer` has an initial value, a completion step and a reduction step. In clojure these are arity 0, 1, and 2 overloads of the reducer function.  In scala:

```scala
trait Reducer[-A, +S] {
  type State
  def init: State
  def apply(s: State, a: A): Context[State]
  def isReduced(s: State): Boolean
  def complete(s: State): S
}

type Context[State] = State
```

This is a `Reducer` over `A`'s ultimately producing an `S`. The reduction step, `apply`, operates on values of type `State` starting with `init`. 

The result is produced by applying the `complete` method to the final state. The result type `S` is not necessarily the same as the internal reduction type `State`. 

The `isReduced` predicate says that a given state is final.  The reduction should stop when such a state is encountered, ignoring remaining `A` values. In clojure this is handled by the `reduced?` function.

## Creating Reducers and Running Reductions

A `Reducer` can be constructed from an initial value and a function by calling `reducer`. It can be run over a _reducible_ source of values such as a `List` by `reduce`.

```scala
val l = List(1.0, 2.0, 3.0)
def f(s: Double, a: Double) = s*0.9 + a*0.1
val r = reducer(0.0)(f)
val s = reduce(l)(r) //  0.561
```

## Context

The type constructor `Context` adds nothing to the foregoing definitions because `Context[State]` is defined simply as `State`.  

However, `Context` can be defined as any functor. For example, `scala.util.Try` to capture errors or `scala.concurrent.Future` for an asynchronous reduction. 

The result of the reduction is obtained in whatever context is used, that is, `reduce` returns a `Context[S]`, given a `Reducer[A, S]`.

If `Context` is defined as `scalaz.concurrent.Task` or `flowlib.Process` the entire reduction is captured as a suspended, potentially asynchronous, computation which can be subsequently run or composed with others.

## Reducible

A typeclass `Reducible` is used to make sources of data eligible for reduction.   

```scala
trait Reducible[R[_]] {
  def apply[A, S](ra: R[A])(f: Reducer[A, S]): Context[S]
}

implicit val listIsReducible = new Reducible[List] { ... }
implicit val optionIsReducible = new Reducible[Option] { .. }
...
```

> An implementation note: `Reducible` instances are defined differently for each concrete `Context`.  In theory, if  `Context` was required to have a `flatMap` operation, we could consolidate these definitions.  In practice, many concrete 
`Context` types don't support deeply nested `flatMap` calls.

## Transducer

A `Transducer[A, B]` converts a reducer of `A`'s to a reducer of `B`'s. It does not depend on the result type, `S`. 

```scala
trait Transducer[+A, -B] { tb =>
  def apply[S](fa: Reducer[A, S]): Reducer[B, S]
  def andThen[C](tc: Transducer[B, C]) = new Transducer[A, C] { ... }
}

def mapper[A, B](f: B => A): Transducer[A, B]
def filter[A](p: A => Boolean): Transducer[A, A]
def flatMapper[A, B, R[_]: Reducible](f: B => R[A]): Transducer[A, B]
def takeN[A](n: Int): Transducer[A, A]
...
```

Nor does a `Transducer` depend on the type of the internal `State` of the given reducer or the concrete `Context` in which reductions are produced.

`Transducer`'s are composed using the `andThen` operation.

## Transducer State

A `Transducer` may require state that evolves during the reduction.  For example, `takeN` must add state to the reduction representing the number of values seen so far.

```scala
val l = List(1.0, 2.0, 3.0)
def f(s: Double, a: Double) = s*0.9 + a*0.1
val r1 = reducer(0.0)(f) // r1.State = Double 
val r2 = takeN(2)(r1)    // r2.State = (Double, Int)
val s = reduce(l)(r2)    // 0.29: Double
```

In this example, `r2` has result type `Double` but internally `r2.State` is `(Double, Int)`. The `Double` tracks the state of `r1` and the `Int` tracks the count of values seen. Ultimately, `r2.complete` is called which strips the count from the result.

Note that reducers and transducers are immutable.  Reduction state is created and contained within the `reduce` operation.

## Conclusion

The point of this is not to compare clojure and scala but to attempt to borrow from clojure.

I think `Reducer`, `Reducible` and `Transducer` have emerged as quite reasonable types in scala.  Each is a polymorphic function, which is not surprising given we are coming from clojure. The types surely don't model every rule, but they say quite a bit. And the whole thing is pure (important for my concurrent programming use case).

I now want to see how useful this is in practice, particularly with `type Context[S] = flowlib.Process[S]`.
