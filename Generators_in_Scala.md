# Generators in Scala

I was playing around with the continuation support in scala last year.  I wanted to create something as useful and easy to use as a python generator.

## The Python

Here is a scrap of python:

```scala
def example(): 
    yield "first" 
    for i in range(1,4): 
        yield str(i) 
    yield "last" 
```

In python this function returns an iterator over the values passed to the yield statements. The nice thing is: it is lazy and can be used as a coroutine.  Anyway, if we iterate like this:

```scala
for r in example(): 
    print r 
```

It prints this:

```scala
first 
1 
2 
3 
last 
```

## The Scala

I was not able to achieve this in scala back then but I can get much closer now.  I was inspired to revisit this subject by a good [blog](blog.html) entry on the same subject by Jim McBeath: [http://jim-mcbeath.blogspot.com/2010/09/standalone-generic-scala-generator.html](http://jim-mcbeath.blogspot.com/2010/09/standalone-generic-scala-generator.html)

The part that stymied me at first was the for loop appearing in the generator.  Lets cut to the solution.  Here is my scala equivalent of the above python code:

```scala
def example = generator[Any] {produce =>
    produce( "first" )
    for( i <- suspendable(1 until 4)) 
        produce(i) 
    produce("last")
}
for( a <- example ) 
    println(a)
```

The `generator` method is the key.  It returns an `Iterator` to the caller and passes a `produce` function (similar to the python yield) to the generator body.  These are the two faces of a generator.  On the 'outside' it provides a pull interface and on the 'inside' it provides a push interface.

## Suspendable Code Paths

The other feature to notice is the `suspendable` method. To understand this, consider the familiar expansion of the for-loop:

```scala
suspendable(1 until 4).foreach(i => produce(i))
```

The compiler must be directed to use Continuation Passing Style (CPS) for any code path that may be suspended.  This is done by annotating return values with `@suspendable`. The `produce` method has a result type of `Unit @suspendable`.  Since `produce` is in turn called from the `foreach` method, its return type must be `@suspendable` too.

The problem is that the `foreach` method on the `Range`, `1 until 4,` is not so annotated.  Nor is the `foreach` method of any standard collection.  That makes it awkward to use collections in generators.  The `suspendable` method  solves this by providing a wrapper with the correct signature.

The contrast with python generators is interesting.  In python, you may only yield from the generator itself, period. No yield is possible in a nested call, annotated or not.  But, since python data structures and for comprehensions don't rely on higher order functions, this restriction is not so bad.

## How Did We Do?

All in all, our scala example comes pretty close to the simplicity of the python generator.  And, better than python, we have the possibility of decomposing our generators into a number of methods if we are prepared to use the `@suspendable` annotation.

But there are some caveats. First: it is easy to write code in the generator that can't be transformed into CPS.  An if without an else will often do it.   In these cases you get a type error.  It is generally not hard to fix these, but you are left with no illusion that you are coding in a normal context.

The second caveat is that CPS code can wind up the stack.  Tail-recursive loops that are optimised in their normal form become stack eaters in CPS form.  Even while-loops are likely to be transformed to un-optimized recursions.

Finally, my `suspendable` wrapper lets you use `Iterable`s in a generator but not the more general class of `Traversable`s. I can't see an obvious way to support the latter.

## Show Me The Code

The code to support the above example is here: [http://gist.github.com/574873](http://gist.github.com/574873) The implementation relies on a class that is simultaneously an Iterator of values and function for producing values:

```scala
class Generator[A] extends Iterator[A] with (A => Unit @ suspendable) 
```

Client code does not necessarily see this class. There are just two methods to import: `generator` and `suspendable`.

