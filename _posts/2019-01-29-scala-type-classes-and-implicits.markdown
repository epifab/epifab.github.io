---
layout: post
title: Scala, Type classes and Implicits
subtitle: What is all this about and why should I care?
date: 2019-01-29 00:00:00 +0000
tags: [Computer Science, Scala, Type Class, Implcits]
---

What is all this about and why should I care?


## Introduction

It's becoming increasingly frequent to meet other fellow developers 
who refers to Scala and its community with adjectives and nouns such as "bonkers", "for crazies", "overcomplicate".

Let's face it, the functional side of things requires a significant mindset shift.  
Have you ever been to a Scala conference?
Some of the people out there will throw at you words such as "endofunctors", "comonads", "kleisli".  
You might feel like you need a PHD in category theory.  
You might feel lost, confused and frustrated.

There's a simple solution to this. Kotlin, for example.  
Concise, expressive and elegant, pleasant to read and write.  
It's a bit like Scala, but without its superpowers.  
Functions are still first class citizens, but you cannot possibly do things in a certain way.  
And that's fair enough, because maybe it wouldn't be the best way after all, right?

Well, the point is, you cannot know until you know.


### Knowledge -> Choice -> Freedom

A wise man once told me:
> if you know, you have a choice; if you have a choice, you're a free man.

I don't actually recall where I took it from or if I just made it up, the point is that it's true.

