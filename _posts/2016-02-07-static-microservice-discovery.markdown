---
published: false
title: Static Microservice Discovery
layout: post
tags: [akka, cluster, microservice, topology, discovery]
categories: [akka-cluster-microservices]
---
As we saw in our [previous post]({% post_url 2016-02-02-microservice-topologies-in-akka-cluster %}), Akka Cluster gives you some predefined topologies directly from the box. And it's very easy to perform it's discovery. For example, for [Direct Topology]({% post_url 2016-02-02-microservice-topologies-in-akka-cluster %}#direct-topology) all you need to discover service is to have actor path, including the address of the node. And to discovery microservice with [Cluster Singleton Topology]({% post_url 2016-02-02-microservice-topologies-in-akka-cluster %}#cluster-singleton-topology) is even simpler. You just need to know is _cluster role name_ and _singleton manager name_ (with it's _path_). So, 