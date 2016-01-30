---
published: true
title: Akka Cluster Microservices: The Monolith
layout: post
tags: [akka, cluster, microservice]
categories: [akka-cluster-microservices]
---
By this post I would like to start series of articles about designing and implementing microservices on top of Akka Cluster. All the following is my personal thought, based on my experience.

Let's invent our abstract monolithic Scala application, mostly written in message-driven manner over [Akka](http://akka.io/) [actor model](https://en.wikipedia.org/wiki/Actor_model), [Play Framework](https://www.playframework.com/) for controller layer and [Akka Cluster](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-usage.html) for nodes coordination. Let's add good unit and integration test coverage (>80%) and highly qualified agile team with [UAT](https://en.wikipedia.org/wiki/Acceptance_testing#User_acceptance_testing) as a part of a team workflow. Too perfect conditions? Okay, then let's add some complexity. Application is highly coupled, violating almost all [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) principles, missing strong [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) (for example, DAO query builder and database connection injected directly into the controller, no service layer at all). All source code is in one repository without splitting into the modules, and so on. Classical monolith.

Before starting our dive, first of all we need to define a goals.

#Strategic Goals

Our goal will be quite simple: _split the monolith application to microservices_.

To realise what do we need, first of all, we need to understand what is microservice. Microservices are small, autonomous services that work together. Personally, I also like the platform-agnostic golden rule of defining microservices, given by Sam Newman in his book ["Building Microservices. Designing Fine-Grained Systems"](http://shop.oreilly.com/product/0636920033158.do): Microservice is something that could be rewritten in two weeks.

##Key Benefits of microservice architecture

There are a lot of articles, proving that microservice architecture is something, what you need. The key benefits of this approach are:

* Technical heterogeneity
* Resilience
* Scaling
* Ease of Deployment
* Composability
* Optimizing for replaceability

#Architectural principles and Design and Delivery Practices

To understand, that we are moving in the right direction, we need to define some platform-specific and application-specific principles and practices to follow.

##Cross-modular versioning is important

As soon as we are going to deal with multiple instances of microservices, which are going to be in the separate module (or modules), we always need to know which version of module is running on which microservice. This might be important when we face to the problem, that after some api or model changes, some modules become incompatible with the other modules. We always need to keep in mind the version compatibilities. There are a lot of [versioning methodologies](https://en.wikipedia.org/wiki/Software_versioning), that might be useful for the concrete task.

##Solutions preferences

Let's define some list of priorities, which approach will be most preferable for us. The key principle here will be to choose the most flexible and scalable solution for a specific tasks.

* Stateless actor in a pool on a node is preferable to [Cluster Sharding](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-sharding.html).
* [Cluster Sharding](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-sharding.html) is preferable to [Cluster Singleton](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-singleton.html)

This section might be extended due some other principles and practices might be discovered during the implementation.

