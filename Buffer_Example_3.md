# Buffer Example 3

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
scala> case object Get
defined module Get
scala> val b = actor {
     |   loop {
     |      pattern { case Put(x) => join { case Get => action { lastSender ! x  }}}
     |   }
     | }
b: scala.actors.Actor = scala.actors.Actor$$anon$1@18ec029
scala> actor { println( "result is: " + (b !? Get)) }
res0: scala.actors.Actor = scala.actors.Actor$$anon$1@7a140f
scala> b ! Put( "hello" )
scala> result is: hello
```

