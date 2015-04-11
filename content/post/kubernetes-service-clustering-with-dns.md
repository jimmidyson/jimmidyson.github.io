+++
date = "2015-04-09T15:24:56+01:00"
draft = true
title = "Clustering on Kubernetes & OpenShift3 using DNS"
topics = [ "clustering" ]
tags = [ "openshift", "fabric8", "kubernetes", "elasticsearch" ]
+++

[openshift]: http://www.openshift.org/ "OpenShift by Red Hat"
[kubernetes]: http://kubernetes.io/ "Kubernetes - clustered Linux containers"
[elasticsearch]: https://www.elastic.co/ "Elasticsearch - distributed, open source search and analytics engine"
[fabric8]: http://fabric8.io/

## tl;dr
1. Set up a [Kubernetes][kubernetes]/[OpenShift][openshift] cluster with DNS enabled.
2. Create your application image that performs a DNS lookup to find cluster nodes.
3. Create a service to access your cluster: target it only at the nodes that
should be accessible by clients (e.g. [Elasticsearch][elasticsearch] client nodes).
4. Create a headless service that uses a common label superset - use the service
name as the DNS entry that your application image looks up to find cluster nodes.
5. Create replication controllers for your pods. If you have multiple pod types
that should form part of the same cluster remember to use a common superset for
your labels.

## The details

One of the big promises of [Kubernetes][kubernetes] & [OpenShift][openshift] is really easy management of
your containerised applications. For standalone or load-balanced stateless
applications, Kubernetes works brilliantly, but one thing that I had a bit of
trouble figuring out was how do perform cluster discovery for my applications?
Say one of my applications needs to know about at least one other node (seed
node) that it should join a cluster with.

