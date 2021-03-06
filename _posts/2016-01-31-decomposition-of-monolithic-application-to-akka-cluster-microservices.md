---
layout: post
title: Decomposition of monolithic application to Akka Cluster microservices
tags: [akka, cluster, microservice, decomposition]
---
In the first part of [series of articles about Akka Cluster Microservices](/2016-01-30-akka-cluster-microservices), I would like to cover question of decomposition of a monolithic application. First of all, we need to understand what is monolithic application and what are micriservices. Also we need to define a goal and principles and practices to achieve this goal. So, let's start.

# The Monolith

Let's invent our abstract monolithic Scala application, mostly written in message-driven manner over [Akka](http://akka.io/) [actor model](https://en.wikipedia.org/wiki/Actor_model), [Play Framework](https://www.playframework.com/) for controller layer and [Akka Cluster](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-usage.html) for nodes coordination. Let's add good unit and integration test coverage (>80%) and highly qualified agile team with [UAT](https://en.wikipedia.org/wiki/Acceptance_testing#User_acceptance_testing) as a part of a team workflow. Too perfect conditions? Okay, then let's add some complexity. Application is highly coupled, violating almost all [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) principles, missing strong [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) (for example, no service layer at all). All source code is in one repository without splitting into the modules, and so on. Classical monolith.

Before starting our dive, first of all we need to define a goals.

# Strategic Goal

Our goal will be quite simple: _split the monolith application to microservices_.

To realise what do we need, first of all, we need to understand what is microservice. The definition, given by Sam Newman in his book ["Building Microservices. Designing Fine-Grained Systems"](http://shop.oreilly.com/product/0636920033158.do) is the following: _microservices are small, autonomous services that work together_. I also like the platform-agnostic golden rule of defining microservices: _microservice is something that could be rewritten in two weeks_.

For our concrete Actor-based application, the typical microservice can be described by the following figure:

![Example of Microservice Data Flow](/resources/2016-01-31-decomposition-of-monolithic-application-to-akka-cluster-microservices/microservice-data-flow.png "Example of Microservice Data Flow")

## Key Benefits of microservice architecture

There are a lot of articles, proving that microservice architecture is something, what you need. The key benefits of this approach are:

* Technical heterogeneity
* Resilience
* Scaling
* Ease of Deployment
* Composability
* Optimizing for replaceability

# Architectural principles and Design and Delivery Practices

To understand, that we are moving in the right direction, we need to define some platform-specific and application-specific principles and practices to follow. This section might be extended due some other principles and practices might be discovered during the implementation.

## Cross-modular versioning is important

As soon as we are going to deal with multiple instances of microservices, which are going to be modularized, we always need to know which version of module is running on the microservice. This might be important when we face to the problem, that after some api or model changes, some modules become incompatible with the other modules. We always need to keep in mind the version compatibilities. There are a lot of [versioning methodologies](https://en.wikipedia.org/wiki/Software_versioning), that might be useful for the concrete task.

## Akka Cluster as a transport

This practice might be against the technology-agnostic approach of designing microservices: the only supported platform is JVM and mostly common used language is Scala. But let's make this choice because of _asynchronous message-driven style of collaboration_, implemented in Akka and extended to Akka Cluster. This style is also good to follow in the concrete module implementation. It can help to build [reactive](http://www.reactivemanifesto.org/), distributed and highly scalable application. Of course, there are a lot of other choices: request/response synchronous and asynchronous RPC (SOAP, Thrift, Protocol Buffers) or also request/response based REST (over HTTP). But in this article we are going to stay only on Akka and Akka Cluser message based approach.

## Solution preferences

Let's define some list of priorities, which approach is most preferable. The key principle here will be to choose the most flexible and scalable solution for a specific tasks.

* Stateless actor in a pool on a node is preferable to [Cluster Sharding](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-sharding.html).
* [Cluster Sharding](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-sharding.html) is preferable to [Cluster Singleton](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-singleton.html)

## Separate public messages and API from internal ones

In order to have API decoupled from the process module, we need to keep internal messages and API independent from external, even if the functionality is duplicated. Internal messages might be changed quite often and API messages shouldn't reflect this changes. API need be backward compatible as long as possible.

# Decomposition

First of all, we need to understand, how we need to split the monolith application into the modules. What strategy need to be applied to keep them loosely coupled and highly cohesive? We, of course, need some shared util and runtime libraries. But more shared components we have, more highly coupled application we will get at the end point. So, let's try to find a golden balance for our concrete task and try to keep the dependency list as small as possible. 

We need to extract the core microservice logic into the module(s) and keep the monolith application working properly and all unit tests passed successfully.

In my opinion, the best solution will be building a microservice based on the following SBT modules:

## Model

Models, shared between microservices. So, it might contain following components:

* Model classes and their companion objects.
* Constants.
* Serialization/deserialization, formats of shared models (let's choose JSON for simplicity sake).
* Utils. Unfortunately, some companion objects (for example, parsers or _apply()_ methods) requires some util components.
* Unit tests. We need to make sure, that serialization/deserialization or utils work fine.

## API

Interface to the corcrete microservice. Should contain enough information to make microservice lookup and discovery. All clients of this microservice should include this module as a dependency. Regarding storing public messages, there are 2 possible implementations:

### Gateway

Module contains only information, which is enough to make microservice lookup and discovery (service topology data). It doesn't contain public messages and their serializers. They should be stored in [Model](#model). In this case [Core](#core) shouldn't have this module in the dependencies.

### Protocol

Module contains information, which is enough to make microservice lookup and discovery (service topology data). It also contains public messages and their serializers, etc. In this case [Core](#core) should be dependent on this module.

## Core

Main microservice logic. Depends on the [Model](#model) and for some concrete implementations on the [API](#api) modules. Should be able to handle public messages. Internal messages shoulnd't be exposed

The figure, which show the dependencies between different microservices with the related modules.

![Microservices dependencies](/resources/2016-01-31-decomposition-of-monolithic-application-to-akka-cluster-microservices/microservice-dependencies.png "Microservices dependencies")

# Conclusion

In this article I tried to cover the first round of decomposition of the monolithic application into the microservices. Of course, it's only beginning of the big journey and probably some other decompositions will be required. In the next articles I'll try to cover questions of different Akka Cluster topologies and some methods of microservices discovery.
