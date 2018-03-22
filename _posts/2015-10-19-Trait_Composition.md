---
---
# Trait Composition (in scala)

Traits are powerful magic when combined with self type annotations and abstract type members. But is this malignant or benevolent magic? I have some tips for dealing with, or avoiding, out-of-control trait compositions.

In the beginning Odersky's idea was that traits should work in the small and in the large. That is: as bases of ordinary objects and as modules making up a system. 

Subsequent experience showed that you can get into a confusing mess with traits as modules. Well, I know I have.

There have been many explanations of _the cake pattern_ over the years, which attempt to keep you organised. But I have learned then forgotten about the layers and the slices.  It is all too much boiler plate.

Here is my take on trait composition.

## Mating Rituals

Trait composition is a way of mating declarations with definitions. The alternative is explicit or implicit parameter passing.

|                      | Declaration | Definition
|:---                  |:---          |:---
|**Trait Composition** | abstract value member | concrete value member
|                      | abstract type member | concrete type member
|**Parameter Passing** | formal value | actual value
|                      | formal type  | actual type

Which of these fundamental mechanisms pleases you depends on the situation but is also a matter of taste.

## Anatomy of a Simple Composition

Consider this API which has declarations for `createNew`, `Robot` and  `battle`:

```scala
trait API {
    def createNew: Robot

    type Robot <: RobotOps // note 1

    trait RobotOps {
        def battle( other: Robot ): Boolean
    }
}
```

Here is a user of the API, which is not committed to any particular implementation:

```scala
trait User extends API {
    def run = createNew battle createNew // note 2
}
```

One possible set of definitions for `createNew`, `Robot` and  `battle`:

```scala
trait Impl extends API {
    def createNew = new Robot(scala.util.Random.nextDouble)

    class Robot(val strength: Double) extends RobotOps {
        def battle( other: Robot ) = strength > other.strength // note 3
    }
}
```

Now we can compose all these traits. 

```scala
object Main extends App with User with Impl {
    println(run)
}
```

Declarations are mated to definitions and a user of the API is mated with a particular implementation.  I think this is easy enough.

## Abstract Types

Note the abstract type member, `Robot` in the `API` trait.  Trait composition and abstract type members go hand in hand.

At note 1 you can see that the type of `Robot` is left open except that it must support `RobotOps`. In java, by comparison, that cannot be expressed.

At note 2 `battle` is called with a parameter of this abstract type. 

But at note 3 the same type is concrete and the parameter of `battle` is known have a `strength` member. A typical java implementation would involve a down cast from an interface to a concrete class at this point.

## Module Hell

Confusion arises when there are more modules and more complicated interdependencies.  It is difficult to pin down the cause of module hell but you know it when you experience it.

Some symptoms I have noticed:

- The distinctions between API, user and implementation roles seem to blur.
- You repeatedly list the same parents for many traits.
- You have long lists of parent traits and you forget which are really needed and what each provides.
- Name clashes among members of the trait composition become more frequent. 

## The Kitchen Sink

Here is a fairly brutal way to break out of module hell:

1. Identify the **pluggable** modules of the composition.  Each has one API trait and (potentially) several alternative implementation traits. 
2. Identify the **fixed** modules of the composition.  Each has a trait defining both an API and its implementation. (It may depend on other traits so is not self contained.)
3. Define a _kitchen sink_ trait that extends all API traits, pluggable and fixed, but not the implementation traits.  
4. Include the kitchen sink in the self type of every other trait.
5. Define implementation objects which extend the kitchen sink with particular pluggable implementation traits.

Here is what this looks like where `Assembly` represents the kitchen sink:

```scala
trait Mod1 { self: Assembly => ... } // pluggable API
trait Impl1 { self: Assembly => ... }
trait TestImpl1 { self: Assembly => ... }

trait Mod2 { self: Assembly => ... } // pluggable API
trait Impl2 { self: Assembly => ... }
trait TestImpl2 { self: Assembly => ... }

trait Mod3 { self: Assembly => ... } // fixed
trait Mod4 { self: Assembly => ... } // fixed

trait Assembly extends Mod1 with Mod2 with Mod3 with Mod4

object Main extends Assembly with Impl1 with Impl2
object Test extends Assembly with TestImpl1 with TestImpl2

```

## More Precision

This kitchen sink approach makes every module depend on every other.  We can't see the exact dependencies between modules. 

If we want to be more precise we must narrow the self types.  That would also prevent unintentional dependencies from developing. 

Suppose `Mod1` is a pure API with no dependencies and its implementation depends only on `Mod2`:

```scala
trait Mod1 { ... }
trait Impl1 { self: Mod1 with Mod2 => ... }
trait TestImpl1 { self: Mod1 => ... }
```

At this point it is tempting to use `extends` instead of self types. That's a matter of taste for the implementation traits `Impl1` and `TestImpl1`. 

Consider these dependencies:

```scala
trait Mod2 { ... }
trait Mod3 { self: Mod2 => ... }
trait Mod4 { self: Mod3 => ... }
```

We don't want to use `extends` here because if `trait Mod3 extends Mod2` it also exposes the `Mod2` API.  That obscures the situation for users of `Mod3`.

Note that `trait Mod4 extends Mod3` would be an error, all else being equal, because it would lack `Mod2` in its self type.  Again, it is better not use `extends` here.

## Reuse a Module

If every module depends on the kitchen sink we can't pull one out and use it elsewhere. 

But if a module can stand alone we can define it this way:

```scala
trait Mod2 { ... }
trait Impl2 extends Mod2 { ... }
trait TestImpl2 extends Mod2 { ... }
```

The definition of `Assembly`, `Main` and `Test` remain the same but the module can be pulled out and reused.

## Hesitant Recommendations

I hesitate to give firm advice in this area. 

But if you are in module hell you might consider applying the kitchen sink approach and then improve the precision of the self types where appropriate.

This might be a good way to start a modular design too.  Initially everything depends on the kitchen sink which makes it easy to move things around during early development.  The self types can be progressively narrowed as the module boundaries firm up.

Oh, and by the way, don't use `override`.