There is an [example in the Kubernetes repo](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/cassandra) for Cassandra here
that requests existing service endpoints from the Kubernetes API
server & use those as the seed servers. You can see the code for it
[here](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/cassandra#seed-provider-source).
That works great for a cluster that allows unauthenticated/unauthorized access
to the API server, but hopefully most people are going to lock down their API
server (OpenShift comes with auth baked in by the way & secure by default). If
you're going to secure your API server then you're going to have to distribute
credentials via
[secrets](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/design/secrets.md)
around to every container that wants to call the API server. Personally I'd
rather only distribute secrets when absolutely necessary: if there's a
way to achieve what we need to achieve without distributing secrets then I would
prefer to do that.

It would be cool if Kubernetes had the concept of cluster seeds baked in & could
provide seeds to pods through configuration, but right now it can't so we're going to take
advantage of a couple of things that Kubernetes provides to do that: [_headless
services_](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md#headless-services)
& [_DNS_](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/dns.md).

Before we go any further, a quick recap of 3 Kubernetes concepts we'll be using
in this post (taken from the excellent Kubernetes documentation):

1. _Pods_ are the smallest deployable units that can be created, scheduled,
and managed. Pods are a colocated group of containers (run on the same node) &
share stuff like network space (e.g. IP address) & disk (via volumes). [Read
more] (https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/pods.md)
2. _Replication controllers_ (RCs) ensure that a    specified number of pod
"replicas" are running at any one time. If there are    too many, it will kill
some. If there are too few, it will start more.    [Reads
more](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/replication-controller.md)
3. _Services_ are an abstraction which defines a logical set of Pods    and a
policy by which to access them. The set of Pods targeted by a Service is
determined by a _Label Selector_. A service is used to access a group of pods
through a consistent IP address without knowing the exact pods, especially
important considering the ephemeral nature of pods. [Read
more](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md)

A _headless service_ is a service that has no IP address (& therefore no
service environment variables, load-balancing or proxying). It is simply used to
track what endpoints (pods) would be part of the service. Perfect for simple
discovery.

I'm going to assume you have a working Kubernetes cluster up & running. If you
don't then you can really easily set one up on any Docker-enabled host via a
script that [Fabric8][fabric8] provides (see
[here](http://fabric8.io/guide/openShiftDocker.html#run-the-start-script) if
you're interested). Note that the script will actually spin up OpenShift3 rather
than vanilla Kubernetes as Fabric8 uses some of the extensions that OpenShift
provides, like builds, deployment pipelines, etc for other things. Everything
below will work on vanilla Kubernetes of course.

You're also going to need to have the
[_DNS cluster add-on_](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/cluster/addons/dns).
Btw, this is another capability that OpenShift provides by default.

For a working (hopefully!) example, I'm going to use [Elasticsearch][elasticsearch] as I'm pretty
familiar with it & it's awesome horizontal scalability lends itself very well to
hopefully explaining this clearly. This is an application that we provide for
one click installation as part of Fabric8 to build up Elasticsearch clusters. To
make this a little more interesting we're actually going to create a cluster of
the 3 different [types of Elasticsearch node](http://www.elastic.co/guide/en/elasticsearch/reference/1.5/modules-node.html):
Master, Data & Client. If you're building a large cluster this is probably what
you would want to do. We're going to make it so that each type can be scaled
individually by resizing the respective replication controller & each node is
going to discover other nodes in the cluster via a headless service.

So to action...

Before we actually create anything, let's prepare our Kubernetes manifests.
We'll send the create requests to the API server at the end - don't jump the gun!

First let's create our replication controllers. All 3 look pretty similar - this
one's for the client nodes:

```json
{
  "apiVersion" : "v1beta1",
  "id" : "elasticsearch-client-rc",
  "kind" : "ReplicationController",
  "labels" : {
    "component" : "elasticsearch",
    "type": "client"
  },
  "desiredState" : {
    "podTemplate" : {
      "desiredState" : {
        "manifest" : {
          "containers" : [ {
            "env" : [
              { "name" : "SERVICE_DNS", "value": "elasticsearch-cluster" },
              { "name": "NODE_DATA",    "value": "false" },
              { "name": "NODE_MASTER",  "value": "false" }
            ],
            "image" : "fabric8/elasticsearch-k8s:1.5.0",
            "imagePullPolicy" : "PullIfNotPresent",
            "name" : "elasticsearch-container",
            "ports" : [
              { "containerPort" : 9200 },
              { "containerPort" : 9300 }
            ]
          } ],
          "id" : "elasticsearchPod",
          "version" : "v1beta1"
        }
      },
      "labels" : {
        "component" : "elasticsearch",
        "type": "client"
      }
    },
    "replicaSelector" : {
      "component" : "elasticsearch",
      "type": "client"
    },
    "replicas" : 1
  }
}
```

Few things of importance here: notice the labels on the pod template:

```json
"labels" : {
  "component" : "elasticsearch",
  "type": "client"
}
```

For the replication controllers for data & master nodes, you will need to update
the `type` in the label - leave the `component` part alone: having a common
superset in the labels for all the node types is what we will use when we create
our headless service.

The environment variables are something that the `fabric8/elasticsearch-k8s`
uses to configure Elasticsearch so you will need to update the `NODE_DATA` &
`NODE_MASTER` environment variables appropriately for the other two replication
controllers for the other node types.

We want all access to the Elasticsearch cluster to go through the client nodes
so let's create a service to do just that:

```json
{
  "id": "elasticsearch",
  "apiVersion": "v1beta1",
  "kind": "Service",
  "containerPort": 9200,
  "port": 9200,
  "selector": {
    "component": "elasticsearch",
    "type": "client"
  }
}
```

Notice that the selector matches the labels of the client nodes replication
controller only.

Finally we create a headless service that we're going to use to discover our
cluster nodes:

```json
{
  "id": "elasticsearch-cluster",
  "apiVersion": "v1beta1",
  "PortalIP": "None",
  "kind": "Service",
  "containerPort": 9300,
  "port": 9300,
  "selector": {
    "component": "elasticsearch"
  }
}
```

Notice the `PortalIP` is set to `None` - that means no IP address will be
allocated to the service. Sadly we still have to specify the `containerPort` &
`port` although these are not used at all.

The final thing to take not of is the `id`: `elasticsearch-cluster`. This is the
DNS name that the service can be discovered under. With a normal service, the
DNS entry that is registered is an A record with the IP address that is
allocated to the service. With a headless service, however, an A record is created
for each service endpoint (pod targeted by the specified selector) for the
service name.

Go ahead & create your resources - create the services first so that cluster
discovery service is available when the nodes first come up.

Once your resources are created & the pods are up, let's check the DNS entries
have been created properly with a quick `dig`. On my setup, this is the result:

```bash
$ dig @localhost elasticsearch-cluster.default.local. +short

172.17.0.13
172.17.0.10
172.17.0.11
```

And for the Elasticsearch client service:

```bash
$ dig @localhost elasticsearch.default.local. +short

172.30.17.196
```

If we use the Elasticsearch client service to check the health of the cluster:

```bash
$ curl http://172.30.17.196:9200/_cluster/health\?pretty

{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "number_of_pending_tasks" : 0
}
```

Yay - it worked! 3 nodes in our cluster discovered without any need for
credentials to be distributed using DNS.

Now play with resizing each of the 3 node types & see what happens - easy cluster
resizing FTW.
