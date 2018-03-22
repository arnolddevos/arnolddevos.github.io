---
---
# Erlang Style

Thoughts about Erlang vs. original scala actors vs. Akka inspired by Steve Vinoski's talk at Yow!

I caught Steve Vinoski's talk at Yow! Sydney and met him afterwards at the conference drinks.  That was something for me because I have read many of his pieces over the years.  Anyway, Steve is a friendly guy.  His talk was about Riak and Erlang and over beers we discussed the difference between Erlang's actors and the mainstream scala offering, Akka. 

This is a subject that interests me. I gave a talk at a ScalaSyd meetup about the frustrations of scala's actors and I am working on an alternative approach, [FlowLib](FlowLib.html).  Now, I don't think the pros and cons of scala actors is a burning topic for Steve but he cheerfully indulged me.

My own conclusion is that Erlang encourages, or maybe enforces, a more functional style than Akka.  And Erlang has an extra little feature called selective receive.  Except that it isn't a little feature, it is essential to the programming model. 

I can illustrate with a toy example. Here is an Erlang process that receives measurements, scales them, and emits the scaled measurement to a downstream process. The scale factor can vary and it must be available *before* the first measurement is scaled. This despite operating in an asynchronous environment with no global message ordering guarantees. 

```Erlang
%% this is Erlang

-module(scaling).
-export([start/1, init/1]).

start(Downstream) ->
  spawn(accumulator, init, [Downstream]).

init(Downstream) ->
  receive
    {scale_factor, Factor} ->
      loop(Downstream, Factor)
  end.

loop(Downstream, Factor) ->
  receive
    {measurement, Value} ->
      Downstream ! {scaled_measurement, Value * Factor},
      loop(Downstream, Factor);

    {scale_factor, NewFactor} ->
      loop(Downstream, NewFactor);

    _ ->
      loop(Downstream, Factor)
  end.
```

Hopefully you can get the drift of this even if you are not familiar with Erlang. Some things to notice: 

* The state of this process is represented by a function and its parameters.  It starts in an `init` state and moves to a `loop` state. 

* An explicit receive operation defines which messages are of interest in a given state and how to react to them. In the `init` state, we are only interested in `scale_factor` messages, all others are held back.  This is selective receive.

* Recursion keeps the process alive. The `loop` function calls itself and its `Factor` parameter represents varying state carried forward iteration by iteration.

For the purpose of this discussion let's call this Erlang style.  Here is the example again using the original scala actors, now deprecated. 

```scala
// this is scala + original actors
  import actors.Actor
  import Actor._

object ScalingOriginalStyle {

  case class ScaleFactor(factor: Double)
  case class Measurement(value: Double)
  case class ScaledMeasurement(value: Double)

  def start(downstream: Actor) = actor {
    receive {
      case ScaleFactor(factor) => loop(downstream, factor)
    }
  }

  def loop( downstream: Actor, factor: Double ): Nothing = {
    receive {
      case Measurement(value) =>
        downstream ! ScaledMeasurement(value * factor)
        loop(downstream, factor)

      case ScaleFactor(newFactor) =>
        loop(downstream, newFactor)

      case _ =>
        loop(downstream, factor)
    }
  }
}
```

A trio of case classes are declared at the top for message types. They have the same role as the message type atoms in the Erlang.  Other than that, and ignoring ceremonial and syntactic differences, this is pretty much the same as the Erlang. Certainly this is Erlang style.   

Now lets look at one way to write the example in scala using Akka actors:

