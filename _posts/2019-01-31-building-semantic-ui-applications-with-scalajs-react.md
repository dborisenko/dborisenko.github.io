---
layout: post
title: Building Semantic-UI applications with scalajs-react
tags: [semantic-ui, react, scalajs]
gh-repo: dborisenko/scalajs-react-components
gh-badge: [star, fork, follow]
---

# Semantic UI React

Semantic is a UI framework designed for theming. It is possible to build nice and beautiful UI using Scala language. Here I will provide short instructions how to do that.

## Dependencies

The code is based one the [semantic-ui-react wrapper](https://github.com/dborisenko/scalajs-react-components#semantic-ui-react) for [Semantic UI React](https://react.semantic-ui.com/) component. It is written using [scalajs-react](https://github.com/japgolly/scalajs-react) library for ScalaJs.

Add dependencies in `build.sbt`:

```scala
libraryDependencies ++= Seq(
  "com.dbrsn.scalajs.react.components" %%% "semantic-ui-react" % "0.3.1"
)
npmDependencies in Compile ++= Seq(
  "semantic-ui-react" -> "0.84.0",
  "react" -> "16.7.0",
  "react-dom" -> "16.7.0"
)
```

## Usage

Example of usage:

```scala
SuiButton(animated = true, onClick = (_: ReactMouseEventFromHtml) => Callback(???))(
  SuiButtonContent(visible = true)("Hello, World!"),
  SuiButtonContent(hidden = true)(SuiIcon(name = SuiIconType("arrow right"))())
)
```

Don't forget to add styles to your html:

```html
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/semantic-ui@2.4.1/dist/semantic.min.css" />
```

## Source Code

Source code can be found [here](https://github.com/dborisenko/scalajs-react-components)
