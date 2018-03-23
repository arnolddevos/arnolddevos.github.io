# Buffer Example 4

```scala
Welcome to Scala version 2.7.7.RC1 (Java HotSpot(TM) Server VM, Java 1.6.0_16).
Type in expressions to have them evaluated.
Type :help for more information.
scala> import scala.actors.Actor._
import scala.actors.Actor._
scala> import au.com.langdale.actors.Joins._
import au.com.langdale.actors.Joins._
scala> case class Put(x: String)
defined class Put
scala> case class PutInt(x: Int)
defined class PutInt
scala> case object Get
defined module Get
scala> case object GetInt
defined module GetInt
scala> val b = actor {
     |   loop {
     |     pattern {
     |       case Put(x) => join {
     |         case Get  => action { lastSender ! x }
     |       }
     |       case PutInt(x) => join {
     |         case Get     => action { lastSender ! x.toString }
     |         case GetInt  => action { lastSender ! x }
     |       }
     |     }
     |   }
     | }
b: scala.actors.Actor = scala.actors.Actor$$anon$1@1ff8506
scala> b ! Put( "hello" )
scala> b ! PutInt(42)
scala> actor { println( "Expect integer: " + (b !? GetInt)) }
res2: scala.actors.Actor = scala.actors.Actor$$anon$1@5a9c6e
scala> Expect integer: 42
```

