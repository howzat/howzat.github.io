========================
Generic Types
========================

This post, potentially strap-lined TL;DR, is a guided articulation of my notes
on Scala's Type System. I've tried to organise the contents sensibly, so it's
graduated with respect to complexity. In order then, the notes cover Type
Parameterisation, Type Bounds and then Type Variance.

According to `Benjamin Pierce
<http://mitpress.mit.edu/books/types-and-programming-languages>`_ : "A type
system is a syntactic method for automatically checking the absence of certain
erroneous behaviors by classifying program phrases according to the kinds of
values they compute".

So, a type system gives us a means to express rules about the kinds of
data our code can operate upon. Scala uses a 'static type system' - which
means the rules are enforced by the compiler before the code is run. That's a
good thing.

=================================
Type Inference
=================================

Here's an example of Scala's type system doing, er - something:
::
  scala> val showMe : String = "This is  a string"
  showMe: String = This is  a string
  scala> val inferMe = "So is this"
  inferMe: String = So is this

Everything in Scala has a type. We can either state it explicitly as in
``showMe`` or defer to Scala's type inferencer. Here ``inferMe`` has been
deduced from the context of the expression as being a String. If you lie, or
Scala can't work it out, you'll land yourself a compile error.

Type inference applies to functions too. This function only operates on a single
type of data (String), a property described as ``monomorphic``.
::
  scala> def get(param:String)  = param
  scala> get("Foo") res0: String = Foo

We have to provide the input parameter type explicitly; using this information
and the function body, the inferencer can work out the right return type. At the
moment ``get`` s input and output types are fixed as ``String``

=================================
Type Parameterisation
=================================

Lets imagine it's our job to extend the ``get`` function to work with
other types. The options are:

1. Overload the get function for each type:
::
  scala> def get(param:String) = param
  scala> def get(param:Int) = param
  scala> def get(param:Boolean) = param // and the others...

As a generally applicable solution this doesn't cut it. Repetition is both
tedious and contrary to one of the basic principals of software  design: `DRY
<http://en.wikipedia.org/wiki/Don't_repeat_yourself>`_

2. Widen the type signature to 'Any':
::
  scala> def get(param:Any) = param
  scala> get(9) res1:Any = 9 // return type is Any
  scala> get("Foo") res2: Any = Foo // return type is also Any

In Scala this is analogous to walking up the down escalator; we've just rolled
type safety out the passenger door. This is a bad thing.  Our example compiles
because 'Any' is at the root of `Scala's type hierarchy
<http://docs.scala-lang.org/tutorials/tour/unified-types.html>`_ , all other
types are now valid sub-classes. Don't do this.

3. Use a "Type Parameter" (a.k.a parametric polymorphism).
Type parameters are placeholders for specific types that will be supplied
later. Type parameters give us configurable type safety, which sounds good.
::
   scala>  def get[T](param:T) = param
   get: [T](param: T)T

``[T]`` is the Type Parameter - it contains a single Type Variable: ``T``. ``T``
is the variable into which a specific type will be substituted. There's nothing
special about ``T`` it could have been any other valid variable name. Behold:
::
    scala> get("Foo") res4: String = Foo
    scala> get(9) res5: Int = 9
    scala> get(9.0) res6: Double = 9.0

Whenever the function is invoked the actual type of ``T`` becomes apparent.  At
which point you can mentally replace all the ``Ts`` in the definition with the
type of the parameter being passed. We now have a generic function that can be
safely applied to different kinds of data. Lets move on to see what kind of
restrictions we can apply to ``T``, and why they might be useful.

=================================
Type Bounds
=================================

Scala is an object oriented language; some constructs are naturally expressed
using inheritance relationships. That has implications for our ``T``, we need a
way to constrain ``T`` to concrete types within an inheritance hierarchy. Lets
see why and how, starting with 'upper type' bounds:
::
   trait Person
   trait Qualification
   type Dr = Person with Qualification

    def operate[T <: Dr](p:T){
     println("Pass me the knife")
   }

The bound is denoted by the symbol ``<:`` which can be read as 'T is a sub-class
of Dr'. So our definition states that valid values of ``T`` are constrained to
concrete types descended from Person and Qualification. ``Dr`` is therefore the
'upper bound', the most general concrete type our function will accept.

This allows us to limit type selection within a hierarchy. Our example for
instance needs a little refinement...
::
   scala> trait CyclingProficiency extends Qualification
   scala> operate(new Person with CyclingProficiency)
   Pass me the knife // uh-oh this looks bad :(

The compiler can help us out, lets refine the restriction.
::
   scala> trait MedicalDoctor extends Qualification
   scala> type Dr = Person with MedicalDoctor // restrict the hierarchy

   scala> operate(new Person with CyclingProficiency)

   <console>:13: error: inferred type arguments [Person with CyclingProficiency]
   do not conform to method operate's type parameter bounds [P <: Person with
   MedicalDoctor]
                 operate(new Person with CyclingProficiency)
                 ^
   <console>:13: error: type mismatch;
    found   : Person with CyclingProficiency
    required: P
                 operate(new Person with CyclingProficiency)

So, to recap - the upper bound is useful for narrowing type selection. We use
it to choose the most general concrete types our code can operate upon.

