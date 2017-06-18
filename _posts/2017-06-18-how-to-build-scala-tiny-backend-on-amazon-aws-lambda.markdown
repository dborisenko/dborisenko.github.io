---
published: false
title: How to build Scala tiny backend on Amazon AWS Lambda
layout: post
tags: [scala, aws, lambda]
categories: [scala-aws-lambda]
---

We are going to create a simple application, which posts and gets provided to lambda set of numbers.

# Initial AWS services

## Create S3 bucket

First of all, let's go to amazon aws console and create a bucket with name `scala-aws-lambda`. We need it for storing lambda code.

# Initial sbt project setup

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

Add the `AwsLambdaPlugin` auto-plugin to your build.sbt:

```scala
enablePlugins(AwsLambdaPlugin)

retrieveManaged := true

s3Bucket := Some("scala-aws-lambda")
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

First of all, we need to specify which actualy handlers we are going to use. For that we need to update `build.sbt` file.

```scala
lambdaHandlers += "post" -> "com.dbrsn.lambda.Main::post"
```



# Articles
* [Writing AWS Lambda Functions in Scala](https://aws.amazon.com/blogs/compute/writing-aws-lambda-functions-in-scala/)
* [sbt-aws-lambda](https://github.com/gilt/sbt-aws-lambda/blob/master/README.md)
