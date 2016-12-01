---
layout: post
title: Cats Law Checking with Discipline
category: scala
tags: fp, learning
---

As I was working through Underscore's book [Advanced Scala With Cats](http://underscore.io/books/advanced-scala/), I
got a bit confused with the `Monad` typeclass method `tailRecM`. This method is a topic for a different post, but as
I tried to figure this out, I decided this was a good opportunity to dig in to how [Cats](http://typelevel.org/cats/) 
defines and checks laws using [Discipline](https://github.com/typelevel/discipline). If Cats' laws for the `Monad` 
typeclass include any laws for `tailRecM`, this would help me learn to implement that method correctly.

Getting this to work took a little more effort than I expected, but the payoff is so very nice.

The pieces are spread around a little, and there is not really clear documentation in one place, so I had to look at a
lot of example code and even a few 
[StackOverflow](https://stackoverflow.com/questions/39561525/how-to-test-monad-instance-using-discipline) 
[answers](http://stackoverflow.com/questions/40266043/how-to-test-a-function-like-monad-with-cats-discipline).

# I Know Some of Those Words

[Cats](http://typelevel.org/cats/) is a Scala library for functional programming. Cats is heavily based around the 
[typeclass pattern](http://danielwestheide.com/blog/2013/02/06/the-neophytes-guide-to-scala-part-12-type-classes.html), 
which allows a developer to add behavior to an object in a sort of external "has-a" relationship. 

In Java, we would typically make a class extend some interface, which would allow other algorithms to use those 
types generically:

```java
public final class Integer extends Number implements Comparable<Integer> { ... }

static <T extends Comparable<? super T>> void sort(List<T> list) { ... }
```

With the typeclass pattern, we implement an external interface, which we can then use in combination with our class.
The Java `Comparable` interface is a step in this direction:

```java
static <T> void sort(List<T> list, Comparator<? super T> c) { ... }
```

In Scala, the language feature of *implicit parameters* makes this even more powerful. With this, we can define our
typeclass instance as an implicit `val` or `def`, and let the compiler pass it implicitly to our other algorithms.

```scala
def sorted[B >: A](implicit ord: math.Ordering[B]): List[A] 
```

```scala
scala> List(2,3,1).sorted
res3: List[Int] = List(1, 2, 3)

scala> case class MyInt(x: Int)
defined class MyInt

scala> List(MyInt(2), MyInt(3), MyInt(1))
res4: List[MyInt] = List(MyInt(2), MyInt(3), MyInt(1))

scala> res4.sorted
<console>:11: error: No implicit Ordering defined for MyInt.
              res4.sorted
                   ^
```

Cats provides several cool things for us: first, it defines a number of typeclasses, including many concepts from 
mathematics (specifically category theory) such as monoids, functors, and monads; some structural capabilities such
as traversability (`Traversable`) or foldability (`Foldable`); and some other basic capabilities like string formatting
(`Show`) and equivalence testing (`Eq`). Cats also provides instances of many of these typeclasses for common library
types, such as a monoid instance for Int, a monad instance for Option, or a `Show` instance for a tuple. Cats provides 
syntactic support for using these so that their use is natural and invisible, for example, adding an extension method
so that you can call `map` on any class for which you have an implicit `Functor` instance, as if it implemented `map`
directly. And Cats provides laws and checking to make sure these instances are well-behaved.
 
# Laws

If we want to make use of an interface in another algorithm, we would like to know what assumptions we can make about
that interface. For example, we'd like to know whether order matters when we combine objects together. These assumptions
can be expressed as mathematical laws which a type must follow.

For example, a [Monoid](https://en.wikipedia.org/wiki/Monoid) is a structure which has two components: a zero element 
and a combination function. To be well-behaved, it must satisfy two laws:

- *Identity*: `a combine zero = a` 
- *Associativity*: `(a combine b) combine c = a combine (b combine c)`

If these hold, order doesn't matter when we combine elements, and the zero element preserves results. So we could
imagine algorithms that might, for example, do work in parallel, or return zero for nominal results, such as
counting "go" vs "no-go" answers where we don't care how many "go" answers we get but want to track how many "no-go"). 
If we know our monoid is well-behaved, we know that making these assumptions is safe and won't introduce bugs. 
So how can we tell if our monoid is well-behaved?

Cats implements tests against these laws using [Discipline](https://github.com/typelevel/discipline). Using this approach,
the laws can be specified as a form of test code. Discipline uses [ScalaCheck](http://www.scalacheck.org/) to run tests
against these laws. This uses random data and multiple iterations to check whether the law holds for a wide range of
inputs.

It turns out we can piggyback on this to make sure our own typeclass instances are lawful using Cats' own laws.

# Testing my Monad

I started with a [`Monad`](https://github.com/typelevel/cats/blob/master/core/src/main/scala/cats/Monad.scala) instance
for a clone of `Option` (I used this only as an exercise - there was no reason I really wanted my own Option). My option
has the following structure:

```scala
sealed trait MyOption[+A]
case object MyNone extends MyOption[Nothing]
final case class MySome[A](value: A) extends MyOption[A]
```

To define a `Monad` instance for this, I needed to implement the three Cats `Monad` methods: `pure`, `flatMap` and
`tailRecM`:

```scala
implicit val myOptionMonad = new Monad[MyOption] {
  def pure[A](value: A): MyOption[A] = MySome(value)

  def flatMap[A, B](opt: MyOption[A])(fn: A => MyOption[B]): MyOption[B] = opt match {
    case MyNone => MyNone
    case MySome(value) => fn(value)
  }

  @tailrec
  def tailRecM[A, B](a: A)(f: A => MyOption[Either[A, B]]): MyOption[B] = f(a) match {
    case MyNone => MyNone
    case MySome(Left(l)) => tailRecM(l)(f)
    case MySome(Right(r)) => MySome(r)
  }
}
```

What I want to do, is to test this `Monad` instance using the Cats 
[monad laws](https://github.com/typelevel/cats/blob/master/laws/src/main/scala/cats/laws/MonadLaws.scala)

First, I'm going to need to figure out how to even get Discipline into my test. This turns out to be pretty easy. 
Discipline provides a trait which can be mixed into your [ScalaTest](http://www.scalatest.org/) suite, with two
minor caveats: first, you must use a `FunSuite`, and second, you need to have the right versions of ScalaTest and 
ScalaCheck or you'll run into a 
[nasty exception](http://stackoverflow.com/questions/33361279/scalatest-java-lang-incompatibleclasschangeerror-implementing-class).
 For newest versions of Cats, you'll need to use the newest (3.0.1)
version of ScalaTest, and everything will work fine (I did not try back-revving ScalaCheck for earlier ScalaTest).
                                       
The restriction on `FunSuite` seems like it would be fairly easy to overcome, as the `Discipline` trait is pretty
lightweight - [Circe](https://circe.github.io/circe/) builds on `FlatSpec` in their base 
[`CirceSuite`](https://github.com/circe/circe/blob/master/modules/tests/shared/src/main/scala/io/circe/tests/CirceSuite.scala).
Discipline also includes some bindings for [Specs2](http://etorreborre.github.io/specs2/) if that's more your thing.

My test therefore looks something like this:

```scala
class MyOptionSpec extends FunSuite with Matchers with Discipline {
  checkAll("MyOption[Int]", MonadTests[MyOption].monad[Int, Int, Int])
}
```

What's all that `MonadTests` stuff? I thought I wanted to run the tests in `MonadLaws`?

If you look at the [Discipline trait](https://github.com/typelevel/discipline/blob/master/shared/src/main/scala/scalatest/Discipline.scala),
you see that it takes the laws as a `Laws#RuleSet`. As Cats implements it, the laws themselves are kept separate from
test classes which build those laws into the `RuleSet` needed by Discipline. These test classes are in a nearby package,
[`cats.laws.discipline`](https://github.com/typelevel/cats/tree/master/laws/src/main/scala/cats/laws/discipline) to the 
laws, in [`cats.laws`](https://github.com/typelevel/cats/tree/master/laws/src/main/scala/cats/laws).
 
What about all the type parameters? In the case of the `MonadLaws` tests, we have three separate parameters to specify
the type of the value of the instance (`M[A]`), the type of a second value to be created by `map` and `flatMap` 
(`A => B` or `A => M[B]`, respectively), and a third type to be used in function composition (`A => B => C`).

# Getting It To Work

In order to compile and run this, we need some other implicit instances. `MonadTests.monad` takes 14 implicit parameters!
Some of these we can get some help with from Cats, but there are a couple we'll need to provide ourselves. I worked
my way through these by iteratively trying to compile and letting the compiler point out the next missing thing.

## Arbitrary

In order to generate random instances of our data types, ScalaCheck uses *Generators* to create instances. These create
random data, although biased towards edge cases like 0, `maxInt`, and so on. Generators can be composed, so we can both
rely on multiple generators to build a larger structure, and rely on an implicit generator to fill a type hole we have.

Put another way, our job is to provide a generator for `OurType[T]`, not for `OurType[SpecificType]`, and we can rely
on other generators to generate the values for `T`. By making our generator parametric, we can avoid having to 
hand-roll generators for the several types which we'll need to test the laws (for example, the `MonadTest` will need
both an `Arbitrary[M[A]]` and an `Arbitrary[M[A => B]]`, and even worse if we want to use different types for `A` and `B`).

Building an `Arbitrary` instance for `MyOption` was reasonably straightforward. Instances are either `MyNone` or `MySome`
containing values from a different `Arbitrary` instance:

```scala
implicit def arbMyOption[T](implicit a: Arbitrary[T]): Arbitrary[MyOption[T]] =
  Arbitrary(
    Gen.oneOf(
      arbitrary[T].map(v => MySome(v)),
      Gen.const(MyNone)
    )
  )
```

There isn't really a non-testing reason I'd need an `Arbitrary` instance for my type, so I'll leave this in my test
scope.

## Eq

The next piece I needed was `Eq[MyOption[T]]`. The `Eq` typeclass lets us check for equality between two items. Again
we can use the trick of relying on an implicit `Eq` to be provided for checking the values of the type `T`. My
implementation also makes use of Cats' syntax support, so instead of calling the `eqv(x, y)` method directly, I can
just use `==` and the compiler will swap these in appropriately (there are some cases where this might not call 
exactly what you think, but as long as you're not defining equivalence strangely its probably not a concern).

The `Eq` for `MyOption` does a tedious but obvious structural comparison with a pattern match, delegating to the
other `Eq` instance to check the contents only when the structure already matches.

```scala
implicit def eqMyOption[T](implicit t: Eq[T]): Eq[MyOption[T]] = new Eq[MyOption[T]] {
  def eqv(x: MyOption[T], y: MyOption[T]): Boolean = x match {
    case MyNone => y match {
      case MyNone => true
      case _ => false
    }
    case MySome(a) => y match {
      case MySome(b) if a == b => true
      case _ => false
    }
  }
}
```

Now this is a typeclass instance I might want to use in other production code, so its a good candidate to go into the
implementation in compile scope, but I've left it in my test class for this exercise for illustrative and exercise
purposes.

As I put this together, I needed an `Eq` instance for the tuple type `(Int, Int, Int)`. Cats to the rescue! Cats 
has appropriate typeclass instances for this in the `cats.instances.tuple._` package.

## Run It!

I have my `MyOptionSpec`, and I have the `Arbitrary`, the `Eq`, and of course the `Monad` instance under test, all in
scope, as well as some Cats standard instances for `Int` and tuple types.  Now we can run the test as usual in our
IDE or command line:

```
> test-only MyOptionSpec
[info] MyOptionSpec:
[info] - MyOption[Int].monad.ap consistent with product + map
[info] - MyOption[Int].monad.applicative homomorphism
[info] - MyOption[Int].monad.applicative identity
[info] - MyOption[Int].monad.applicative interchange
[info] - MyOption[Int].monad.applicative map
[info] - MyOption[Int].monad.apply composition
[info] - MyOption[Int].monad.cartesian associativity
[info] - MyOption[Int].monad.covariant composition
[info] - MyOption[Int].monad.covariant identity
[info] - MyOption[Int].monad.flatMap associativity
[info] - MyOption[Int].monad.flatMap consistent apply
[info] - MyOption[Int].monad.followedBy consistent flatMap
[info] - MyOption[Int].monad.invariant composition
[info] - MyOption[Int].monad.invariant identity
[info] - MyOption[Int].monad.map flatMap coherence
[info] - MyOption[Int].monad.monad left identity
[info] - MyOption[Int].monad.monad right identity
[info] - MyOption[Int].monad.monoidal left identity
[info] - MyOption[Int].monad.monoidal right identity
[info] - MyOption[Int].monad.mproduct consistent flatMap
[info] - MyOption[Int].monad.tailRecM consistent flatMap
[info] - MyOption[Int].monad.tailRecM stack safety
[info] ScalaTest
[info] Run completed in 774 milliseconds.
[info] Total number of tests run: 22
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 22, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[info] Passed: Total 22, Failed 0, Errors 0, Passed 22
[success] Total time: 1 s, completed Dec 1, 2016 12:00:22 AM
```

Great! My monad instance is lawful, and we notice that the original questions:
- are there any laws for `tailRecM`?
- does my instance conform to those laws?

are both "yes".

# Conclusion

Law testing is a powerful technique to make sure that your typeclass instances are lawful, and therefore likely to
work as advertised in other algorithms, without introducing bugs. If I build on Cats' typeclasses I definitely want
to make sure I follow Cats' laws as well as doing other appropriate testing for my data structures and functions.
Despite a bit of learning curve, Discipline provides a powerful way to do this kind of testing.

If I wanted to reconstruct this approach in some other library without leveraging Cats, I think the biggest challenge
would be building the structure which constructs the `Laws#RuleSet` from the individual `Laws` definitions. Cats test
classes provide a model to follow here, but these seem like they would take a bit of work to replicate and get right.
Of course, this would be something that you set up much less often than you build instances to be tested, and really,
why not just leverage Cats?

If you want to see the complete example test, it can be found on github:
- [Type definition and Monad instance](https://github.com/chrisphelps/learning-cats/blob/master/src/main/scala/MyOption.scala)
- [Law test](https://github.com/chrisphelps/learning-cats/blob/master/src/test/scala/MyOptionSpec.scala) 

Definitely check out Cats, Discipline, ScalaCheck, and Circe. All are great work and very useful. The Underscore book
 that prompted my investigation of Cats is also recommended.
