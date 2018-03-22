# An Incremental JSON Generator

Or yet another esoteric way to concatenate strings.

The scala type system is Turing complete, if not sentient.  On the other hand, JSON is just a simple data syntax. This post is about cracking a nut with a jack hammer.

The idea is to generate JSON incrementally and to represent the state of the under-construction output as a type.  For example, a function that starts a new JSON object as an array element would have a type signature like this:

```scala
JBuilder[OpenArray[X]] => JBuilder[OpenObject[OpenArray[X]]]
```

### Big JSON

The aim is to generate JSON from arbitrary source data.  A scala embedded DSL will be used to map the source to the result.

However, the JSON document will be big - too big for memory.  And even if it isn't, the source data is not available all at once.  The bottom line is we have to start generating JSON without seeing all the source data and continue in incremental (or streaming) mode.

And the DSL should be type safe, meaning that if it type checks at compile time the resulting JSON will be well formed. And that means we have to explicitly represent the JSON encoding state somehow. At compile time. Including array/object nesting.

I actually had the problem of generating "Big JSON" from Big Data in the context of an interface to a Cassandra database.  A Cassandra cluster can store hundreds of TB but a single request via its thrift interface can only return 64MB or so. A series of Cassandra requests was needed to supply the source data for the JSON document.

If you are skeptical about the wisdom of encoding large bodies of data as single JSON documents, I don't disagree.   Nevertheless...

### Part 1 - The Encoding State

Trait `JState` heads a family of types that represent JSON encoding states.

```scala
sealed trait JState
case class Start() extends JState 
case class OpenArray[E <: JState](enclosing: E, delimiter: Boolean = false) extends JState
case class OpenObject[E <: JState](enclosing: E, delimiter: Boolean = false) extends JState
case class Complete() extends JState
```

These classes represent four main states for JSON generation: an initial, empty sentence, a partially complete JSON array or object and a complete sentence.

Nesting of arrays and objects is represented by the member: `enclosing`. Its type `E` is not fixed and this makes `JState` a heterogeneous stack.

*   `OpenArray[Complete]` represents a top level array under construction.
*   `OpenArray[OpenObject[Complete]]` represents an array under construction as a field in an object under construction.
*   `OpenArray[OpenArray[OpenObject[Complete]]]` represents a two-level array under construction as a field in an object under construction.

### Part 2 - The Builder

A `JState` doesn't do anything by itself.  The next part is `JBuilder` which combines a `JState` with a buffer object.  The buffer accepts generated JSON text and the `JState` indicates what JSON can be legally accepted at this point.

```scala
case class JBuilder[S <: JState]( buffer: B, state: S) { ... }
```

The buffer is independent of the state so it can be swapped out when it is full and JSON generation can continue.  The buffer type, `B`, is abstract. It is declared in a surrounding trait.

The type of a `JBuilder` depends on its `state`. So, a `JBuilder[OpenArray[Complete]]` is waiting for an array element or a closing bracket ']' to be appended to its buffer.

Operations to generate JSON and update the `JBuilder` are provided by implicit wrappers `StartOps`, `ArrayOps` and `ObjectOps`.

```scala
object JBuilder {
  def apply(b: B = empty): JBuilder[Start] = apply(b, Start())
  implicit def startOps(b: JBuilder[Start]) = new StartOps(b)
  implicit def arrayOps[E <: JState](b: JBuilder[OpenArray[E]]) = new ArrayOps(b)
  implicit def objectOps[E <: JState](b: JBuilder[OpenObject[E]]) = new ObjectOps(b)
}
```

The applicable wrapper and therefore the available operations for any `JBuilder` are selected by its type or, indirectly, the type of its state member.  For example, a `JBuilder[OpenArray[Complete]]` is convertible to `ArrayOps`.

### Part 3 - The DSL

Syntax is often a matter of taste so it is convenient to have the DSL operations deferred to wrappers that are easy to fiddle with without touching the main structures.

