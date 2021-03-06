---
layout: post
title: How the interop between Scala.js and TypeScript can be simplified
tags: [scalajs, typescript, interop, idea, semanticdb]
published: false
---

The typical way of interop between Scala.js and JavaScript is to implement wrappers for all functions and components annotated with `@js.native` in the places where runtime-level JavaScript logic is called. Such a wrapper can be considered as a type definition for the underlying JavaScript component. In a similar way TypeScript interop with JavaScript. You need to make `*.d.ts` files, where you specify just types of underlying JavaScript components. Looking from that perspective, TypeScript and Scala.js interop with JavaScript in a very similar way. The only difference is that TypeScript is very widespread language between front-end developers. But Scala.js keeps to be very niche technology. Due to this reason we can find a lot of TypeScript components and not so many Scala.js based components.

In the following post I will try to make thoughts and propose ideas how the interop between Scala.js and TypeScript can be improved. I don't have any concrete instructions or implementations which can help to do our life easier. I'm not even sure that it's possible to implement it in a such a way. But let me just start.

# The Problem

I have an experience of writing scalajs components (in particular, using great [scalajs-react](https://github.com/japgolly/scalajs-react) library). And I can tell you, if you want to use any react components (they are usually written using TypeScript) in your Scala.js application you have to write a lot of boilerplate code. You need a wrapper for each small class/type/function which you are going to use in your application. Such a code is usually not very sophisticated, but it's very annoying to write this useless boilerplate. It's probably also the same if you want to use your Scala.js code inside your TypeScript application. But it's probably not so common and I will keep this topic out of the scope of my post. Here I will try to reason how the interop between Scala.js and TypeScript can be simplified.

# The solution idea

The main idea is in making TypeScript 1st class citizen of Scala ecosystem. In such a way as Java components are spreaded across the whole Scala platform. The standard library is full of Java based components. `String`, `DateTime`, value types, arrays, and so on. All such components are Java based. And to make them life in your scala-js application, the authors of scala-js created a lot of wrappers for them. You can [check scala-js repository](https://github.com/scala-js/scala-js/tree/da24eaa3dd45d2485fe17b9e177e6388a1b97eca/javalanglib/src/main/scala/java/lang) to make your impression how Java is spreaded inside Scala ecosystem. It is fine for such JVM based platform as Scala. But after making Scala not only JVM based, we can borrow the idea of such an easy interop between two absolutely different worlds. And let's investigate how it is possible to make Scala language more friendly with TypeScript.

## TypeScript compiler

First of all, let's check conceptually, is it possible to implement TypeScript language constructions in terms of Scala language. After checking [TypeScript Language Specification](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md) we can build the table of TypeScript and compatible Scala language constructions. We are not going to write complete compiler of TypeScript to Scala.js. But instead we can focus only on [TypeScript declarations](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#23-declarations) (`d.ts` files or component definitions without implementations). That components are the most interesting for us and allow us to merge TypeScript ecosystem into Scala.js ecosystem. Even if component has an actual implementation, for purpose of this post we can ignore it and deal only with definitions.

From the [grammar specification](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#a9-scripts-and-modules) we can derive our tables. In the first table we specify root elements of the grammar.

| Grammar name                                                                                                                                                              | Definition                                                                                                                                                                                            |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [InterfaceDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#71-interface-declarations)                       | `interface` BindingIdentifier TypeParameters<sub>opt</sub> InterfaceExtendsClause<sub>opt</sub> ObjectType                                                                                            |
| [TypeAliasDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#310-type-aliases)                                | `type` BindingIdentifier TypeParameters<sub>opt</sub> `=` Type `;`                                                                                                                                    |
| [NamespaceDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#101-namespace-declarations)                      | `namespace` IdentifierPath `{` NamespaceBody `}`                                                                                                                                                      |
| [AmbientDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#121-ambient-declarations)                          | `declare` AmbientVariableDeclaration<br/>`declare` AmbientFunctionDeclaration<br/>`declare` AmbientClassDeclaration<br/>`declare` AmbientEnumDeclaration<br/>`declare` AmbientNamespaceDeclaration    |
| [ImportAliasDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#103-import-alias-declarations)                 | `import` BindingIdentifier `=` EntityName `;`                                                                                                                                                         |
| [AmbientModuleDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#122-ambient-module-declarations)             | `declare` `module` StringLiteral `{`  DeclarationModule `}`                                                                                                                                           |
| [ImportDeclaration](http://www.ecma-international.org/ecma-262/6.0/#sec-imports)                                                                                          | `import` ImportClause FromClause `;`<br/>`import` ModuleSpecifier `;`                                                                                                                                 |
| [ExportDeclarationElement](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#11341-export-modifiers)                      | `export` InterfaceDeclaration<br/>`export` TypeAliasDeclaration<br/>`export` AmbientDeclaration<br/>`export` ImportAliasDeclaration                                                                   |
| [ExportDefaultDeclarationElement](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#11342-export-default-declarations)    | `export` `default` AmbientFunctionDeclaration<br/>`export` `default` AmbientClassDeclaration<br/>`export` `default` IdentifierReference `;`                                                           |
| [ExportListDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#11343-export-list-declarations)                 | `export` `*` FromClause `;`<br/>`export` ExportClause FromClause `;`<br/>`export` ExportClause `;`                                                                                                    |
| [ExportAssignment](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#1135-export-assignments)                             | `export` `=` IdentifierReference `;`                                                                                                                                                                  |

And in the following table we can define some sub-elements of the previous grammar.

| Grammar name                                                                                                                                                          | Definition                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| [AmbientVariableDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#1211-ambient-variable-declarations)    | `var` AmbientBindingList `;`<br/>`let` AmbientBindingList `;`<br/>`const` AmbientBindingList `;`  |
| [AmbientFunctionDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#1212-ambient-function-declarations)    | `function` BindingIdentifier CallSignature `;`                                                    |
| [AmbientClassDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#1213-ambient-class-declarations)          | `class` BindingIdentifier TypeParameters<sub>opt</sub> ClassHeritage `{` AmbientClassBody `}`     |
| [AmbientEnumDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#1214-ambient-enum-declarations)            | `const`<sub>opt</sub> `enum` BindingIdentifier `{` EnumBody<sub>opt</sub> `}`                     |
| [AmbientNamespaceDeclaration](https://github.com/Microsoft/TypeScript/blob/f30e8a284ac479a96ac660c94084ce5170543cc4/doc/spec.md#1215-ambient-namespace-declarations)  | `namespace` IdentifierPath `{` AmbientNamespaceBody `}`                                           |

## Scala.js Analogues

We can easily see that all definitions in TypeScript are compatible with Scala.js. Even more, that is how Scala.js wrappers are actually created. Typical example of the wrapper can be found [here](https://github.com/dborisenko/scalajs-react-components/blob/9ca4f774114df75ffdfe7dac853be85011aec1d3/storm-react-diagrams/src/main/scala/com/dbrsn/scalajs/react/storm/diagrams/StormReactDiagrams.scala)

```scala
@js.native
@JSImport("storm-react-diagrams", JSImport.Namespace)
object StormReactDiagrams extends js.Object {
  // ...
  @js.native
  class PointModel extends BaseModel {
    def isConnectedToPort(): Boolean = js.native
    def getLink(): LinkModel = js.native
    def getX(): Double = js.native
    def getY(): Double = js.native
  }
  // ...
}
```

Which is quite similar to the real [TypeScript source code](https://github.com/projectstorm/react-diagrams/blob/5b142c1c6136eff75b81a819b4b73de644c307ab/src/models/PointModel.ts). So, the easiest approach is to (maybe even automatically) create wrapper for each `.ts` or `.d.ts` TypeScript source file.

## SemanticDB

Can TypeScript source files be used inside Scala.js application without generating of such a huge amount of wrappers boilerplate code? Maybe. 

## SBT plugin
