Generic Types
========================

This post, potentially strap-lined TL;DR, is a guided articulation of my notes
on Scala's Type System. I've tried to organise the contents sensibly, so it's
graduated with respect to complexity. In order then, the notes cover Type
Parameterisation, Type Bounds and then Type Variance.

According to [Benjamin Pierce](
http://mitpress.mit.edu/books/types-and-programming-languages) "A type
system is a syntactic method for automatically checking the absence of certain
erroneous behaviors by classifying program phrases according to the kinds of
values they compute".

So, a type system gives us a means to express rules about the kinds of
data our code can operate upon. Scala uses a 'static type system' - which
means the rules are enforced by the compiler before the code is run. That's a
good thing.


Type Inference
=================================

Here's an example of Scala's type system doing, er - something:

```scala
scala> val showMe : String = "This is  a string"
showMe: String = This is  a string

scala> val inferMe = "So is this"
inferMe: String = So is this
```

Everything in Scala has a type. We can either state it explicitly as in
``showMe`` or defer to Scala's type inferencer. Here ``inferMe`` has been
deduced from the context of the expression as being a String. If you lie, or
Scala can't work it out, you'll land yourself a compile error.

Type inference applies to functions too. This function only operates on a single
type of data (String), a property described as ``monomorphic``.

```scala
scala> def get(param:String)  = param
scala> get("Foo") res0: String = Foo
```

We have to provide the input parameter type explicitly; using this information
and the function body, the inferencer can work out the right return type. At the
moment ``get`` s input and output types are fixed as ``String``

Type Parameterisation
=================================

Let's imagine it's our job to extend the ``get`` function to work with other
types. The options are:

1. Overload the get function for each type:

```scala
scala> def get(param:String) = param
scala> def get(param:Int) = param
scala> def get(param:Boolean) = param // and the others...
```

As a generally applicable solution this doesn't cut it. Repetition is both
tedious and contrary to one of the basic principals of software design: [DRY]
(http://en.wikipedia.org/wiki/Don't_repeat_yourself)

2. Widen the type signature to ``Any``:

```scala
scala> def get(param:Any) = param
scala> get(9) res1:Any = 9 // return type is Any
scala> get("Foo") res2: Any = Foo // return type is also Any
```

In Scala this is analogous to walking up the down escalator; we just rolled
type safety out the passenger door. This is a bad thing.  Our example compiles
because 'Any' is at the root of [Scala's type hierarchy](
(http://docs.scala-lang.org/tutorials/tour/unified-types.html) , all other
types are now valid sub-classes. Don't do this.

3. Use a "Type Parameter" (a.k.a parametric polymorphism)
  Type parameters are placeholders for specific types that will be supplied
  later. Type parameters give us configurable type safety, which sounds good.

```scala
scala>  def get[T](param:T) = param
get: [T](param: T)T
```

``[T]`` is the Type Parameter - it contains a single Type Variable: ``T``. ``T``
is the variable into which a specific type will be substituted. There's nothing
special about ``T`` it could have been any other valid variable name. Behold:

```scala
scala> get("Foo") res4: String = Foo
scala> get(9) res5: Int = 9
scala> get(9.0) res6: Double = 9.0
```

Whenever the function is invoked the actual type of ``T`` becomes apparent.  At
which point you can mentally replace all the ``Ts`` in the definition with the
type of the parameter being passed. We now have a generic function that can be
safely applied to different kinds of data. Let's move on to see what kind of
restrictions we can apply to ``T``, and why they might be useful.


Type Bounds
=================================

Scala is an object oriented language; some constructs are naturally expressed
using inheritance relationships. That has implications for our ``T``, we need a
way to constrain ``T`` to concrete types within an inheritance hierarchy. Let's
see why and how, starting with 'upper type' bounds:

```scala
trait Person
trait Qualification
type Doctor = Person with Qualification

 def operate[T <: Doctor](p:T){
  println("Pass me the knife")
}
```

The bound is denoted by the symbol ``<:`` which can be read as 'T is a sub-class
of Doctor'. So our definition states that valid values of ``T`` are constrained
to concrete types descended from Person and Qualification. ``Doctor`` is
therefore the 'upper bound', the most general concrete type our function will
accept.

This allows us to limit type selection within a hierarchy. Our example for
instance needs a little refinement...

```scala
scala> trait CyclingProficiency extends Qualification

scala> operate(new Person with CyclingProficiency)
Pass me the knife // uh-oh this looks bad :(
```

The compiler can help us out, let's refine the restriction.

```scala
scala> trait MedicalDegree extends Qualification
scala> type Doctor = Person with MedicalDegree // restrict the hierarchy

scala> operate(new Person with CyclingProficiency)

<console>:13: error: inferred type arguments [Person with CyclingProficiency]
do not conform to method operate's type parameter bounds [P <: Person with
MedicalDegree]
              operate(new Person with CyclingProficiency)
              ^
<console>:13: error: type mismatch;
 found   : Person with CyclingProficiency
 required: P
              operate(new Person with CyclingProficiency)
```

Our restriction has excluded cyclists (without medical degrees) from operating,
good job! So, to recap - the upper bound is useful for narrowing type
selection. We use it to define the most general concrete type our code can
operate upon.

As upper bounds constrain to narrower types, so lower bounds constrain to wider
ones. The lower bound is denoted by the ``>:`` symbol, which we can read as
'super-class of'.  Let's look at an example ripped from [Joshua Suereth's book
- Scala in Depth](http://www.manning.com/suereth)

```scala
class Container {

   type Things >: List[Int]

   def echo(a : Things) = a
}
```

Container has been defined with an inner type ``Things``, ``Things`` has been
constrained using a lower bound to values which are equal or
super-types of ``List[Int]``. That means we can legally create instances of
Container where ``Things`` has a more general type than ``List[Int]``.

```scala
scala> val first = new Container { type Things = Traversable[Int] }
first: Container{type Things = Traversable[Int]} = $anon$1@53edd9ee

scala> first.echo(Set(1))
res0: first.Things = Set(1)
```

Here we've created a new instance of Container that works with any type that
descends from ``Traversable[Int]`` e.g. ``Set[Int]``. It's potentially counter
intuitive to see  ``echo(Set(1))`` working without complaint. Set is
not a super-type of List?!  The thing to remember is that the restriction of
``Things`` applies to the concrete type of the instance (``Traversable``) not
the original definition.

What is prohibited is trying to create a new Container to hold Sets directly.
This fails because ``Set`` is not in an inheritance relationship with ``List``.

```scala
scala> val first = new Container { type Things = Set[Int] }
       type Things has incompatible type
               val first = new Container { type Things = Set[Int]}
```

The practical application of lower bounds is often less intuitively apparent; to
grasp its usefulness we have to move onto what happens when we start
sub-classing generic types. Let detour briefly to see the problem, and return to
this again in just a second.


Co-Variance and Contra-Variance
=================================

Extra considerations apply when we combine type parameterisation with
sub-classing. Were going to see what problems arise and how they are solved
using something called Type Variance.

Variance declares how type parameters can be changed to create new but
conformant types. For the purposes of exposition, let's create our own Generic
Type.

```scala
class Box[T]() {}
```

It would be reasonable to assume that a ``Box[String]`` could be considered a
sub-type of ``Box[Any]``. Any parameter requiring a ``Box[Any]`` should be
safely satisfied by passing a ``Box[String]``, right? Not so. In Scala generic
types have non-variant sub-typing by default. The type parameter of ``T`` cannot
be changed.

```scala
scala> class Box[T] {}

scala> val box = new Box[String]
box: Box[String] = Box@621cc66c

scala>  val box2: Box[Any] = box // Try and widen the type

// Nope - it wont work

<console>:9: error: type mismatch;
found   : Box[String]
    required: Box[Any]
   Note: String <: Any, but class Box is invariant in type T.
   You may wish to define T as +T instead. (SOLS 4.5)
           val box2: Box[Any] = box
```

The variance we are looking for is called co-variance. Co-variance allows us to
use a parent type in place of ``T``, the resulting types will then be considered
conformant. To make a class co-variant we add a plus sign (+) to the type
parameter. Co-variance tells the compiler that it's safe for this class to
appear in contexts where we are casting the variable to a super-type.

Let's update our Box and prove to ourselves it works

```scala
scala> class Box[+T] {}

scala> val box = new Box[String]
box: Box[String] = Box@4ce2fbd3

// Now try and widen the assignment
scala> val box2: Box[Any] = box
box2: Box[Any] = Box@4ce2fbd3  // Success :)
```
Now, pay attention, this is the point of the detour - let's perform a thought
experiment

```scala
scala> class Box[+T] { def update( f:T) {} }

scala> val strings = new Box[String]

// Widen the type to Any, who knows what's in here now
scala> val anythings : Box[Any] = strings

scala> val jeepers = anythings.update(false) // This can NOT be allowed!
```

Co-Variance has allowed us to widen the type to ``Any``, at which point we can
potentially make unsafe assignments. The exact same situation arises with Java
Arrays, where a runtime ``ArrayStoreException`` is raised. Scala takes a different
approach which has the advantage of being enforceable at compile

```scala
scala>  class Box[+T] { def update( f:T) {}  }

<console>:7: error: covariant type T occurs in contravariant position in type T of value f
        class Box[+T] { def update( f:T) {}  }
                                 ^
```

So our thought experiment doesn't actually compile. Scala classifies method
parameters as *contra-variant* positions, and Scala wont let us put a
*co-variant* parameter in a *contra-variant* position. This rule stops
us shooting ourselves in the face and storing a Boolean in a list of Strings.

Ok. Detour complete, let's return to why lower bounds are useful. The error
message tells us that method parameters are contra-variant. We have
to make sure that ``f`` is in a super-type relationship with ``T``.

Huzzah, we've just learned how to do this! Add a lower type bound to the method.

```scala
scala>  class Box[+T] { def update[S >: T]( f:S) {}  }
defined class Box
```

Here we've introduced another type variable ``S`` for the parameter type. That's
so we can make the assertion that ``S`` is a 'super-type' of ``T``, restricting
values of ``f`` to be contra-variant to ``T``.

If you think about it, it makes sense. Functions should be co-variant in
parameter type and contra-variant in return type. Providing a narrower input
type is always safe, e.g. passing ``List`` in place of a
``Traversable``. Likewise returning a super-type of the return value is always
safe, e.g. ``Iterable`` in place of a ``String``.

That's it; we've seen how to parameratise classes and functions and how to
restrict type variables with bounds. We've learned how to make Generic classes
that make proper use of co-variance & contra-variance.

A few points have been edited out because they distracted from the narrative I
wanted to provide. I'll round those up in a subsequent post in due course.
