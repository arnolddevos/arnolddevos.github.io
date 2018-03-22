# Wrap up your POJOs

## Or

> Create a functional DSL for (almost) any imperative POJO building API.

(We will use Jetty - but it is just an example.)

## Imperative Configuration (incomplete)

```java
// Server
Server server = new Server(threadPool);

// Handler Structure
HandlerCollection handlers = new HandlerCollection();
ContextHandlerCollection contexts = new ContextHandlerCollection();
contexts.setHandlers( ... )
handlers.setHandlers(new Handler[] { contexts, new DefaultHandler() });
server.setHandler(handlers);

// JMX
MBeanContainer mbContainer = new MBeanContainer(
        ManagementFactory.getPlatformMBeanServer());
server.addBean(mbContainer);

// HTTP Connector
ServerConnector http = new ServerConnector(server);
http.setPort(8080);
http.setIdleTimeout(30000);
server.addConnector(http);
 
// Start the server
server.start();
```

## Declarative Configuration (fragment)

```xml
<Call name="addConnector">
 <Arg>
  <New id="httpConnector" class="org.eclipse.jetty.server.ServerConnector">
   <Arg name="server"><Ref refid="Server" /></Arg>
   <Arg name="factories">
    <Array type="org.eclipse.jetty.server.ConnectionFactory">
     <Item>
      <New class="org.eclipse.jetty.server.HttpConnectionFactory">
       <Arg name="config"><Ref refid="httpConfig" /></Arg>
      </New>
     </Item>
    </Array>
   </Arg>
   <Set name="host"><Property name="jetty.http.host" /></Set>
   <Set name="port"><Property name="jetty.http.port" /></Set>
   <Set name="idleTimeout"><Property name="jetty.http.idleTimeout" /></Set>
  </New>
 </Arg>
</Call>
```

## Our Aim

```scala
val s1 = server (
  connector(host("localhost") ~ port(9897)) ~
  connector(port(9898)) ~
  routes(
    path("/test1") ~ respond { 
      case GET()  => Plain ~ "Hello World\n" 
      case POST() => Plain ~ "Noted\n"
    },
    path("/test2") ~ react { 
      case GET() => complete( Plain ~ "Hello World Too\n") 
    }
  )
  ~ started
)
```

## Core Types

```scala

  trait Build[+A] { 
    def run(): A
  }

  trait Config[-A] { 
    def affect(a: A): Unit
  }

```

## Lift and Rinse

Name | Impure | Pure
---|---|---
build | () => A | Build[A]
config | A => Unit | Config[A]
assign | (A, B) => Unit | B => Config[A]
inject | (A, B) => Unit | Build[B] => Config[A]
map | A => B | Build[A] => Build[B]
contra | A => B | Config[B] => Config[A]
pass | | Config[A]

## Combinators

Name | Arg | Arg | Result
---|---|---|---
extend | Build[A] | Config[A] | Build[A]
andThen | Config[A] | Config[A] | Config[A]
compose0 | Build[B] => Config[A] | Build[B] | Config[B] => Config[A]
compose1 | Build[B] => Config[A] | Build[A] => Build[B] | Config[B] => Config[A]

## Applied to Jetty

```scala
def createConnector = 
  map[Server, ServerConnector](new ServerConnector(_))
val addConnector = 
  inject[Server, ServerConnector](_ addConnector _)
val connector = 
  compose1(addConnector, createConnector)
val port = 
  assign[ServerConnector, Int] ( _ setPort _ )
val host = 
  assign[ServerConnector, String] ( _ setHost _ )
val addConnectionFactory = 
  inject[ServerConnector, ConnectionFactory] ( 
        _ addConnectionFactory _ )

```

## DSL

Operator | X requirement | Y requirement | Result
---|---|---|---
Y(X) | implicit X => Config[A] | Build[A] | extend
Y ~ X | implicit X => Config[A] | implicit Y => Config[A] | andThen

## Example

```scala
val s2 = server (
  connector(port(9899)) ~
  respond { 
    case GET() & PathParts("test") => Plain ~ "Hello World\n"

    case GET() & PathParts("svg")  => SVG ~  
        <svg xmlns={SVG.NS} version="1.1"><circle r="100"/></svg> 

  }
  ~ started
)

s2.run()
```

## Algrebra

### `Config[A]` is a monoid

  - Neutral element is `pass[A]`
  - Append operation is `andThen`
  - It is associative (not commutative tho`)

### Also a contravariant functor

  - contramap operation is `contra[A, B]( f: A => B)`

## Algebra

### `Build[A]` is a comonad

```scala
  def map[A, B](f: A => B): Build[A] => Build[B] = 
    bla => build(f(bla.run()))

  def coreturn[A]( bla: Build[A]): A = bla.run()

  def cobind[A, B](f: Build[A] => B): Build[A] => Build[B] =
    bla => build(f(bla))

  def cokleisli[A](cfa: Config[A]): Build[A] => A =
    bla => bla.extend(cfa).run()
```
## Thanks for Listening

>  See https://github.com/arnolddevos/fender
>  @a4dev