```scala
// this is scala + Akka
import akka.actor._

object ScalingAkkaStyle {

  val system = ActorSystem("SimpleSystem")
  def scaling(downstream: ActorRef) = system.actorOf(Props(new Scaling(downstream)))

  case class ScaleFactor(factor: Double)
  case class Measurement(value: Double)
  case class ScaledMeasurement(value: Double)

  trait State
  case class Init( deferred: Seq[Measurement]) extends State
  case class Loop(factor: Double) extends State

  class Scaling(downstream: ActorRef) extends Actor {
    var state: State = Init(Seq())

    def receive = { 
      case ScaleFactor(newFactor) =>
        state match {

          case Init(deferred) =>
            for(Measurement(value) <- deferred)
              scaleAndSend(newFactor, value)
            state = Loop(newFactor)

          case Loop(_) =>
            state = Loop(newFactor)
        }

      case m @ Measurement(value) =>
        state match {

          case Init(deferred) =>
            state = Init( deferred :+ m)

          case Loop(factor) =>
            scaleAndSend(factor, value)
        }

      case _ =>  
    }
    
    def scaleAndSend(factor: Double, value: Double) = 
      downstream ! ScaledMeasurement(factor * value)
  }
}
```

The lines of code are starting to mount up and the logic is becoming less clear with clauses belonging to different states interleaved. I emphasise that this is just one way to write this and it might be clearer using Akka FSM. 

Either way, it is not Erlang style.

We have the case classes for messages as in the previous example.  We add to this a pair of case classes for actor state.  Now we get to the receive loop and here is the real difference.   

The receive method introduces an effect that will be run for every arriving message.  There is no opportunity for selective receive and we must buffer messages we can't handle immediately.  As control is inverted no parameters are available and state, now including deferred messages, must be carried somewhere outside of the receive block. A `var` in this example.  

In Akka, you don't call `receive`, `receive` calls you! And that is what is complicating the above code compared to Erlang. It also mitigates against a functional style forcing at least one `var` and associated imperative passages.

[Update 3/1/2014] But wait: Ronald Kuhn, Edward Steel and others point out that the `become` operator works here.  This is Roland's rewrite with small edits of mine:

```scala
// this is scala + Akka
import akka.actor._

object ScalingAkkaBecomesStyle {

  val system = ActorSystem("SimpleSystem")
  def scaling(downstream: ActorRef) = system.actorOf(Props(new Scaling(downstream)))

  case class ScaleFactor(factor: Double)
  case class Measurement(value: Double)
  case class ScaledMeasurement(value: Double)

  class Scaling(downstream: ActorRef) extends Actor with Stash {
    def receive = {
      case ScaleFactor(factor) =>
        context.become(loop(factor))
        unstashAll()

      case _: Measurement =>
        stash()

      case _ =>
    }

    def loop(factor: Double): Receive = {
      case ScaleFactor(newFactor) => 
        context.become(loop(newFactor))

      case Measurement(value) => 
        downstream ! ScaledMeasurement(factor * value)

      case _ => 
    }
  }
}
```

Is this Erlang style?  I would say its close.  The main thing is that state is represented by a method and its parameters, similar to the Erlang example, and that gives us indisputably shorter and clearer code compared to the previous Akka example.  

Referring back to the other points used to describe Erlang style, these are not so clearly satisfied.  

* We still don't call `receive` it calls us.  Inversion of control remains key to the Akka logic.  Albeit we do call `become` with roughly the same intent as Erlang `receive`: to specify the interpretation of the next message. 

* We have the `stash` mechanism instead of selective receive. Perhaps not a big deal.

* And this is imperative code, every expression here is evaluated for its side effect.  A receive block is an effect therefore new states can only  be conveyed by effects: `stash`, `unstashAll` and `become`.

In fairness, the original scala actors example is not purely functional either, it just looks like it is.  The `receive` function there throws an exception to pass control back to the scheduler. And in Erlang, send (!) is obviously an effect.

All in all, you reason about this Akka example very differently to the Erlang example.

What about the original scala actors, are they better?  Probably not.  They still suffer from implementation problems such as exceptions for passing control and an inefficient implementation of selective receive based on partial functions.  Philipp Haller, their author, wrote about solutions and alternatives under the headings _Translucent Functions_ and _Scala Joins_. 

Another question is: do we like Erlang style anyway?  Go back 5 or more years and at least some scala programmers considered  it poor style to have more than one receive/react in an actor and recursively formulated actors just plain counter intuitive. "Actors changing behaviour? Nonsense sir!" 

In the meantime the scope of Akka has grown much greater and it has swept all before it.

Thanks all for your comments on this piece.
