Typeclassopedia
===============

Understanding the Typeclassopedia type classes. Primary goal is to gain understanding for myself (using Scala).


Type classes
------------

Type classes describe an operation for a *not yet* specific type. 

    trait JsWrite[T] {
      def toJson(t:T):JsValue
    }

Allows you to define a `toJson` method that can operate on any type.

    def toJson[T](value:T)(implicit write:JsWrite[T]) = write.toJson(value)
   
Now I can convert an arbitraty type to json

    case class A(value:String)
    object A {
      implicit val write = 
        new JsWrite[A] { 
          def toJson(a:A) = JsString("-- " + a.value + " --") 
        }
    }

For example

    toJson(A("test")) == JsString("-- test --")
    
It also allows me to enhance any instance with a `toJson` method

    implicit class JsonOps[T](instance:T)(implicit write:JsWrite[T]) {
      def toJson = write.toJson(instance)
    }

For example

    A("test").toJson == JsString("-- test --")
    
This example allows me to do a few things:

- Describe the operation of converting to json: `T => JsValue`
- Provide default implementations for well known types (think `Int`, `String`, etc.)
- Allow a method to be applied to any type as long as there is a contract (or type class) for that type. Even for types that you do not own (or can modify) yourself.
- Enhance types with extra methods while you do not know the exact type


Identity function
-----------------

A function that always returns the same value that was used as its argument.

    def identity[T](x:T):T = x


Typeclassopedia
---------------

There are a lot of type classes in the typeclassopedia that operate on container types.

Theoretically you can however define a type alias for a non-container type so that it looks like a container type.

    type Container[A] = A

This means you can treat a simple type like a `String` as if it was a container.

    val stringContainer:Container[String] = "string"
    
It probably will not be very useful in many cases. It might however show the type classes in a different light.


Laws
----

A lot of type classes in the typeclassopedia have laws. These laws allow you to check if you have made a valid implementation of that particular type class.

My understanding of these laws and how they are written down is very limiting. I do however understand that not every implementation of a certain function (even if they compile correctly) performs the correct actions.

    trait TypeClass[T] {
      def append(a:T, b:T):T = a
    }

The above example would compile but would not satisfy the laws of the type class that represents appending one instance to another.


Functor
-------

A type class for container types. If implemented for the given container type it allows you to apply a function to the contents of the container.

    trait Functor[F[_]] {
      def map[A, B](m: F[A], f: A => B): F[B]
    }

This means that for every type `T[A]` that has a function with a signature `(A => B):T[B]` it's easy to define a `Functor`.

*Functor* seems to be the name for any context of which the context can be transformed.

If we define it for `Option` it would look like this

    object OptionFunctor extends Functor[Option] {
      def map[A, B](m:Option[A], f: A => B):Option[B] = m.map(f)
      //or
      def map[A, B](m:Option[A], f: A => B):Option[B] = 
        m match {
          case Some(value) => Some(f(value))
          case None => None
        }
    }

If we define it for our generic (and weird) `Container` type it would be this

    object ContainerFunctor extends Functor[Container] {
      def map[A, B](m:Container[A], f: A => B):Container[B] = f(m)
    }
   
   
Why define it as a type class?
------------------------------

In Scala the `map` function is defined in the `Option` class, so what is the reason you would define the `map` function outside of the `Option` class?

I can think of a few answers:
  
1. `Option` can not be modified
2. `Option` is not allowed to have functions
3. At the receiving site you do not care about the container type, as long as it can perform the `A => B` transformation
4. The function is only used in a very specific scenario and thus does not belong in the `Option` class

**1. `Option` can not be modified** 

This is only applicable for types from the language or a library that is not yours. 

**2. `Option` is not allowed to have functions** 

This is not applicable because I am working with Scala.

**3. We do not care about the container type** 

