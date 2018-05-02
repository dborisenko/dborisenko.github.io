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

But what if we have multiple errors? It's quite common in modern development to have a function which can go wrong in a multiple different ways. Let's say, your process can fail even to start due to `ConfigNotFoundError` or can raise an error during the processing. 

There are also few possible combinations. The most direct and straight way is just to encode your error result as `Throwable`. That is how it's done in the most cases: `Future`, `Try` or even `IO` from cats (I'm not talking about `EitherT[IO, E, T]` or `IO[Either[E, T]]` — it's a bit different way). The disadvantage of this approach is that you actually know nothing about your error in the compilation time. You have to wait in runtime and try to handle _all possible errors_ with some default scenarios if something really unexpected has happened.

Another option will be to encode your error into your types. You can always return `Either[IllegalArgumentException, R]` or `IO[Either[IllegalArgumentException, R]]` or even `EitherT[IO, IllegalArgumentException, T]`. Obviously, in this case you have concrete type of your error. You know what can go wrong — you know that you have to deal with error of concrete type IllegalArgumentException. So, you know which errors you must handle.

But what should you do if you have multiple types of this error? You still can build sealed trait hierarchies:

```scala
sealed trait Error
final case class ConfigKeyNotFoundError(key: String) extends Error
case object NumberMustBe42Error extends Error
final case class IllegalArgumentError(cause: IllegalArgumentException) extends Error
final case class NoSuchElementError(cause: NoSuchElementException) extends Error
final case class OtherError(message: String) extends Error
```

This way allows you to have quite good and readable result of your process:

```scala
def subprocessFromModule1(): Either[NoSuchElementException, Int] = Left(new NoSuchElementException)

def subprocessFromModule2(): Either[IllegalArgumentException, Int] = Left(new IllegalArgumentException)

import cats.syntax.either._
def process(): Either[Error, Int] = for {
  result1 <- subprocessFromModule1().leftMap(NoSuchElementError(_))
  result2 <- subprocessFromModule2().leftMap(IllegalArgumentError(_))
} yield result1 + result2
``` 

The disadvantage of this approach is that you have to wrap all your errors, which might happen inside of your process. If you have multiple sub-processes from different modules which return different types of errors — all of them must be wrapped.

If you don't want to wrap them then your signature can be very ugly and unmaintainable:

```scala
def process(): Either[Either[IllegalArgumentException, NoSuchElementException], Int] = for {
  result1 <- subprocessFromModule1().leftMap[Either[IllegalArgumentException, NoSuchElementException]](Right(_))
  result2 <- subprocessFromModule2().leftMap[Either[IllegalArgumentException, NoSuchElementException]](Left(_))
} yield result1 + result2
```

The type signature here is already a bit over-complicated. And if we have 3 errors it can be `Either[Either[Either[IllegalArgumentException, NumberFormatException], NoSuchElementException], Int]`, etc. You of course can play with type definitions and try to hide it. But you always have to assemble them at some point. So, your code for 3 errors will be `???.leftMap[Either[Either[IllegalArgumentException, NumberFormatException], NoSuchElementException]](v => Right(Left(_)))`. Does not look so nice, right?

But are there any other ways? Yes, there are!

# Coproduct type of errors
