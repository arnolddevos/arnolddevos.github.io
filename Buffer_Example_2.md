# Buffer Example 2

```scala
Welcome to Scala version 2.7.7.RC1 (Java HotSpot(TM) Server VM, Java 1.6.0_16).
Type in expressions to have them evaluated.
Type :help for more information.
scala> import scala.actors.Actor._
import scala.actors.Actor._
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
     |     react {
     |       case Put(x)    => react {
     |         case Get => reply(x) // * see below
     |       }
     |       case PutInt(x) => react {
     |         case Get    => reply(x.toString)
     |         case GetInt => reply(x)
     |       }
     |     }
     |   }
     | }
b: scala.actors.Actor = scala.actors.Actor$$anon$1@1f4c4a3
scala> b ! Put( "hello" )
scala> b ! PutInt(42)
scala> actor { println( "Expect integer: " + (b !? GetInt)) }
res2: scala.actors.Actor = scala.actors.Actor$$anon$1@8cd4db
scala>
scala>
scala>
```

