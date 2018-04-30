---
layout: post
title: Universal health-check without dependencies
tags: [cats-effects, health-check]
bigimg: /img/4D226A28-8A60-4005-9AF4-20BEF039DF30.jpg
---
In the modern age of micro-services it's vitally important to have good health-checks. It's never considered as a hard task. There are few approaches around. Somebody just do a simple ping-pong (just return static pre-defined response on a given endpoint), somebody enables heavy and powerful frameworks with embedded health-check abilities.

I asked myself, can we implement strong and powerful health-check which allows us to monitor all backing services (like Postgres, Kafka, Akka, etc) without bringing a lot of complex dependencies into the module and without having huge fragmentation of all sub-modules. Here I will try to keep functional approach and have universal but powerful health-check library. For simplicity reasons, I will use `circe` as a json library and `http4s` as http server. But it's just matter of taste. I would also like to keep ability to integrate my health-checks into other http-servers (for example, in `akka-http`).

## Model

So, let's start. What is our actual model? I will start with simple Status ADT with 2 possible data types: `Ok` and `Failure`.

```scala
@JsonCodec(encodeOnly = true)
sealed abstract class HealthCheckStatus(val isOk: Boolean) {
  def isFailure: Boolean = !isOk
}

object HealthCheckStatus {

  case object Ok extends HealthCheckStatus(isOk = true)

  final case class Failure(error: String) extends HealthCheckStatus(isOk = false)

}
```

I need another one model class for abstracting of the check itself. Let's call this component `HealthCheckElement`

```scala
final case class HealthCheckElement[F[_]](name: String, status: F[HealthCheckStatus], metadata: Map[String, String])
```

Pay attention, that I use type constructor `F[_]` here. I would like to keep my check as generic as possible. So, it will represent 2 possible checks:

* The instructional check, which is not yet materialized and has to be evaluated to know the actual result of the check: `HealthCheckElement[IO]` (here I use `IO` monad from `cats-effects`).
* Already materialized check with ready to use result: `HealthCheckElement[Id]` (here I use `Id` identity type from `cats`).

And the list of all possible checks I will hold in the following structure:

```scala
final case class HealthCheck[F[_]](statuses: NonEmptyVector[HealthCheckElement[F]]) {
  def withCheck(name: String, check: F[HealthCheckStatus], metadata: Map[String, String] = Map.empty): HealthCheck[F] =
    HealthCheck(statuses.append(HealthCheckElement(name, check, metadata)))
}
```

Here I also use `F[_]` with possible values `HealthCheck[IO]` for checks-instructions and `HealthCheck[Id]` for already ready checks.

We also need a companion object:

```scala
object HealthCheck {
  def ok[F[_]](name: String, metadata: Map[String, String] = Map.empty)(implicit A: Applicative[F]): HealthCheck[F] =
    HealthCheck(NonEmptyVector.one(HealthCheckElement(name, A.pure(Ok), metadata)))

  def ok[F[_]](name: String, resolver: String => Try[String], keys: String*)(implicit A: Applicative[F]): HealthCheck[F] =
    ok[F](name, keys.flatMap(k => resolver(k).toOption.map((k, _))).toMap)

  def failure[F[_]](name: String, error: String, metadata: Map[String, String] = Map.empty)(implicit A: Applicative[F]): HealthCheck[F] =
    HealthCheck(NonEmptyVector.one(HealthCheckElement(name, A.pure(Failure(error)), metadata)))
}
```

## Folding

And now let's implement actual folding from check-instructions into already valuated health-checks. For this let's add method `fold` into `HealthCheck` class:

```scala
import cats.implicits._
import cats.MonadError

  // ...
  def fold[R](success: HealthCheck[Id] => R, failure: HealthCheck[Id] => R)(implicit A: MonadError[F, Throwable]): F[R] =
    statuses.map { v =>
      v.status.recover {
        case error => Failure(error.getMessage)
      }.map(s => HealthCheckElement[Id](v.name, s, v.metadata))
    }.sequence[F, HealthCheckElement[Id]].map { elems =>
      if (elems.exists(_.status.isFailure)) failure(HealthCheck(elems)) else success(HealthCheck(elems))
    }
  // ...
```

Here we give an instruction how to handle success cases (`success: HealthCheck[Id] => R`) and failure cases (`failure: HealthCheck[Id] => R`). We are also aware that error might happen inside this _any_ of the check. That is why we have recover logic (based on `MonadError` from `cast`). All errors need to be turned to `HealthCheckStatus.Failure` type of our ADT.

