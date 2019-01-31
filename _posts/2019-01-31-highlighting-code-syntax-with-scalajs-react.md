---
layout: post
title: Highlighting code syntax with scalajs-react
tags: [react-syntax-highlighter, syntax, highlight, react, scalajs]
gh-repo: dborisenko/scalajs-react-components
gh-badge: [star, fork, follow]
---

React Syntax Highlighter is a syntax highlighting component for react with prismjs or highlightjs ast using inline styles. It is possible to use it in Scala. Here I will provide short instructions how to do that.

## Dependencies

The code is based one the [react-syntax-highlighter wrapper](https://github.com/dborisenko/scalajs-react-components#react-syntax-highlighter) for [React Syntax Highlighter](https://www.npmjs.com/package/react-syntax-highlighter) component. It is written using [scalajs-react](https://github.com/japgolly/scalajs-react) library for ScalaJs.

Add dependencies in `build.sbt`:

```scala
libraryDependencies ++= Seq(
  "com.dbrsn.scalajs.react.components" %%% "react-syntax-highlighter" % "0.3.1"
)
npmDependencies in Compile ++= Seq(
  "react-syntax-highlighter" -> "10.1.2",
  "babel-runtime" -> "6.26.0",
  "react" -> "16.7.0",
  "react-dom" -> "16.7.0"
)
```

## Usage

Example of usage:

For highlightjs-based component:

```scala
SyntaxHighlighter(HljsLanguage.javascript, style = HljsStyle.docco)("(num) => num + 1")
```

For prismjs-based component:

```scala
PrismSyntaxHighlighter(PrismLanguage.javascript, style = PrismStyle.dark)("(num) => num + 1")
```

## Source Code

Source code can be found [here](https://github.com/dborisenko/scalajs-react-components)
