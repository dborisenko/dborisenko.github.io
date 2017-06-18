---
published: false
title: How to build Scala tiny backend on Amazon AWS Lambda
layout: post
tags: [scala, aws, lambda]
categories: [scala-aws-lambda]
---

Let's create empty project
```
$ sbt new scala/scala-seed.g8
Minimum Scala build.

name [My Something Project]: scala-aws-lambda

Template applied in ./scala-aws-lambda

$ cd scala-aws-lambda/
```

Add the following to your `project/plugins.sbt` file:

```scala
addSbtPlugin("com.gilt.sbt" % "sbt-aws-lambda" % "0.4.2")
```

Add the `AwsLambdaPlugin` auto-plugin to your build.sbt:

```scala
enablePlugins(AwsLambdaPlugin)
```