As upper bounds are useful for selecting narrower types, lower bounds are
useful for selecting wider types. The lower bound is denoted by the ``>:``
symbol.  Lets look at an example ripped from Joshua Suereth's book `Scala in
Depth <http://Scala in Depth>`_
::
    class Container {
      type Things >: List[Int]
      def echo(a : Things) = a
    }

Container has been defined with an inner type ``Things``, ``Things`` has been
constrained using a lower bound to values which are equal or
super-types of ``List[Int]``. That means we can legally create instances of
Container where ``Things`` has a mxore general type than ``List[Int]``.
::
   scala> val first = new Container { type Things = Traversable[Int] }
   first: Container{type Things = Traversable[Int]} = $anon$1@53edd9ee
   scala> first.echo(Set(1))
   res0: first.Things = Set(1)

Here we've created a new instance of Container that works with any type that
descends from ``Traversable[Int]`` e.g. ``Set[Int]``. It's potentially counter
intuitive to see  ``echo(Set(1))`` working without complaint. Set is
not a super-type of List?!  The thing to remember is that the restriction of
``Things`` applies to the concrete type of the instance (``Traversable``) not
the original definition.

What is prohibited is trying to create a new Container to hold Sets directly.
This fails because ``Set`` is not in an inheritance relationship with ``List``.
::
   scala> val first = new Container { type Things = Set[Int] }
          type Things has incompatible type
                  val first = new Container { type Things = Set[Int]}

The practical application of lower bounds is often less intuitively apparent; to
grasp its usefulness we have to move onto what happens when we start
sub-classing generic types. Let detour briefly and come back to this again in a
second.

=================================
Co-Variance and Contra-Variance
=================================

Extra considerations apply when we combine type parameterisation with
sub-classing. Were going to see what problems arise and how they are solved
using something called Type Variance.

Variance declares how type parameters can be changed to create new but
conformant types. For the purposes of exposition, lets create our own Generic
Type.
::
   class Box[T]() {}

It would be reasonable to assume that a ``Box[String]`` could be considered a
sub-type of ``Box[Any]``. Any parameter requiring a ``Box[Any]`` should be
safely satisfied by passing a ``Box[String]``, right? Not so. In Scala generic
types have non-variant sub-typing by default. The type parameter of ``T`` cannot
be changed.
::
   scala> class Box[T] {}
   scala> val box = new Box[String]
   box: Box[String] = Box@621cc66c
   scala>  val box2: Box[Any] = box
   <console>:9: error: type mismatch;
   found   : Box[String]
       required: Box[Any]
      Note: String <: Any, but class Box is invariant in type T.
      You may wish to define T as +T instead. (SOLS 4.5)
              val box2: Box[Any] = box

The variance we are after is called co-variance. Co-variance allows us to use a
parent type in place of  ``T``, the resulting types will then be considered
conformant. To make a class co-variant we add a plus sign (+) to the type
parameter. Co-variance tells the compiler that it's safe for this class to
appear in contexts where we are casting the variable to a super-type.
::
   scala> class Box[+T] {}
   scala> val box = new Box[String]
   box: Box[String] = Box@4ce2fbd3
   scala> val box2: Box[Any] = box
   box2: Box[Any] = Box@4ce2fbd3

All well and good, but here's a (non-compiling) thought experiment:
::
   scala> class Box[+T] { def update( f:T) {} }
   scala> val first = new Box[String]
   scala> val second : Box[Any] = first
   scala> val wtf = second.update(false) // Woah, this should not be allowed!

Co-Variance has allowed us to widen the type to ``Any``, at which point we can
potentially make unsafe assignments. The same situation arises with Java Arrays,
where a runtime ArrayStoreException is raised. Scala takes a different
approach which has the advantage of being enforceable at compile time. Scala
restricts the positions a co-variant parameter can appear.

Taken from 'Scala By Example':
::
   "A co-variant type parameter of a class may only appear in co-variant positions
   inside the class. Among the co-variant positions are the types of values in the
   class, the result types of methods in the class, and type arguments to other
   co-variant types. Not co-variant are types of formal method parameters.

The example in our thought experiment doesn't compile, ``f : T`` is a method
parameter. It prevents us from storing the Boolean.
::
   scala>  class Box[+T] { def update( f:T) {}  }
   <console>:7: error: covariant type T occurs in contravariant position in type T of value f
           class Box[+T] { def update( f:T) {}  }
                                    ^
This is where contra-variance and lower bounds might start to make sense. Notice
the error message given above. Contra-variance means that to be a conformant
type the type has to be in a super-type relationship with ``T``.

As a rule our functions should be co-variant in parameter type and
contra-variant in return type. Think about it- providing a narrower input type
is always safe, e.g. a List in place of a Traversable. Likewise returning a
super-type of the return value is always safe, e.g. ``Any`` in place of
``String``.

To fix our ``Box`` therefore we have to restrict the return type to be
contra-variant on ``T``. We've just seen how to do this - type
bounds, huzzah!
::
   scala>  class Box[+T] { def update[S >: T]( f:S) {}  }
   defined class Box

That's it; we've seen how to parameratise classes and functions and how to
restrict type variables with bounds. We've learned how to make Generic classes
that make proper use of co-variance & contra-variance.

A few points have been edited out because they distracted from the narrative I
wanted to provide. I'll round those up in a subsequent post in due course.
