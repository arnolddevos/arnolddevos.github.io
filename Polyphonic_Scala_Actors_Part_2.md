# Polyphonic Scala Actors Part 2

The Joins module is a non-intrusive extension to the scala actors library that allows an actor to respond to a combination of messages called a chord.

The background can be read in [Polyphonic Scala Actors Part 1](Polyphonic_Scala_Actors_Part_1.html).

## A Short Manual

The module defines three operators: `pattern`, `join` and `action`.  These are used to set out _join patterns_ as nested partial functions forming a tree of cases.

The pattern operator:

```scala
pattern { case pat1 => .... case pat2 => ... } 
```

introduces a tree of join patterns and causes the actor to dequeue messages until one branch of the join is matched completely.  All events not participating in the successful join are requeued and the action clause is executed.

The join operator:

```scala
pattern { case pat1 => join { case pat2 => ... }} 
```

joins patterns pat1 and pat2. Each must be matched by a distinct message before the join is considered to match but the messages may arrive in any order.

Here is the simplest branching join pattern:

```scala
pattern { case pat1 => join { case pat2 => ... case pat3 => ...}}
```

This defines two joins.  pat1 is joined with pat2 and pat1 is joined with pat3.

Once a join pattern is matched, an action can be taken:

```scala
pattern { case pat1 => join { case pat2 => action { act1 }}}
```

This defines an action, act1, to be executed when the join of pat1 and pat2 is matched. The action can trigger further pattern matching with a nested `pattern {...}` or by(recursively) invoking the enclosing pattern.  The loop operator can be used to do this also:

```scala
loop { pattern { case pat1 => join { case pat2 => action { act1 }}}}
```

This continuously matches pat1 joined with pat2.

An action may also call react or receive directly and can respond to any messages other than those that were consumed by the successful join pattern.

Within an action, `lastSender` designates the sender of the message matching the last pattern in the chord (which may or may not be the last message received). This may be used to effect a reply to that actor.

## Examples

Here is the standard example for joins.  It implements a buffer as described in [Polyphonic Scala Actors Part 1](Polyphonic_Scala_Actors_Part_1.html):

```scala
import scala.actors.Actor._
import au.com.langdale.actors.Joins._
case class Put(x: Any)
case object Get
val buffer = actor {
  loop {
    pattern {
      case Put(x) => join { case Get => action { lastSender ! x }}
    }
  }
}
```

This example and another showing a more elaborate tree of cases are [here](http://gist.github.com/234907).

## Implementation

The Joins module code is [here](http://gist.github.com/234907).

The heart of the implementation is the `State` class:

```scala
class State( 
   matches: List[PartialMatch], 
   history: List[Event], 
   serial: Int
) extends 
   PartialFunction[Any,Unit] {
 ... 
}
```

An instance represents the matching state of the actor.  It contains the partially matched join patterns and the messages seen so far.  A `State` is also a partial function that can be passed to `react` to receive a message.

The `pattern` operator constructs an initial `State` instance and passes it to `react`.  When that instance receives a message it constructs a successor `State` instance and  passes it to react.

A successor state is constructed by matching an incoming message against all pending patterns in the current `State`.  A match results in either an action to perform or another  pattern that is added to the next `State`.  Any messages from the history, not as yet used in this join, are replayed against the new pattern.

Eventually one of the join patterns is matched completely resulting in an action.  Any unused messages are re-queued and the `action` is executed.

This is a somewhat brute force solution with (I think) worst case algorithmic complexity O(n*m) for n messages and m case clauses.  On the other hand, a simple actor with a react clause and m case clauses would have the same complexity.  In the absence of any introspection of the partial functions this seems the best that can be done.

## Discussion

The advertised benefit of implementing actors in a library using only general purpose features of the language seems to be vindicated here.  It has enabled this extension to be created as an additional module but with a syntax that, I think, reads naturally.

The resulting module is non-intrusive, meaning you don't have to adopt a variant of the actors library to use it and you don't need to redesign any message classes.  There is no time and memory cost to actors that don't need the join pattern machinery and no (necessary) impact on their design if another actor does use join patterns.

The syntax is general enough that it could support other implementations, too.

