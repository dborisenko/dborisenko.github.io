---
layout: post
title: Building Diagrams with scalajs-react
tags: [storm-react-diagrams, diagrams, react, scalajs]
gh-repo: dborisenko/scalajs-react-components
gh-badge: [star, fork, follow]
---

# STORM React Diagrams

STORM React Diagrams is a super simple, no-nonsense diagramming library written in React that just works. It is possible to use it in Scala. Here I will provide short instructions how to do that.

## Dependencies

The code is based one the [storm-react-diagrams wrapper](https://github.com/dborisenko/scalajs-react-components#storm-react-diagrams) for [STORM React Diagrams](https://www.npmjs.com/package/storm-react-diagrams) component. It is written using [scalajs-react](https://github.com/japgolly/scalajs-react) library for ScalaJs.

Add dependencies in `build.sbt`:

```scala
// 1) setup the diagram engine
val engine = new DiagramEngine()
engine.installDefaultFactories()

// 2) setup the diagram model
val model = new DiagramModel()

// 3) create a default node
val node1 = new DefaultNodeModel("Node 1", "rgb(0,192,255)")
val port1 = node1.addOutPort(s"Out")
node1.setPosition(pos1x, pos1y)

// 4) create another default node
val node2 = new DefaultNodeModel("Node 2", "rgb(192,255,0)")
val port2 = node2.addInPort(s"In")
node2.setPosition(pos2x, pos2y)

// 5) link the ports
val link1 = port1.link(port2)

// 6) add the models to the root graph
model.addAll(node1, node2, link1)

// 7) load model into engine
engine.setDiagramModel(model)

DiagramWidget(diagramEngine = engine)()
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
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/storm-react-diagrams@5.2.1/dist/style.min.css" />
```

## Source Code

Source code can be found [here](https://github.com/dborisenko/scalajs-react-components)
