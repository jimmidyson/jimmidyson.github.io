+++
date = "2015-04-09T15:24:56+01:00"
draft = true
title = "Clustering on Kubernetes & OpenShift3"
topics = [ "clustering" ]
tags = [ "openshift", "fabric8", "kubernetes", "elasticsearch" ]
+++

[openshift]: http://www.openshift.org/ "OpenShift by Red Hat"
[kubernetes]: http://kubernetes.io/ "Kubernetes - clustered Linux containers"
[elasticsearch]: https://www.elastic.co/ "Elasticsearch - distributed, open source search and analytics engine"
[fabric8]: http://fabric8.io/

One of the big promises of Kubernetes & OpenShift3 is really easy management of your
containerised applications. For standalone or load-balanced stateless applications,
Kubernetes works brilliantly, but one thing that I had a bit of trouble figuring
out was how do perform cluster discovery for my applications? Say one of my applications
needs to know about at least one other node (seed node) that it should join a cluster with.
I'm going to use Elasticsearch through this post as it's awesome horizontal scalability
lends itself very well to hopefully explaining this clearly. This is a real
example that we provide as part of Fabric8 to build up Elasticsearch clusters.

Before we go any further, a quick recap of 3 Kubernetes concepts we'll be using
in this post (taken from the excellent Kubernetes documentation):

1. _Pods_ are the smallest deployable units that can be created, scheduled,
   and managed. Pods are a colocated group of containers that run on the same
   physical host & share stuff like network space (e.g. IP address) & disk (via volumes). [Read more]
   (https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pods.md)
2. _Replication controllers_ (RCs) ensure that a
   specified number of pod "replicas" are running at any one time. If there are
   too many, it will kill some. If there are too few, it will start more.
   [Read more](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/replication-controller.md)
3. _Services_ are an abstraction which defines a logical set of Pods
   and a policy by which to access them. The
   set of Pods targeted by a Service is determined by a Label Selector. A service is used to access a group of pods through a consistent
   endpoint without knowing the exact pods, especially important considering the
   ephemeral nature of pods. [Read more](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md)

I'm going to assume you have a working Kubernetes cluster up & running. If you
don't then you can really easily set one up on any Docker-enabled host via a
script that Fabric8 provides (see
[here](http://fabric8.io/guide/openShiftDocker.html#run-the-start-script) if
you're interested). Note that the script will actually spin up OpenShift3
rather than vanilla Kubernetes as Fabric8 uses some of the extensions that
OpenShift provides, like builds, deployment pipelines, etc for other things.
Everything below will work on vanilla Kubernetes of course.


