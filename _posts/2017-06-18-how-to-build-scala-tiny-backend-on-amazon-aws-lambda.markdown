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
```

# Articles
* [Writing AWS Lambda Functions in Scala](https://aws.amazon.com/blogs/compute/writing-aws-lambda-functions-in-scala/)
* [sbt-aws-lambda](https://github.com/gilt/sbt-aws-lambda/blob/master/README.md)