We can also add some more helper methods. For example, `transform` method of `HealthCheck` to change context of our check (for example, we can make `HealthCheck[Future]` from `HealthCheck[IO]`, if natural transformation for `IO ~> Future` exists).

```scala
import cats.~>

  // ...
  def transform[G[_]](implicit NT: F ~> G): HealthCheck[G] =
    HealthCheck(statuses.map(hc => HealthCheckElement[G](hc.name, NT(hc.status), hc.metadata)))
  // ...
```

Or we can easily raise and `Id`-based check into any other context with defined `Applicative`:

```scala
import cats.arrow.FunctionK
import cats.Applicative
import cats.~>

implicit def idToApplicative[G[_]](implicit A: Applicative[G]): Id ~> G = new FunctionK[Id, G] {
  override def apply[A](fa: A): G[A] = A.pure(fa)
}


implicit class HealthCheckIdOps(val hc: HealthCheck[Id]) extends AnyVal {
  def lift[G[_]: Applicative]: HealthCheck[G] = hc.transform[G]
}
```

We of course, need to define our `circe` encoders. But we only need to define them for already evaluated types based on `Id`:

```scala
import cats.Id
import io.circe.Encoder
import io.circe.generic.semiauto.deriveEncoder

implicit lazy val encodeHealthCheckElement: Encoder[HealthCheckElement[Id]] = deriveEncoder
implicit lazy val encodeHealthCheck: Encoder[HealthCheck[Id]] = deriveEncoder
```

## Backing services checks

And now you can ask, how this library can be universal without implementing any particular checks for Kafka, Postgres or anything else. Well, our main aim was to avoid any dependencies into the common health-check module itself. But we still hold that dependencies in that application modules, which uses that backing components. To integrate them we can easily use non-abstract methods as parameters of universal health-check library. Here can be few examples:

```scala
implicit class HealthCheckIOOps(val hc: HealthCheck[IO]) extends AnyVal {
  def withKafkaProducerCheck(
    send: (HealthCheckKafkaTopic, HealthCheckKafkaKey, HealthCheckKafkaValue) => Future[Boolean]
  ): HealthCheck[IO] = hc.withCheck(
    name = "KafkaProducer",
    check = IO.fromFuture(IO(send("health-check", "health", "check"))).map(HealthCheckStatus(_, "Kafka Producer health-check failed"))
  )

  def withActorSystemCheck(
    isRunning: => Boolean, actorSystemVersion: String, akkaHttpVersion: Option[String] = None
  ): HealthCheck[IO] = hc.withCheck(
    name = "ActorSystem",
    check = IO(HealthCheckStatus(isRunning, "Actor System is terminated")),
    metadata = Map("akka.actor.ActorSystem.Version" -> actorSystemVersion) ++
      akkaHttpVersion.map("akka.http.Version.current" -> _)
  )

  def withPostgresCheck(
    selectOne: => Future[Vector[Int]]
  )(implicit ec: ExecutionContext): HealthCheck[IO] = hc.withCheck(
    name = "PostgresDatabase",
    check = IO.fromFuture(IO(selectOne.map(r => HealthCheckStatus(r == Vector(1), "Database is not available"))))
  )
}
```

## Kafka special case

Here we can see some ready-to use helper methods for Postgres, Kafka or Akka health-checks. And that is how we are going to use them in the application code:

```scala
val config: Config = ConfigFactory.load()

val healthCheck: HealthCheck[IO] = HealthCheck
  .ok[IO]("App", (key: String) => Try(config.getString(key)), "metrics.tags")  // if we need to parse some `application.conf` data to metadata.
  .withActorSystemCheck(isActorSystemRunning, akka.actor.ActorSystem.Version, Some(akka.http.Version.current))  // We need to pre-fill isActorSystemRunning: Boolean flag. We also add versions of Akka Actor System and Akka.Http.
  .withPostgresCheck(db.run(sql"SELECT 1;".as[Int]))  // Health-check for Postgres will be just simple run "SELECT 1;". We use `slick` as a database driver here.
  .withKafkaProducerCheck(healthCheckProducer.send(_, _, _).map(m => m.hasOffset && m.hasTimestamp))  // Kafka Producer health-check is just sending heart-bit message to health-check topic
  .withCheck("CustomCheck", IO(isApplicationRunning).map(HealthCheckStatus(_, "Application is not running")))  // We can also add some custom check.
```

