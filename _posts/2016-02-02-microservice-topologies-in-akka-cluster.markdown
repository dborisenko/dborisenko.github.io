---
published: true
title: Microservice Topologies in Akka Cluster
layout: post
tags: [akka, cluster, microservice, topology, direct, cluster-singleton, cluster-sharding, cluster-router-group]
categories: [akka-cluster-microservices]
---
Service discovery is an essential aspect for microservice architecture. Does usage of a Akka Cluster simplify this problem for us? To answer this question, first of all we need to analyse data flows within a cluster and understand how different elements (service nodes or clients) of a microservice can be arranged inside a cluster. Let's call this arrangement topology.

_Microservice topology is the arrangement of the various elements of a cluster_

Let's describe some of the possible microservice topologies within an Akka Cluster.

# Direct topology

Every _Actor_ in a cluster has a path, which includes it's network address. And this path is a part of serializable _ActorRef_ value. In this way you can have an actor, running on any machine and knowing it's reference _ActorRef_ is enough to send message directly to that actor from any other machine. Let's call this topology _direct_.

# Cluster Singleton Topology

This topology is based on [Cluster Singleton](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-singleton.html) approach. In this topology we ensure that we have exactly one actor of a certain type running somewhere in the cluster.

# Cluster Sharding Topology

This topolog is based on [Cluster Sharding](http://doc.akka.io/docs/akka/2.4.1/scala/cluster-sharding.html) approach. Cluster sharding is useful when we need to distribute actors across several nodes in the cluster and want to be able to interact with them using their logical identifier, but without having to care about their physical location in the cluster, which might also change over time.

# Cluster Router Group Topology

This topology is based on [Cluster Aware Routers with Group of Routees](http://doc.akka.io/docs/akka/snapshot/java/cluster-usage.html#Cluster_Aware_Routers). Router sends messages to the specified path using actor selection. The routees can be shared between routers running on different nodes in the cluster.

# Other Topologies

The list of the topologies is of course not complete. There might be other different variations, like as message queue based topology (ZMQ, NSQ), involving Apache Kafka, etc.
