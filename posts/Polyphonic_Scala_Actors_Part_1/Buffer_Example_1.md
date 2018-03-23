# Buffer Example 1

```scala
Welcome to Scala version 2.7.7.RC1 (Java HotSpot(TM) Server VM, Java 1.6.0_16).
Type in expressions to have them evaluated.
Type :help for more information.
scala>
scala> import scala.actors.Actor._
import scala.actors.Actor._
scala> case class Put(x: String)
defined class Put
scala> case object Get
defined module Get
scala> val b = actor {
     |   loop {
     |     react { case Put(x) => react { case Get => reply(x) }}
     |   }
     | }
b: scala.actors.Actor = scala.actors.Actor$$anon$1@158c3b7
scala> actor { println( "result is: " + (b !? Get)) }
res0: scala.actors.Actor = scala.actors.Actor$$anon$1@1d50fd2
scala> b ! Put( "hello" )
scala> result is: hello
```

