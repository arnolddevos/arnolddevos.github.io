---
---
# Temple of purity

### Creed

f(x) == f(x)

## Catechism

* Equational Reasoning

* Abstract Algebra
  
  The Monad and the Monoid

## Impure

Mutants

```scala
var x = 1
...
x = 2
```

Effects

```scala
readByte() != readByte()
```
## Pure in an Impure World?

![Control System](/posts/Actor_Discussion/Feedback_loop_with_descriptions.svg)

## The State Monad

```scala
/**Mutable variable in state thread S containing a value of type A. 
http://research.microsoft.com/en-us/um/people/simonpj/papers/lazy-functional-state-threads.ps.Z 
*/
sealed trait STRef[S, A] {
  protected var value: A

  /**Reads the value pointed at by this reference. */
  def read: ST[S, A] = returnST(value)

  /**Modifies the value at this reference with the given function. */
  def mod[B](f: A => A): ST[S, STRef[S, A]] = st((s: Tower[S]) => {
    value = f(value);
    (s, this)
  })

  ...

```

## Actors to Manage Mutable State

```scala
  class Diff(other: Actor) extends Actor {

    var last: Option[Int] = None

    def act = loop {
      react {
        case i: Int => 
          last match {
            case Some(`i`) =>
            case _ => other ! i; last = Some(i)
          }

        case _ => ???
      }
    }
  }
```
## What's Wrong

the bodies are not functions and actor programming is not functional

```scala
    var last: Option[Int] = None
```

## What's Wrong

messages, channels and actor bodies are not typed! enough said

```scala
        case i: Int => ...
        case _      => ???
```

## What's Wrong

actors don't compose. the wiring is baked into the actor bodies

```scala
  class Diff(other: Actor) extends Actor { ... }
```

tricky to change connection topology in flight

## What's Wrong

poor support for message joins - needed for any topology beyond trees

old school actors coud do this:

```scala

  react {
    case Right(b) => react {
      case Left(a) => // got both an A and a B
    }
  }

```

A and B can arrive in either order

## What's Wrong

poor support for flow control - you need to design your own mini communication protocols

```scala
    val print = new Printer
    val diff = new UglyDiff(print)
    val emit = new Emitter(200, diff)

    print.start
    diff.start // diff can overrun the mailbox of print
    emit.start // emit can overrun the mailbox of diff
```

## Actor Improvements

  * Akka Finite State Machines 
  * Akka Typed Channels

## Alternatives

  * Futures, async iteratees etc.  

    (but connection topology is limited)

  * Functional Reactive Programming 

    (but glitches vs. synchronous operation)

  * Clustered stream processors (Storm) and CEP systems?

## My Solution - Flow Actors library


## Look Mum, no Vars

```scala
  object diff extends Actor {
      
    val input = Input[Int](6)
    val output = Output[Int]()
       
    def start: Action = input react { 
      i => output(i) { ready(i) }
    }

    def ready(last: Int): Action = input react { 
      case `last` => ready(last)
      case i => output(i) { ready(i) }
    }
      
    run(start, 1)
  }
```

Note

* multiple input and output channels
* explicit input buffer size
* run a team of actors on same channels

## Safe Wiring Operations

```scala
emit.output --> diff.input; diff.output --> print.input
diff.error --> supervisor.input
```
Note

* supervisor channel
* positive flow control
* wire live actors safely

## Here it is

Github: 

https://github.com/arnolddevos/FlowActors

Slides: 

http://notes.backgroundsignal.com/Actor_Discussion.html

@a4dev
      