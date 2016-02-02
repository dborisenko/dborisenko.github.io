---
published: false
title: Microservice Topologies in Akka Cluster
layout: post
tags: [akka, cluster, microservice, topology, direct, cluster-singleton, cluster-sharding, cluster-router-group]
categories: [akka-cluster-microservices]
---
Service discovery is an essential aspect for microservice architecture. Does usage of a Akka Cluster simplify this problem for us? To answer this question, first of all we need to analyse data flows within a cluster and how different elements (service nodes or clients) of a microservice can be arranged inside a cluster. Let's call this arrangement topology.

_Microservice topology is the arrangement of the various elements of a cluster_

# Direct topology

Every _Actor_ in a cluster has a path, which includes it's network address. And this path is a part of serializable _ActorRef_ value. In this way you can have an actor, running on any machine and knowing it's reference _ActorRef_ is enough to send message directly to that actor from any other machine. Let's call this topology _direct_ and illustrate it by the following figure:

![Example of a Direct Topology](/resources/ "Example of a Direct Topology")

# Cluster Singleton Topology

# Cluster Sharding Topology

# Cluster Router Group Topology
