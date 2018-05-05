---
layout: post
title: Versioned Kafka binary protocol with Scodec
tags: [kafka, scodec, message]
bigimg: /img/8125592c-112a-403b-9bd9-8692a17a66cc.jpg
---

# Kafka binary protocol mess

Immutable data set is nature of Kafka. That is major architectural concept affected all design decisions. And one of the complex problem is to keep message compatibility with old legacy messages after migration to the newest message format. There are multiple serialization and deserialization libraries attempting to solve this complex problem (and none of them is really solving it):

- You can start your journey to Kafka Serialization zoo with [Apache Avro](https://avro.apache.org/). This is library written on Java. The main conceptual idea of it is to separate read scheme and write scheme. Every time when you save message with given avro write schema you need to have this schema in the special registry. When you change your format you can use new write schema. And when your applications reads message from Kafka topic, it loads write schema of that concrete message and the latest read schema. The main challenge here is to keep compatibility between all versions of write schemas and the read schema. I have experience, where all that zoo of schemas became quite hard to maintain due to huge amount of data migrations. Avro does not support versioning from the box. Surprisingly, it somehow became de-facto standard of Kafka message serialization system.

- Another libraries which you will probably try to play with are [Google Protocol Buffers](https://developers.google.com/protocol-buffers/) and [Apache Thrift](https://thrift.apache.org/). That libraries do not have immutable schema (like Avro write schemas). The main idea of that libraries is to append schema every time when you need to change format. You still need to be able to process the old messages, because you don't remove old data type descriptors. The only requirement can be to have all data fields optional. As soon as they are optional you can easily stop writing to old data fields and read and migrate in the new data structure. The main disadventure of that approach is that you actually spoil your message format with old invalid data fields in the same time with valid data fields. And after checking message format it's hard to see what is actual up to dated message format.

- It's also common to use different Json libraries. Disadvantage of that approach can be similar described in the previous section. This approach can depend on the implementation details of your underlying library and it's ability to handle absent or nullable fields.

This domain is quite complex. The common practice is to have tolerant consumer who is able to read all message in all possible versions. Apart from the serialization issues you might face to the issues of different message schemas on the consumer and producer side (you will probably define deployment order to fix that) and other issues.

# The Scala way

As always, Scala (more precisely, Typelevel stack) can help us to simplify this huge mess in traditionally concise, generic and clean way. Typelevel stack contains pretty old but still nice library [scodec](https://github.com/scodec/scodec). Scodec is scodec is a combinator library for working with binary data. It focuses on contract-first and pure functional encoding and decoding of binary data and provides integration into shapeless. It does not support versioning from the box and it's really pretty simple. But it's simplicity is the main advantage. Together with composability.

