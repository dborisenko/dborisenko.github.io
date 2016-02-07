---
published: false
title: Static Microservice Discovery
layout: post
tags: [akka, cluster, microservice, topology, discovery]
categories: [akka-cluster-microservices]
---
As we saw in our [previous post]({% post_url 2016-02-02-microservice-topologies-in-akka-cluster %}), Akka Cluster gives you some predefined topologies from the box. And it's very easy to perform it's discovery. For example, for [Direct Topology]({% post_url 2016-02-02-microservice-topologies-in-akka-cluster %}#direct-topology) all you need to discover service is to have actor path, including the address of the node. And to discovery microservice with [Cluster Singleton Topology]({% post_url 2016-02-02-microservice-topologies-in-akka-cluster %}#cluster-singleton-topology) is even simpler. You just need to know is _cluster role name_ and _singleton manager name_ (with it's _path_).

The simplest way to make service discovery is just to make class in _service-api_ module, which incapsulates all necessary data. Let's call this method as _static service discovery_ and have a look some examples.

# Cluster Singleton Static Discovery

As we agreed in the [post about decomposition into the modules]({% post_url 2016-01-31-decomposition-of-monolithic-application-to-akka-cluster-microservices %}#decomposition), we are going to split service into service implementation (stored in _service-core_ module) and service gateway (interface in _service-api_ module).

## Singleton Service API Module

{% highlight scala %}
object FooDescriptor {
  val SingletonManagerName = "fooManager"
  val SingletonManagerPath = s"user/$SingletonManagerName"
  val ClusterRole = "foo"
}

object FooProxyFactory {
  val ProxyActorName = "fooProxy"
  
  def proxyProps(settings: ClusterSingletonProxySettings): Props = ClusterSingletonProxy.props(
    singletonManagerPath = FooDescriptor.SingletonManagerPathg,
    settings = settings.withRole(FooDescriptor.ClusterRole)
  )

  def proxyProps(actorSystem: ActorSystem): Props = proxyProps(ClusterSingletonProxySettings(actorSystem))

  def createProxy(actorRefFactory: ActorRefFactory, actorSystem: ActorSystem, name: String): ActorRef = actorRefFactory.actorOf(
    proxyProps(actorSystem),
    name = name
  )

  def createProxy(actorSystem: ActorSystem, name: String = ProxyActorName): ActorRef = createProxy(actorSystem, actorSystem, name)
}
{% endhighlight %}

As we can see from the code above, _FooDescriptor_ contains full information, required by creating proxy actor for access to the Foo Singleton.

## Singleton Service Core Module

As soon as core module has api module as a dependency, we can use information from _FooDescriptor_ to assemble our own singleton.

{% highlight scala %}
object FooManagerFactory {
  def managerProps(settings: ClusterSingletonManagerSettings): Props = ClusterSingletonManager.props(
    singletonProps = Props(injected[FooActor]),
    terminationMessage = PoisonPill,
    settings = settings.withRole(FooDescriptor.ClusterRole)
  )
  
  def managerProps(actorSystem: ActorSystem): Props = managerProps(ClusterSingletonManagerSettings(actorSystem))
  
  def createManager(actorRefFactory: ActorRefFactory, actorSystem: ActorSystem): ActorRef = actorRefFactory.actorOf(
    managerProps(actorSystem),
    name = FooDescriptor.SingletonManagerName
  )

  def createProxy(actorSystem: ActorSystem): ActorRef = createProxy(actorSystem, actorSystem)
}
{% endhighlight %}
