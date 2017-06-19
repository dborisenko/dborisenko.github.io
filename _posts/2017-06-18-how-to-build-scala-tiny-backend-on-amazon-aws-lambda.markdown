---
published: false
title: How to build Scala tiny backend on Amazon AWS Lambda
layout: post
tags: [scala, aws, lambda]
categories: [scala-aws-lambda]
---

We are going to create a simple application, which posts and gets provided to lambda set of numbers.

# Initialize AWS services

## Create S3 bucket

First of all, let's go to amazon aws console and create a bucket with name `scala-aws-lambda`. We need it for storing lambda code.

## Create DynamoDB table

We are going to store our data in DynamoDB, NoSQL database provided by Amazon. Let's create table with name `numbers-db` with partition key `id` and sort key `at`.

# Initialize sbt project

Let's create an empty project
```
$ sbt new scala/scala-seed.g8
Minimum Scala build.

name [My Something Project]: scala-aws-lambda

Template applied in ./scala-aws-lambda

$ cd scala-aws-lambda/
```

Add the following to your `project/plugins.sbt` file:

```scala
resolvers += "JBoss" at "https://repository.jboss.org/"

addSbtPlugin("com.gilt.sbt" % "sbt-aws-lambda" % "0.4.2")
```

Add the `AwsLambdaPlugin` auto-plugin and s3-bucket name (the actual lambda binary will be stored there) to your `build.sbt`. We also need additional library dependencies to be able to handle lambda input (`aws-lambda-java-core`).

```scala
enablePlugins(AwsLambdaPlugin)

retrieveManaged := true

s3Bucket := Some("scala-aws-lambda")

libraryDependencies += "com.amazonaws" % "aws-lambda-java-core" % "1.1.0"
```

To create new lambda you need to run the following command:
```
AWS_ACCESS_KEY_ID=<YOUR_KEY_ID> AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY> sbt createLambda
```
To update your lambda you need to run this command:
```
AWS_ACCESS_KEY_ID=<YOUR_KEY_ID> AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY> sbt updateLambda
```

# Post method

First of all, we need to specify which actualy handlers we are going to use. Also we need to use one of the json parser. Let's use `circe`. For all of that we need to update `build.sbt` file.

```scala
lambdaHandlers += "post" -> "com.dbrsn.lambda.Main::post"

libraryDependencies ++= Seq(
  "io.circe" %% "circe-generic",
  "io.circe" %% "circe-parser",
  "io.circe" %% "circe-java8"
) map (_ % "0.8.0")

libraryDependencies += "com.amazonaws" % "aws-java-sdk-dynamodb" % "1.11.150"
```

Now we can write our simple main class with post method, which will be called from Amazon Lambda.
We need imports:

```scala
// Amazon AWS DynamoDB
import com.amazonaws.regions.Regions
import com.amazonaws.services.dynamodbv2.document.{DynamoDB, Item}
import com.amazonaws.services.dynamodbv2.{AmazonDynamoDB, AmazonDynamoDBClientBuilder}
// Amazon AWS Lambda
import com.amazonaws.services.lambda.runtime.Context
// Circe encoding/decoding
import io.circe.generic.auto._
import io.circe.java8.time._
import io.circe.parser._
import io.circe.syntax._
// Other
import org.apache.commons.io.IOUtils
import scala.collection.JavaConverters._
```

Our input order will be the following:

```scala
/**
  * Input order
  */
final case class Order(clientId: ClientId, numbers: Set[Int])

object Order {
  final type ClientId = String
}
```

Our output and persisted order will be:

```scala
/**
  * Output and persisted order
  */
final case class PersistedOrder(orderId: OrderId, at: LocalDateTime, order: Order)

object PersistedOrder {
  final type OrderId = String
}
```

Finally, Main object itself

```scala
object Main {
  // Initializing DynamoDB client
  lazy val client: AmazonDynamoDB = AmazonDynamoDBClientBuilder.standard().withRegion(Regions.US_EAST_1).build()
  lazy val dynamoDb: DynamoDB = new DynamoDB(client)

  lazy val clock: Clock = Clock.systemUTC()

  val tableName: String = "numbers-db"
  val encoding = "UTF-8"

  def post(input: InputStream, output: OutputStream, context: Context): Unit = {
    // Parsing order from input stream
    val order = parse(IOUtils.toString(input, encoding)).flatMap(_.as[Order])
    // Wrapping it to an object, which we would like to persist
    val persistedOrder = order.map(PersistedOrder(UUID.randomUUID().toString, LocalDateTime.now(clock), _))

    val persisted = persistedOrder.flatMap { o =>
      // Getting DynamoDB table
      val table = dynamoDb.getTable(tableName)
      Try {
        // Creating new DynamoDB item
        val item = new Item()
          .withPrimaryKey("id", o.orderId)
          .withLong("at", Timestamp.valueOf(o.at).getTime)
          .withString("clientId", o.order.clientId)
          .withList("numbers", o.order.numbers.toList.asJava)
        // Persisting item to DynamoDB
        table.putItem(item)
        o
      }.toEither
    }

    // Throw exception if it happened or write output order in json format otherwise
    persisted.map(_.asJson).fold(throw _, json => IOUtils.write(json.noSpaces, output, encoding))
  }

}
```


# Articles
* [Writing AWS Lambda Functions in Scala](https://aws.amazon.com/blogs/compute/writing-aws-lambda-functions-in-scala/)
* [sbt-aws-lambda](https://github.com/gilt/sbt-aws-lambda/blob/master/README.md)