Once upon a time I was a PHP developer.  
When I reveal this shocking fact today, my colleagues tend to smile
and say something like "I'm sorry to hear that".  
I understand it.
PHP was designed to be easy: dynamic types, interpreted, garbage collection, synchronous, global state and dollars everywhere.
It is possible to inject code snippet directly into the HTML.  
What's wrong with that? Well, nothing.  
Besides, there's nothing you can achieve with whatever other language that you cannot achieve with PHP.  
PHP is [turing complete](https://en.wikipedia.org/wiki/Turing_completeness),
and I've actually learned today that also [MS PowerPoint is](https://www.andrew.cmu.edu/user/twildenh/PowerPointTM/Paper.pdf)!

Why bother then? Why Scala? Why getting involved at all with all that crazy functional stuff?

Because I want to know.
Because I want to have a choice.

Now, if you already went all the way down the rabbit hole 
and explored the functional abyss to finally make your way out of the tunnel 
you can probably stop reading here.


## The case

Let's take polymorphism for example.

In Java you can have different implementations of the same method with different type signatures (overloading).
Of course, you can do the same in Scala.

Here's a Json encoder object:

```scala
object JsonEncoder {
  def encode(value: Int): String = value.toString
  def encode(value: Double): String = value.toString
}
```

You can actually use this:

```
scala> JsonEncoder.encode(3)
String = 3
scala> JsonEncoder.encode(4.0)
String = 4.0
```

Cool, I have a Json encoder capable of dealing with integers and doubles.  
Pretty useless, for now, let's make it more interesting.  

Let's say that I want to be able to encode also lists of numbers. First attempt:

```scala
object JsonEncoder {
  def encode(value: Int): String = value.toString
  def encode(value: Double): String = value.toString
  def encode(value: List[Int]): String = 
    "[" + value.map(encode).mkString(",") + "]"
  def encode(value: List[Double]): String = 
    "[" + value.map(encode).mkString(",") + "]"
}
```

Beautiful right? Not really.  
There are at least 3 problems:
1. code duplication: the two methods body are identical
2. everytime we add an encoder for a new type `T`, we'll also have to add a new `encode` method for `List[T]`
3. (and most importantly) this code does not compile because of type erasure

One solution that comes to my mind is to use generics and have something like:  

```scala
def encode[T](value: List[T], tEnc: T => String): String =
  "[" + value.map(tEnc).mkString(",") + "]"
```

The intuition is that given a list of `T` and an `encode` function `T -> String`,
then I can build an `encode` function `List[T] -> String`.

This complicates things a little for our end users,
because now we require a second argument to encode list of numbers:

```scala
JsonEncoder.encode(List(1, 2, 3), JsonEncoder.encode)
```

As a user of the library, what I really want to do is:

```scala
JsonEncoder.encode(List(1, 2, 3))
```

Shame: close, but not good enough.


## Type classes

What is a type class?

Quoting Wikipedia: 
> In computer science, a type class is a type system construct that supports ad hoc polymorphism.  
> This is achieved by adding constraints to type variables in parametrically polymorphic types.

Vague? Cryptic? I can agree.  
Let's jump on an example and figure out how this concept can help us out here.

First, let's rethink our architecture.  
Instead of having a single object that knows how to encode everything,
we can have a generic interface that can be extended for concrete types:

```scala
trait JsonEncoder[T] {
  def encode(t: T): String
}

object JsonEncoder {
  val intEncoder = new JsonEncoder[Int] {
    override def encode(t: Int): String = t.toString
  }
  
  val doubleEncoder = new JsonEncoder[Double] {
    override def encode(t: Double): String = t.toString
  }
  
  def listEncoder[T](tEncoder: JsonEncoder[T]) = new JsonEncoder[List[T]] {
    override def encode(ts: List[T]): String = 
      "[" + ts.map(tEncoder.encode).mkString(",") + "]"
  }
}
```

It's a bit verbose, but we can work on it.
Since the `JsonEncoder` has only one *single abstract method*,
it qualifies as a [functional interface](https://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html),
so we can create new instances by simply implementing a lambda expression `T -> String` for the `encode` method:

```scala
trait JsonEncoder[T] {
  def encode(t: T): String
}

object JsonEncoder {
  val intEncoder: JsonEncoder[Int] = (t: Int) => t.toString

  val doubleEncoder: JsonEncoder[Double] = (t: Double) => t.toString
  
  def listEncoder[T](tEncoder: JsonEncoder[T]): JsonEncoder[List[T]] = 
    (ts: List[T]) => "[" + ts.map(tEncoder.encode).mkString(",") + "]"
}
```

To recap, rather than having a bunch of encode methods, 
we have a `JsonEncoder` interface, instances for `JsonEncoder[Int]`, `JsonEncoder[Double]`
and a factory method to build new instaces of `JsonEncoder[List[T]]`.

Here's how to use it:

```scala
import JsonEncoder._
intEncoder.encode(1)
doubleEncoder.encode(2.0)
listEncoder(intEncoder).encode(List(1, 2, 3))
```

An interesting advantage here is that we are now able to encode deeply nested lists:

```scala
import JsonEncoder._
listEncoder(listEncoder(intEncoder)).encode(List(List(1, 2), List(3, 4)))
```

Unfortunately, from our end user prospective things didn't improve much as 
I need to explicitly reference or build the JSON encoder I need.

What's next? Well, here's the idea:  
What if the compiler was able to automatically figure out for us the right encoder to use 
given a value of type `T`?  
What if there was a magic, type-safe method for finding *at compile time* a `JsonEncoder[T]` for a specific `T`?

It's important to underline "at compile time": we don't want to deal with runtime errors here.  

Well, turns out that the Scala compiler has a solution for this.


## Implicits

Here we go, the most controversial functionality of Scala: *The Implicit*.

Let's dive into it:

```scala
implicit val magicNumber: Int = 3
def sum(aNumber: Int)(implicit anotherNumber: Int) = aNumber + anotherNumber
sum(2)
```

The code above will return 5.

How is it possible? Let's have a closer look line by line:

1. `implicit val magicNumber: Int = 3`  
   declares an implicit value for the type `Int`. 
2. `def sum(aNumber: Int)(implicit number: Int) = aNumber + anotherNumber`  
   declares a function that takes two numbers (one explicit and one implicit) and sums them 
3. `sum(2)`  
   invokes the function by only passing the first parameter,
   the compiler will figure out the value for the implicit one 

What about the following code?

```scala
def sum(aString: String)(implicit anotherString: String) = aString + anotherString
sum("hello")
```

The answer is: it does not compile.  
The compiler is in fact unable to find an implicit value of type `String` for the parameter `anotherString`.

Another important feature around implicits is the ability to build
an implicit value from other implicit values:

```scala
implicit def magicString(implicit number: Int): String = number.toString + "!"
```

The `magicString` function takes an implicit parameter of type `Int` and turns it into a `String`.  
Since `magicString` is also marked as implicit,
the compiler will use it to build implicit values of type `String`:

```scala
implicit val magicNumber: Int = 3
implicit def magicString(implicit number: Int): String = number.toString + "!"

def sum(aString: String)(implicit anotherString: String) = aString + anotherString
sum("the magic string is: ")
```

As we would expect, the code above will print "the magic string is: 3!"


### Implicit def and generics

The implicit resolution in Scala is extremely powerful,
especially when combined with generics:

```scala
implicit def magicList[T](implicit magicT: T): List[T] = List(magicT)
```

Exactly like the `magicString` function discussed above,
`magicList` can now be used by the compiler to build implicit
values of type `List[T]` based on implicits of type `T`.

Let's consider the following code:

```scala
implicit val magicInt: Int = 3
implicit def magicList[T](implicit magicT: T): List[T] = List(magicT)

def printMagic[T]()(implicit t: List[List[Int]]) = println(t)

printMagic()
```

The `printMagic` function in this case requires an implicit of type `List[List[Int]]`.
We have never directly defined such an implicit value,
however the compiler is able to calculate it
by recursively applying the `magicList` function:

```scala
magicList(magicList(magicNumber))  // List(List(3))
```

Powerful, right?


### Type class in practise

Now let's go back to our `JsonEncoder`.
First, let's make our encoder instances implicit:

```scala
implicit val intEncoder: JsonEncoder[Int] = (t: Int) => t.toString
implicit val doubleEncoder: JsonEncoder[Double] = (t: Double) => t.toString
implicit def listEncoder[T](implicit tEncoder: JsonEncoder[T]): JsonEncoder[List[T]] = 
  (ts: List[T]) => "[" + ts.map(tEncoder.encode).mkString(",") + "]"
```

By doing this, basically we are teaching the compiler:
1) this is how to encode integers
2) this is how to encode doubles
3) given that you know how to encode instances of type `T`, this is how you encode lists of `T`

