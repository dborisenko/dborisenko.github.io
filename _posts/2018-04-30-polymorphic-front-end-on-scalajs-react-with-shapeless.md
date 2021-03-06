---
layout: post
title: Polymorphic Front-End on scalajs-react with shapeless
tags: [scalajs-react, react-mobile, ios, web, android, shapeless]
bigimg: /assets/img/256bf2b0-eefc-476b-8041-1609997e124d.jpg
---
Mobile development was historically quite a split area. We do have 2 separate markets of Android and iOS development (or 3? Is Windows Mobile still alive?). There were a lot of mobile frameworks, which were trying to follow the concept "write once — use everywhere" and minifying logical splitting of codebase into 2 (3? 4? 5?) different languages. And the recent attempt of [react-native](https://facebook.github.io/react-native/) seems to be quite successful. [React](https://reactjs.org/) platform with separation of runtimes into DOM and Native seems to be quite a good approach.

I tried to play with react and what I wanted to check is how easy to marry this framework with Scala. And I found great ScalaJs library [scalajs-react](https://gitter.im/japgolly/scalajs-react). Unfortunately, this library supports only rendering to DOM. And I started experimenting, how possible is to implement polymorphic code, which changes its morphism depending on the platform on compile time. At the end of this journey I would like to receive something like this:

```scala
import japgolly.scalajs.react.ScalaComponent
import japgolly.scalajs.react.vdom.html_<^._
import shapeless.Poly1

object ContentControl extends Poly1 {
  private lazy val webComponent = ScalaComponent.builder[ContentId]("Content").render_P { contentId =>
    <.img(^.`class` := "ui fluid image", ^.src := s"/image/content/$contentId.jpg")
  }.build

  implicit def caseWeb: Case.Aux[SemanticUiWebDom, ContentId => VdomElement] = at[SemanticUiWebDom](_ => content => webComponent(content))
}
```

In this code, I use shapeless `Poly1` to make polymorphism for my platform and depending on case decides which component it can render and return it respectively. In this example, we use `SemanticUiWebDom` as a platform definition, which is [Semantic UI](https://semantic-ui.com/) web platform, rendered to DOM directly. For this platform, we can easily return HTML tags with type `VdomElement`. But for react-native, it will be `Image` with underlying `UIImage` for iOS and `android.media.Image` for Android.   

So, the idea is _to create multiple atomic and small controls for each platform and complex composite screens where we just specify what platform (a type of controls) we would like to combine and on compile time we assemble absolutely polymorphic view_. The composite screen might look like this:

```scala
import japgolly.scalajs.react.ScalaComponent
import japgolly.scalajs.react.vdom.html_<^._
import shapeless.Poly1

type CasePlatform[PolyView, Platform, Props] = Case.Aux[PolyView, Platform :: HNil, Props => VdomElement]

object ItemComponent extends Poly1 {
  private def component[Platform](
    implicit
    H: CasePlatform[HeaderControl.type, Platform, HeaderControl.Props],
    I: CasePlatform[ContentControl.type, Platform, ContentId],
    F: CasePlatform[FooterControl.type, Platform, FooterControl.Props],
    FE: CasePlatform[FeedElementLayout.type, Platform, FeedElementLayout.Props]
  ) = ScalaComponent.builder[(Platform, ComponentModel)]("ItemComponent").render_P {
    case (platform, content) =>
      FeedElementLayout(platform).apply(FeedElementLayout.Props(
        header = HeaderControl(platform).apply(HeaderControl.Props(
          content.user.data.avatar, content.user.data.username, content.createdAt
        )),
        content = ContentControl(platform).apply(content.id),
        footer = FooterControl(platform).apply(FooterControl.Props(content.likes))
      ))
  }.build

  private lazy val webComponent = component[SemanticUiWebDom]

  implicit def caseWeb: Case.Aux[SemanticUiWebDom, ComponentModel => VdomElement] = at[SemanticUiWebDom](web => props => webComponent((web, props)))
}
```

And platform cases are just simple traits:

```scala
object Platform {

  sealed trait SemanticUiWebDom

  sealed trait IOSMobileNative

  sealed trait AndroidMobileNative

}
```

I'm only starting playing around with the concepts, but during this experiment, I would like to learn concepts of `react-native` and it's implications with Scala.