Here I use Kafka Producer health-check. It needs to have some special configuration. We don't want to wait long time to be able to say, is our Kafka connection working fine or not. So, our Kafka Producer configuration will look like:
```scala
val configMap: Map[String, AnyRef] = Map(
  ProducerConfig.BOOTSTRAP_SERVERS_CONFIG -> "...",
  ProducerConfig.ACKS_CONFIG -> "all",
  ProducerConfig.RETRIES_CONFIG -> new java.lang.Integer(0),
  ProducerConfig.CLIENT_ID_CONFIG -> "health-check",
  ProducerConfig.MAX_BLOCK_MS_CONFIG -> new java.lang.Long(2.seconds.toMillis),
  ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG -> new java.lang.Integer(2.seconds.toMillis.toInt)
)
val healthCheckProducer: Producer[String, String] = new KafkaProducer[Key, Value](configMap.asJava, Serdes.String().serializer(), Serdes.String().serializer())
```

And some syntax improvement for terrible Kafka signature:

```scala
implicit class ProducerOps[Key, Value](val producer: Producer[Key, Value]) extends AnyVal {
  private def producerCallback(callback: Try[RecordMetadata] => Unit): Callback =
    (metadata: RecordMetadata, exception: Exception) =>
      callback(if (exception == null) Success(metadata) else Failure(exception))

  private def producerCallback(promise: Promise[RecordMetadata]): Callback = producerCallback(result => {
    promise.complete(result)
    ()
  })

  def produce(record: ProducerRecord[Key, Value]): Future[RecordMetadata] = {
    val promise = Promise[RecordMetadata]()
    try {
      producer.send(record, producerCallback(promise))
    } catch {
      case NonFatal(e) => promise.failure(e)
    }
    promise.future
  }

  def send(topic: String, key: Key, value: Value): Future[RecordMetadata] = {
    produce(new ProducerRecord[Key, Value](topic, key, value))
  }
}
```

## Http4s Health Check Server

Here is implementation of my health-check service. It's simple, returning `Ok` if everything is fine and `ServiceUnavailable` if something goes wrong.

```scala
import cats.effect.Effect
import cats.implicits._
import io.circe.syntax._
import org.http4s.HttpService
import org.http4s.circe._
import org.http4s.dsl.Http4sDsl

import scala.language.higherKinds

class HealthCheckService[F[_]: Effect](check: () => HealthCheck[F]) extends Http4sDsl[F] {

  val service: HttpService[F] = HttpService[F] {
    case GET -> Root / "healthcheck" => check().fold(v => Ok(v.asJson), v => ServiceUnavailable(v.asJson)).flatten
  }
}
```

And here is default `http4s` implementation of server application.

```scala
import cats.effect.Effect
import fs2.StreamApp
import org.http4s.HttpService
import org.http4s.server.blaze.BlazeBuilder

import scala.concurrent.ExecutionContext
import scala.language.higherKinds

abstract class HealthCheckServer[F[_]: Effect](
  port: Int = 8080,
  host: String = "0.0.0.0",
  check: () => HealthCheck[F]
)(implicit ec: ExecutionContext) extends StreamApp[F] {
  def stream(args: List[String], requestShutdown: F[Unit]): fs2.Stream[F, StreamApp.ExitCode] =
    new HealthCheckStream(port, host, check).stream

  def run(): Unit = main(Array.empty)
}

class HealthCheckStream[F[_]: Effect](port: Int, host: String, check: () => HealthCheck[F]) {

  private val healthCheckService: HttpService[F] = new HealthCheckService[F](check).service

  def stream(implicit ec: ExecutionContext): fs2.Stream[F, StreamApp.ExitCode] = BlazeBuilder[F]
    .bindHttp(port, host).mountService(healthCheckService, "/").serve
}
```

## Akka Http Health Check Server

Another option is that we can integrate healthcheck into our existed akka-http application.

```scala
val healthCheckRoute: Route = (get & path("healthcheck")) { ctx =>
  healthCheck().fold(v => complete(v), v => complete((ServiceUnavailable, v))).unsafeToFuture().flatMap(_(ctx))
}
// ...
val route: Route = handleExceptions(ApiExceptionHandler.handle)(concat(
  otherRoute,
  healthCheckRoute
))
```

## Code snippet

In this example I tried to explain my thoughts about making zero-dependencies generic health-check library which is simple composable and runable everywhere. The full example of health-check can be found in the following [github snippet](https://gist.github.com/dborisenko/85078d3b91c365da6af98ba2c1395107) 