Finally, we can rewrite our `encode` function so that it can take an implicit encoder parameter:

```scala
def encode[T](t: T)(implicit encoder: JsonEncoder[T]): String =
  encoder.encode(t)
```

Here's the full code:

```scala
trait JsonEncoder[T] {
  def encode(t: T): String
}

object JsonEncoder {
  def encode[T](t: T)(implicit encoder: JsonEncoder[T]): String =
    encoder.encode(t)

  implicit val intEncoder: JsonEncoder[Int] = (t: Int) => t.toString

  implicit val doubleEncoder: JsonEncoder[Double] = (t: Double) => t.toString
  
  implicit def listEncoder[T](implicit tEncoder: JsonEncoder[T]): JsonEncoder[List[T]] = 
    (ts: List[T]) => "[" + ts.map(tEncoder.encode).mkString(",") + "]"
}
```

Amazingly, everything works as expected:

```
scala> JsonEncoder.encode(List(1, 2, 3))
String = [1,2,3]
scala> JsonEncoder.encode(List(List(1, 2), List(3, 4)))
String = [[1,2],[3,4]]
scala> JsonEncode.encode("hello world!")
Compilation error: could not find an implicit for JsonEncoder[String]
```

What we just implemented is an example of a type class.


## Conclusion

With this post I hope I could shed some light on an important design pattern and its implementation in Scala.  
I hope this helped a little to clear some hostility and diffidence about Scala and the functional world.

There is a lot more to be said around implicits, especially about anti-patterns and how they have been abused,
making some Scala code fragile, difficult to understand and debug.

As one of my colleagues once said about the subject: "From great power comes great responsibilities."  
You have the power now, use it responsibly.


### References 

- [Scala with Cats](https://underscore.io/books/scala-with-cats)
- [Guide to Shapeless](https://underscore.io/books/shapeless-guide)
