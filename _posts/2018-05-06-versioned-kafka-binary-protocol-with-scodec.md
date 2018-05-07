---
layout: post
title: Versioned Kafka binary protocol with Scodec
tags: [kafka, scodec, message]
bigimg: /img/8125592c-112a-403b-9bd9-8692a17a66cc.jpg
gh-repo: dborisenko/kafka-versioned-scodec
gh-badge: [star, fork, follow]
---

# Kafka binary protocol mess

Immutable data set is nature of Kafka. That is major architectural concept affected all design decisions. And one of the complex problem is to keep message compatibility with old legacy messages after migration to the newest message format. There are multiple serialization and deserialization libraries attempting to solve this complex problem (and none of them is really solving it):

- You can start your journey to Kafka Serialization zoo with [Apache Avro](https://avro.apache.org/). This is library written on Java. The main conceptual idea of it is to separate read scheme and write scheme. Every time when you save message with given avro write schema you need to have this schema in the special registry. When you change your format you can use new write schema. And when your applications reads message from Kafka topic, it loads write schema of that concrete message and the latest read schema. The main challenge here is to keep compatibility between all versions of write schemas and the read schema. I have experience, where all that zoo of schemas became quite hard to maintain due to huge amount of data migrations. Avro does not support versioning from the box. Surprisingly, it somehow became de-facto standard of Kafka message serialization system.

- Another libraries which you will probably try to play with are [Google Protocol Buffers](https://developers.google.com/protocol-buffers/) and [Apache Thrift](https://thrift.apache.org/). That libraries do not have immutable schema (like Avro write schemas). The main idea of that libraries is to append schema every time when you need to change format. You still need to be able to process the old messages, because you don't remove old data type descriptors. The only requirement can be to have all data fields optional. As soon as they are optional you can easily stop writing to old data fields and read and migrate in the new data structure. The main disadvantage of that approach is that you actually spoil your message format with old invalid data fields in the same time with valid data fields. And after checking message format it's hard to see what is actual up to dated message format.

- It's also common to use different Json libraries. Disadvantage of that approach can be similar to that disadvantages described in the previous section. This approach can depend on the implementation details of your underlying library and it's ability to handle absence or nullable fields.

This domain is quite complex. The common practice is to have tolerant consumer who is able to read all message in all possible versions. Apart from the serialization issues you might face to the issues of different message schemas on the consumer and producer side (you will probably define deployment order to fix that) and other issues.

# The Scala way

As always, Scala (more precisely, Typelevel stack) can help us to simplify this huge mess in traditionally concise, generic and clean way. Typelevel stack contains pretty old but still nice library [scodec](https://github.com/scodec/scodec). Scodec is scodec is a combinator library for working with binary data. It focuses on contract-first and pure functional encoding and decoding of binary data and provides integration into shapeless. It does not support versioning from the box and it's really pretty simple. But it's simplicity is the main advantage. Together with composability.

Let's start with the example of usage. Let's define simple data structure `User` (of version v1):

```scala
/**
 * User data structure v1.
 */
final case class User(
  email: String,
  name: Option[String],
  activated: Boolean
)
```

Let's create a simple codec for this data type using scodec.

```scala
import scodec.codecs._
import scodec.Codec
import shapeless.{ ::, HNil }

val userCodecV1: Codec[User] = (utf8_32 :: optional(bool, utf8_32) :: bool).xmap(
  { case email :: name :: activated :: HNil => User(email, name, activated) },
  u => u.email :: u.name :: u.activated :: HNil
)
```

We started thinking about future evolution of data structure in advance. We did not want to involve any complex evolution scenarious or any schema generations. We just want to keep our codec as much as possible in code. And we can start using a [versioned codec wrapper](https://github.com/dborisenko/kafka-versioned-scodec/blob/676ac520525c54f57f784d4a85c44ef7ca303e14/src/main/scala/com/dbrsn/versioned/VersionedCodec.scala) which allows us to do a simple versioning of all our possible formats. The example of code can be found [here](https://github.com/dborisenko/kafka-versioned-scodec/blob/676ac520525c54f57f784d4a85c44ef7ca303e14/src/test/scala/com/dbrsn/versioned/VersionedCodecSpec.scala).

After some time we might come to the idea that we need to add additional field to this data type. Let's say, we want to store total number of posts. Our data type might become:

```scala
/**
 * User data structure v2.
 */
final case class User(
  email: String,
  name: Option[String],
  activated: Boolean,
  numberOfPosts: Long
)

import scodec.codecs._
import scodec.Codec
import shapeless.{ ::, HNil }

// Here we make sure that old data produce correct new data type. Let's assume our statrtup number of posts is 0L.
val userCodecV1: Codec[User] = (utf8_32 :: optional(bool, utf8_32) :: bool).xmap(
  { case email :: name :: activated :: HNil => User(email, name, activated, 0L) },  // This is our migration schema because it produces new correct data type User.
  u => u.email :: u.name :: u.activated :: HNil
)

// But the new message codec already knows how to encode the new data type
val userCodecV2: Codec[User] = (utf8_32 :: optional(bool, utf8_32) :: bool :: int64).xmap(
  { case email :: name :: activated :: numberOfPosts :: HNil => User(email, name, activated, numberOfPosts) },
  u => u.email :: u.name :: u.activated :: u.numberOfPosts :: HNil
)
```

It's easy to see that here in one-liner we provide new version of codec with the newest field included and in another one-liner we provide the migration schema for to build new data type from the old stored binary. It is pretty easy, isn't it? And let's bundlem both of that versions together:

```scala
import com.dbrsn.versioned.VersionedCodec

val versionedCodec: Codec[User] = new VersionedCodec[User](
  entityIdentity = "User",  // Used as prefix in the binary file. Easy way to make sure that file format belongs to the given entity.
  currentVersion = 2,
  codecByVersion = {
    case 1 => userCodecV1
    case 2 => userCodecV2
  }
)
```

And this is how we can use it:

```scala
import scodec.Err
import scodec.bits.BitVector

val inUser: User = User("email@email.com", Some("Denis"), true, 123L)

val binUser: Either[Err, BitVector] = versionedCodec.encode(inUser).toEither
val outUser: Either[Err, User] = versionedCodec.decodeValue(binUser.right.get).toEither
```

This format is binary only. So, it does not have human readable representation. This can be counted as disadvantage of this approach.

This is how our user from the example looks like in HEX representation: 

```text
000000045573657200020000000f656d61696c40656d61696c2e636f6d80000002a232b734b9c00000000000001ec
```

And this is how it looks like in binary view:

```text
00000000: 0000 0004 5573 6572 0002 0000 000f 656d  ....User......em
00000010: 6169 6c40 656d 6169 6c2e 636f 6d80 0000  ail@email.com...
00000020: 02a2 32b7 34b9 c000 0000 0000 001e c0    ..2.4..........
```
