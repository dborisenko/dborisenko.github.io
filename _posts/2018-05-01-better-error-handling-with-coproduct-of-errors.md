---
layout: post
title: Better error handling with Coproduct of errors
tags: [error, shapeless, exception]
bigimg: /img/8125592c-112a-403b-9bd9-8692a17a66cc.jpg
---
Scala is a very reach platform. It gives you many ways to solve the same problem. Even such fundamental and basic problem as error handling. In this post I am going to describe an approach, that is becoming more common to use. It allows you to know your concrete errors, gives you flexibility to combine them and manage your effect that could potentially contain an error.

# Deliver your errors

But let's start from the beginning. Here we can list some of the ways of handling the error. Each of them has it's pros and cons and we are not going to discuss them here.

* You can use approach of Java and throw an exceptions. Java itself supports checked (exceptions that are checked at compile time — `throws` keyword as part of method signature) and unchecked (respectively, exceptions that are not checked at compiled time — this type of error might lead to random explosions in runtime) exceptions. Scala does not support this differentiation and you need to be more careful with errors. Of course, way violates referential transparency and makes function impure.

```scala
def process(): Int = throw new IllegalArgumentException()
```

* You can return any monadic-like exception wrapper like `Try`.

```scala
def process(): Try[Int] = Failure[Int](new IllegalArgumentException())
```

* Even asynchronous `Future` has built-in ability to deliver you information about your error.

```scala
def process(): Future[Int] = Future.failed[Int](new IllegalArgumentException())
```

* In some cases you don't even need to know your error. And if you just want to encode the fact that your process was finished successfully or with failure `Option[T]` is more than enough.

```scala
def process(): Option[Int] = None
```

* In other cases you need to return more complex result. For such a cases you can build your own sealed trait hierarchies of result.

```scala
sealed trait Result
final case class Ready(result: Int) extends Result
case object Pending extends Result
final case class Failed(error: Throwable) extends Result

def process(): Result = Failed(new IllegalArgumentException())
```

* Your error can be brought by your effect system (like `cats-effects`, effect system of `scalaz`, different kinds of Bifunctorial IOs, or just `EtherT[IO, E, T]`).

```scala
def processIo(): IO[Int] = IO.raiseError(new IllegalArgumentException())
def processEt(): EitherT[IO, IllegalArgumentException, Int] = EitherT(IO(Left(new IllegalArgumentException())))
``` 

* Or you can return a monad where error is one of the possible results (`Either[Throwable, T]`, `Either[String, T]`). The left side of your `Eather` can have strings, throwable errors or sealed traits hierarchies and / or classes of errors.

```scala
def process(): Either[IllegalArgumentException, Int] = Left(new IllegalArgumentException())
``` 

As you can see, there are a lot of possible approaches. All of them are valid in Scala and can be chosen based on the requirements and use cases. 

# Know your errors

But what if we have multiple errors? It's quite common in modern development to have a function which can go wrong in a multiple different ways. Let's say, your process can fail even to start due to `ConfigNotFoundError` or can raise an error during the processing. There are also few possible combinations. 

The most direct and straight way is just to encode your error result as `Throwable`. That is how it's done in the most cases: `Future`, `Try` or even `IO` from cats (I'm not talking about `EitherT[IO, E, T]` or `IO[Either[E, T]]` — it's a bit different way). The disadvantage of this approach is that you actually don't know what error happen in compilation time. You have to wait in runtime and try to handle all possible errors with some default scenarios if something really unpredicted happened.

