---
layout: post
title: Type level programming
subtitle: Compile time testing
date: 2019-06-17 00:00:00 +0000
tags: [Computer Science, Scala]
---

## Type level programming

As a software engineer my day-to-day activity is about building software that interacts with an external world
made of APIs, databases, the file system, even actual people.  
The external world mutates constantly and is ruled by **side effects**.

Type-level programming is about software that runs at **compile time**.  
When you code at this level you don't deal with the external world at all.  
Your world is the compiler world (or a subset of it) and it's ruled by types and logical propositions.


#### How does that help

Let's consider the following:
```scala
class Person(age: Int)
val me = new Person(34)
val myGrandson = new Person(-60)
``` 

This code compiles just fine, even if it probably makes no sense.  
What would make a little more sense instead would be something like:

```scala
class Person(age: PositiveInt)
val me = new Person(34)
val myGrandson = new Person(-60)  // <-- Compile time exception
```

The fact that the latter does not compile is a great advantage
as this particular bug cannot occur, especially in production. 

This is a little example of what can be achieved by type-level programming,
and it's in fact a solved problem in Scala-land thanks to libraries such as
[refined](https://github.com/fthomas/refined)
or [cats](https://typelevel.org/cats/datatypes) (thinking about `NonEmptyList` and so on).

I was so fascinated by all this that I dove into this world head first,
and I must say it has been quite a journey.

As a driving feature I decided to build
[a DSL that ensures SQL queries correctness at compile time](https://github.com/epifab/tydal).

This made me discover a lot of interesting aspects of type-level programming,
and this article is an attempt of summarizing some of those.


#### How does it work

In Scala, the most powerful toolset that comes to mind is macros.
Macros have been a forever-experimental feature that allows you to do all sort of things at compile time
including modifying or generating code before it gets compiled.  

However, I am not going to cover this topic here,
mostly because [macros are going to radically change in Scala 3](https://www.scala-lang.org/blog/2018/04/30/in-a-nutshell.html).

The other feature that unlocks type-level programming in Scala is the implicit,
and the things you can achieve with them.


#### Fields

Database columns in tydal are modelled by the `trait Column[T]`.
A `VARCHAR` column, for example, can be modelled as a `Column[String]`, whereas a `BIGINT` column can be modelled as a `Column[Long]`.

That's pretty straightforward and intuitive.
What about nullable fields? Simple, if a field can be `NULL` then it should be modelled as a `Field[Option[T]]`.

This is good, as it allows you to store and retrieve values from the database in a type-safe manner.

A SQL `WHERE` clasue in tydal is modelled with the `trait Filter`.
  


What about comparison between two fields?
Well, surely, two fields are comparable if their type is the same.
Also two fields are comparable if they can be safely casted to the same type.
For example, you can compare two `INT` but you can also compare an `INT` and a `FLOAT`,
because an `INT` can be safely converted to a `FLOAT`.
In the same way, you can compare a `INT NULL` and a `FLOAT NOT NULL`
because you can cast both of them to a `FLOAT NULL`.

How was this modelled in Tydal?