```scala
class ObjectOps[P <: JState]( b: JBuilder[OpenObject[P]]) {
  def | [T:Element](f: (String, T)): JBuilder[OpenObject[P]] = ...
}

class ArrayOps[P <: JState](b: JBuilder[OpenArray[P]]) {
  def | [T:Element] (t: T): JBuilder[OpenArray[P]] = ...
}
```

This particular DSL uses the '|' operator to build JSON sentences from source data.  The '|' operator for a JSON object expects a tuple that will become an object field. For a JSON array it expects a value that will become an array element.

In each case, conversion of the value type `T` to JSON is handled by a typeclass instance `Element[T]`.

Here are some examples of the DSL. Assume `a1` is a `JBuilder[OpenArray[X]]` and o1 is a `JBuilder[OpenObject[X]]`:

```scala
val a2 = a1 | 23.6 | "dave" | 42l             // append 3 elements to an array
val o2 = o1 | "name" -> "dave" | "age" -> 42  // append 2 fields to an object
```

Overloads of '|' accept the keywords `array`, `obj` and `end` (actually case objects) that delimit JSON arrays and objects. For example:

```scala
val o3 = a2 | obj  | "name" -> "dave" | "age" -> 42
```

The `obj` token appends an object to the array `a2`.  But the object is not complete so the result, `o3`, is `JBuilder[OpenObject[OpenArray[X]]]`.  Next, we obtain another field that completes the object:

```scala
val a3 = o3 | "city" -> "Woolooware" | end 
```

The `end` token completes the object and the result, `a3`, has the same type as `a1`, where we came in.  Notice that the tokens obj and end don't appear in the same expression.  There is no way to use natural grouping syntax with brackets.  This is an _incremental_ JSON generator so we don't necessarily see whole JSON constructs at any point.

The implementation of '`| end`' looks like this, where `b` is the `JBuilder` we are operating upon:

```scala
def | (d: end.type) = JBuilder(append(b.buffer, "}"), b.state.enclosing )
```

The `b.state.enclosing` expression is where the state stack is popped and the type of the of the enclosing builder is recovered.

### Part 4 - Value Conversions

The `Element` typeclass is responsible for converting values to JSON. It has instances for `String`, `Int`, `Long`, `Double` and `Boolean`.  Of course, applications can add more.  There are also instances for `Option`s, Maps and `List`s of these which can be recursively nested. For example:

```scala
val a4 = a3 | Some(23.5) | None | Map( "staff" -> "harry" :: "jane" :: Nil )
```

This appends  a number, a null and an object containing an array to a3 producing a4.  Options have a little more support in the form of a `|?` operator:

```scala
a3 |  Some("thing")            // appends a string to an array
a3 |? Some("thing")            // appends a string to an array
a3 |  None                     // appends a null to an array 
a3 |? None                     // no change
o3 |  "field" -> Some("thing") // appends a string field to an object
o3 |? "field" -> Some("thing") // appends a string field to an object
o3 |  "field" -> None          // appends a null field to an object
o3 |? "field" -> None          // no change  
```

### Builder Functions

It is natural to abstract parts of the JSON encoding as functions `JBuilder[X] => JBuilder[Y]`.  The `|>` operator makes applying these functions to builders look similar to other JSON generation operations.

```scala
type PersonArray = JBuilder[OpenArray[X]]

val encodePerson: Person => PersonArray => PersonArray = {
  case Person(name, age, gender) => _ | obj | "name" -> name | "age" -> age | "sex" -> gender | end
}

val jake: Person
val julie: Person

encodePerson(julie)((encodePerson(jake)(a3))   // append jake and julie to an array
a3 |> encodePerson(jake) |> encodePerson(jule) // same again, but more clarity
```

As an alternative, the `encodePerson` function could have been realised as a `Element` typeclass instance.  That is not  possible or desirable in all cases, however.

