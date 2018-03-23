---
---
# Polyphonic Scala Actors Part 1

An actor responds to one message at a time.  What if you want it to recognise some combination of messages?  The idea is to respond when a combination, or "chord", occurs irrespective of the order in which the individual messages, or "notes", arrive.

## C omega

The research language [Comega](http://research.microsoft.com/en-us/um/cambridge/projects/comega/) supports this concept as seen in this example:

```java
public class Buffer {
  public async Put(string s);
  public string Get() & Put(string s) { return s; }
}
static Buffer b = new Buffer();
```

You can read the full explanation of the example in the Comega [tutorial](http://research.microsoft.com/en-us/um/cambridge/projects/comega/doc/comega_tutorial_buffer.htm). Briefly, a `Buffer`, `b`, is defined using chords.  It passes each string it receives via its `Put` method to some caller of its `Get` method. The producers and consumers would all be running in different threads.

The `&` operator is the key piece of syntax.  It joins Get and Put to describe a two note chord.   The associated block is executed each time the chord occurs, passing the string from producer to consumer.

## Haller and Van Cutsem

Philipp Haller has [implemented](http://lamp.epfl.ch/~phaller/joins) almost the same syntax for scala actors.  Here is the buffer example from the [paper](http://infoscience.epfl.ch/record/125992) by Haller and Van Cutsem:

```scala
val Put = new Join1[Int]
val Get = new Join
class Buffer extends JoinActor {
  def act {
    receive { case Get() & Put(x) => Get reply x }
  } 
}
val b = new Buffer
```

In this formulation, `Get` and `Put` are objects called _join messages_ in the paper. The `JoinActor` class augments the standard scala `Actor` class with machinery to handle join messages.

The requirement to define messages this way is an awkward aspect of this approach.  Message definitions are never local to one actor and it may be difficult, at the point of definition, to anticipate if a particular message will be used in chords defined somewhere else.  A related issue is that the case clauses for these join patterns are somewhat specialised and can't have guard predicates.

## Nested React Clauses

More typical actor code would use case classes for these messages. Actually, this particular example is simple enough that it can be written in an idiomatic way ([Buffer Example 1](/posts/Polyphonic_Scala_Actors_Part_1/Buffer_Example_1.html)):

```scala
case class Put(x: String)
case object Get
val b = actor {
  loop {
    react { case Put(x) => react { case Get => reply(x) }}
  } 
}
```

The `&` operator is gone, replaced by nested `react` clauses.  The inner block forwards the `String` from the `Put` message to the sender of the `Get` message.   The actor's message queuing semantics make this behaviour independent of the order of message arrival.

This is simple and idiomatic.  It works naturally with other actors and their message classes. It will even work if the case patterns have guards.  But it is not general, as can be seen if we try to add another chord ([Buffer Example 2](/posts/Polyphonic_Scala_Actors_Part_1/Buffer_Example_2.html)):

```scala
case class Put(x: String)
case class PutInt(x: Int)
case object Get
case object GetInt
val b = actor {
  loop {
    react { 
      case Put(x)    => react { 
        case Get => reply(x) // * see below 
      }
      case PutInt(x) => react { 
        case Get    => reply(x.toString)
        case GetInt => reply(x)
      }
    }
  }
}
```

This example wants to handle messages for integers as well as strings, but it has a bug. Imagine the message sequence `Put; PutInt; GetInt` is received.  The `PutInt; GetInt` chord will fail to be matched because the actor will be stuck  after the initial `Put` waiting for a `Get` at the line marked (*).

## Pattern/Join/Action

Can we have chords and case classes?   Better, can we have chords with arbitrary case patterns including guards?  Here is the original example again, coded with a proposed syntax ([Buffer Example 3](/posts/Polyphonic_Scala_Actors_Part_1/Buffer_Example_3.html)):

```scala
case class Put(x: String)
case object Get
val b = actor {
  loop {
     pattern { case Put(x) => join { case Get => action { lastSender ! x  }}}
  } 
}
```

Three operators: `pattern`, `join` and `action` set out chords as nested partial functions forming a tree of cases.

*   `pattern` introduces a group of chords and listens until one of them occurs
*   `join` nests cases within each other to form chords
*   `action` specifies an action to be executed when a chord is matched

When the action is executed:

*   `lastSender` designates the sender of the message matching the last pattern in the chord (which may or may not be the last message received)
*   the actor's queue will contain any messages received that were not part of the chord

And here is the more complex buffer from above, with bug corrected ([Buffer Example 4](/posts/Polyphonic_Scala_Actors_Part_1/Buffer_Example_4.html)):

```scala
case class Put(x: String)
case class PutInt(x: Int)
case object Get
case object GetInt
val b = actor {
  loop {
    pattern { 
      case Put(x) => join { 
        case Get  => action { lastSender ! x } 
      }
      case PutInt(x) => join { 
        case Get     => action { lastSender ! x.toString }
        case GetInt  => action { lastSender ! x }
      }
    }
  }
}
```

The semantics and implementation of this syntax are discussed in [Polyphonic Scala Actors Part 2](/2009/11/14/Polyphonic_Scala_Actors_Part_2.html).