In the world that I'm used to we work with interfaces (or traits) to specify contracts. I could create an interface to tell that I only care about the `A => B` conversion

    trait Functor[A] {
      def map(f: A => B):Functor[B]
    }

And then in the function

    def doStuff(x:Functor[String]) = x.map(_ + "!")
    
This would only work if we have control over instantiation of the class that should implement this trait. In case of `Option` this would not work because `Some` is `final` and `None` is an `object`. Another option would be to implement the interface in the class directly. Again, for `Option` this is not possible because it's part of the standard library which I rather not modify for personal gain.

Some might argue that structural typing could be used, I however have no clue on how to do that with parameterized methods and the return type of the method. On top of that, it would require a very strict signature.

In conclusion, it seems the interface route is not workable for every type out there. It seems can not use an interface as a way to achieve my goals.
    

**4. The function is specific to a single scenario** 

This is actually another way to say *Interface Segregation Principle*

From wikipedia:

> The interface-segregation principle (ISP) states that no client should be forced to depend on methods it does not use.

It's quite interesting that a principle from Object-Oriented Design can be solved using a concept that emerges from the functional world.


Apply
-----

A type class for container types. If implemented for the given container type it allows you to apply the function in the given container to the value in the container.

    trait Apply[Z[_]] {
      def apply[A, B](a: Z[A], f: Z[A => B]): Z[B]
    }

This means that for every type `T[A]` that has a function with a signature `(T[A => B]):T[B]` it's easy to define an `Apply`.

*Apply* seems to be the name for any context that can apply a transformation that resides in the same type of context.

If we define it for `Option` it would look like this

    object OptionApply extends Apply[Option] {
      def apply[A, B](a: Option[A], f: Option[A => B]): Option[B] =
        f.flatMap(a map _)
      //or
      def apply[A, B](a: Option[A], f: Option[A => B]): Option[B] =
        (f, a) match {
          case (Some(function), Some(value)) => Some(function(value))
          case _ => None
        }
    }

If we define it for our generic (and weird) `Container` type it would be this

    object ContainerApply extends Apply[Container] {
      def apply[A, B](f: Container[A => B], a: Continer[A]): Container[B] = f(a)
    }


Semigroup
---------

A type class for any type. If implemented for the given type it allows you to append two instances of that type.

    trait Semigroup[T] {
      def append(a: T, b: T): T
    }

This means that for every type `T` that has a function with a signature `(T):T` it's easy to define an `Semigroup`. Note that in this particular case the laws of semigroup dictate which methods could be used.

*Semigroup* seems to be the name for any type that can be appended.

If we define it for `String` it would look like this

    object StringSemigroup extends Semigroup[String] {
      def append(a: T, b: T): T = a + b
    }

Category
--------

A type class for container types that have two type parameters. If implemented for the given container type it means that the container type represents a relation between the two type parameters. More importantly it allows you to compose the containers in a way that the relation changes.

    trait Category[~>[_, _]] {
      def id[A]: A ~> A
      def compose[A, B, C](f: B ~> C, g: A ~> B): A ~> C
    }

This means that for every type `T[A, B]` that has a function with the signature `(T[B, C]):T[A, C]` it's easy to define an `Category`. Note that it should be possible to create an instance of `T[A, A]` as well.

*Category* seems to be the name for any context of related types that can be chained. Just like functions: if you have `A => B` and `B => C` you could create a function `A => C`.

If we define it for `Function1` it would look like this

    object FunctionCategory extends Category[Function1] {
      def id[A]: A => A = a => a
      def compose[A, B, C](f: B => C, g: A => B): A => C =
        f.compose(g)
        //or
        g.andThen(f)
      //or
      def id[A]: Function1[A, A] = new Function1[A, A] {
        def apply(a: A) = a
      }
      def compose[A, B, C](f: Function1[B, C], g: Function1[A, B]): Function1[A, C] =
        new Function1[A, C] {
          def apply(a: A) = f(g(a))
      }
    }
