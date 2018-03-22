# Querying a Dataset with Scala's Pattern Matching

This post is about matching patterns against collections, combining patterns in interesting ways and defining patterns in terms of other patterns.  Patterns defined in this way look and feel like a lightweight, domain specific query language.  Very little supporting code is required.

## A Dataset and a Query

Here are some collections representing a genealogy (their contents are given at the end):

```scala
type Name = String
val male: Set[Name] = ...
val female: Set[Name] = ...
val children: Map[Name, Set[Name]] = ...
val parents: Map[Name, Set[Name]] = invert(children)
```

Now, define some extractors like this:

```scala
class Extractor[A,B]( f: A => Option[B] ) { def unapply( a: A ) = f(a) }
val Parents = new Extractor(parents.get)
val Children = new Extractor(children.get)
```

And we can use these in a pattern matching expression to query our data:

```scala
"Julie" match {
  case Parents(p) => "Julies parents are: " + p
  case Children(c) => "Julies parents are unknown but has children: " + c
  case _ => "Don't know any of Julie's relatives"
}
// => "Julies parents are Set(Nalda, John)"
```

There are many other ways to write the same query. Here is one:

```scala
parents.get("Julie") map { p => "Julies parents are: " + p } getOrElse { 
  children.get("Julie") map { c => "Julies parents are unknown but has children: " + c } getOrElse { 
    "Don't know any of Julie's relatives" 
  }
}
```

For this particular example the pattern matching expression seems clearer. To be fair, it did require us to define some extractors first. As the examples grow more complicated, the pattern matching approach looks prettier.

## Traversals and Filters

The `Parents` and `Children` extractors yield collections.  If we want to compose, say, `Children` with `Children` to extract grandchildren we must be able to match across collections.  We can overload `unapply` to accept collections:

```scala
class Extractor[A,B](f: A => Option[B]) {
  def unapply( a: A) = f(a)
  def unapply[C]( ta: Traversable[A])(implicit g: Flattener[B,C]): Option[C] = g(ta.view.map(f))
}
trait Flattener[B,C] extends (Traversable[Option[B]] => Option[C])
```

Here the implicit `Flattener` determines how to combine a collection of matches into a result. The implementation is omitted for brevity, but assume that failed matches are ignored and matches yielding sets are merged into a union. Other types of matches are returned as-is in a collection.

Now we can say:

```scala
"Nalda" match { 
  case Children(Children(c)) => "Nalda's grandchildren are: " + c 
  case Children(_) => "Nalda has children but no grandchildren"
  case _ => "Nalda is childless"
}
// => "Nalda's grandchildren are: Set(Zoe, Haley, James, David, Candy, Katey, Mel)"
```

Here our improved Extractor allows us to traverse the relationships in the data.   It also allows us to filter. Assuming `Female(x)` is an extractor that matches members of the `female` Set:

```scala
"Nalda" match { 
  case Children(Children(Female(d))) => "Nalda's granddaughters are: " + d 
  case Children(Children(_)) => "Nalda has grandchildren but no granddaughters"
  case _ => "Nalda has no grandchildren"
}
// => "Nalda's granddaughters are: List(Zoe, Haley, Candy, Katey, Mel)"
```

## Creating a Language

Case statements create partial functions from extractors.   We can also create extractors from partial functions.  This gives the ability to name and reuse pattern matching expressions in other patterns.

To make this easy, define this convenience method:

```scala
def pattern[B](pf: PartialFunction[Name,B]) = new Extractor(pf.lift)
```

Now we can name some of the foregoing expressions and reduce repetition of common patterns:

```scala
val Female = pattern { case n if female contains n => n }
val GrandChildren = pattern { case Children(Children(c)) => c }
val GrandDaughters = pattern { case GrandChildren(Female(c)) => c }
```

Here are some more family relationships:

```scala
val Mother = pattern { case Parents(Female(p)) => p }
val Siblings = pattern { case self @ Parents(Children(siblings)) => siblings - self }
val Sisters = pattern { case Siblings(Female(s)) => s }
val Male = pattern { case n if male contains n => n }
val Brothers = pattern { case Siblings(Male(b)) => b }
```

Something like a language is emerging.

## Conjunctions and Disjunctions

So far, each pattern represents a linear sequence of traversals or filters.   Branching patterns can be created with the help of a conjunction "&" operator:

```scala
object & {  def unapply[A](a: A) = Some(a, a)  }
```

Now we can write:

```scala
"Julie" match {
  case Brothers(_) & Sisters(_) => "Julie has both brother(s) and sister(s)"
  case Siblings(_) => "Julie's siblings are all the same sex"
  case _ => "Julie has no siblings"
}
// => "Julie has both brother(s) and sister(s)"
```

The conjunction operator "&" requires that two subordinate patterns be matched (or more than two if conjunctions are chained).

We already have a built in disjunction operator "|" for case patterns. And, of course, a sequence of case clauses form a disjunction.

## The Code

The code and the sample data illustrated, together with a few additional pattern constructors and combinators, are here: [http://gist.github.com/586429](http://gist.github.com/586429)

