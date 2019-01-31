---
layout: post
title: Building Sortable lists with scalajs-react
tags: [react-sortable-hoc, react, scalajs]
bigimg: /img/???.jpg
gh-repo: dborisenko/scalajs-react-components
gh-badge: [star, fork, follow]
---

# React Sortable (HOC)

React Sortable (HOC) is a set of higher-order components to turn any list into an animated, touch-friendly, sortable list. It is possible to use it with Scala. Here I will provide short instructions how to do that.

## Dependencies

The code is based one the [react-sortable-hoc](https://github.com/dborisenko/scalajs-react-components#react-sortable-hoc) for [React Sortable (HOC)](https://github.com/clauderic/react-sortable-hoc) component. It is written using [scalajs-react](https://github.com/japgolly/scalajs-react) library for ScalaJs.

Add dependencies in `build.sbt`:

```scala
libraryDependencies ++= Seq(
  "com.dbrsn.scalajs.react.components" %%% "react-sortable-hoc" % "0.3.1"
)
npmDependencies in Compile ++= Seq(
  "react-sortable-hoc" -> "1.4.0",
  "react" -> "16.7.0",
  "react-dom" -> "16.7.0"
)
```

## Usage

Example of usage:

```scala
case class Model(text: String)
case class Props(model: Model)

val item1 = "Test 1"
val item2 = "Test 2"

SortableList[Model, Props].Props(
  listToDisplay = List(Model(item1), Model(item2)),
  sortableContainerProps = SortableContainer.Props(),
  externalProps = Props,
  itemComponent = raw
).render
```

## Source Code

Source code can be found [here](https://github.com/dborisenko/scalajs-react-components)
