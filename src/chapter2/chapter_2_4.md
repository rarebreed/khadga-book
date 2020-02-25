# Kubernetes objects and config

There are 3 parts to kubernetes that you must get to know and become comfortable with:

- Kubernetes architecture
- Kubernetes objects
- Kubernetes config

## Kubernetes architecture

For some additional documentation, the kubernetes site is a good resource.

For our purposes, we need to understand the big picture of all the pieces of kubernetes and what it
is good for.  To start with, lets throw out some terminology to exercise your brain:

- Master
- Cluster
- Nodes
- Pods
- Objects
- Namespace

### Master

The master is a software component installed on cluster that tells what the nodes to do.  It manages
the state of the cluster and the nodes, and tries to get the desired state in your config, to the
actual state of the nodes and objects in your cluster

### Clusters

A cluster is a collection of nodes.  This is generally what you will work on as a whole.  When you
setup minikube on your local machine, it is actually setting up a cluster (of one node) on your
machine.

### Nodes

Nodes are an abstraction of compute resources.  It could be an actual bare metal machine, or a VM
running inside a physical machine.  Pods will run on nodes.  You can have one or more nodes per
cluster.

### Pods

Pods are the smallest unit in kubernetes that actually runs containers.  However, it is possible to
have more than one container running in a pod, but usually that is only if the containers have a
tight coupling to one another.  Generally speaking, you have a one container to one pod
relationship.

### Objects

In kubernetes, pretty much everything is an `Object`.  There are many kinds of Objects:

- Deployments
- Pod
- Service
- Ingress
- ManagedCertificates
- Secrets

To name a few.  You can see a list of them by running `kubectl api-resources`.  These various
objects can either be created imperatively by running commands through kubectl, or declaratively
through the use of config files.  We will concentrate on the latter, but sometimes it is necessary
to run some commands imperatively (for example, to delete objects).

### Namespace

A namespace is a way to isolate objects.  Objects (eg, Deployments, Pods, Services, etc) can be run
in a namespace and isolated from other namespaces.  It is sometimes useful to create a namespace for
testing or development purposes.

## Why kubernetes?

One might ask, why use kubernetes at all?  Afterall, you could use google app engine to host a
website, or an AWS EC2 or ECS instance.  An ECS instance is just a container afterall.

The thing with kubernetes is that is an orchestrator for containers.  Okay, if you are like me, you
saw some variant of that definition a million times, and wondered "You keep on saying that word
orchestration.  I do not think it means what you think it means".

What is meant by _orchestration_?  To get a better appreciation for that, we need to understand the
problem that kubernetes attempts to solve.

Containers can be ephemeral and short lived, either on accident or by design.  Moreover, some
containers might have a dependency on other containers.  Imagine for example one service that needs
a database to get data, or perhaps you are running a microservice architecture where different
services are running on separate containers rather than one monolithic server.  The rise of the
microservice architecture went hand in hand with the popularity of kubernetes.  Why?

Since microservices now run separately, what do you do if one of them crashes?  Or what if you need
to roll out a bug fix or new version?  Do you stop the world while you push out new changes, or can
you just deploy a new service?  This is effectively what kubernetes is good at, and why it is said
to _orchestrate_ containers.  Kubernetes basically says something like "You told me that you want X
number of container A, Y number of container B, and Z number of container C, with load balancing to
go to the appropriate container based on url paths.  I'll make sure that our set (cluster) of
containers is setup that way.  And if any of the containers goes down, I'll bring them back up for
you".

## Contexts: Minikube vs GKE

We want a way to develop and test our code before pushing to a real cloud provider.  When you first
install gcloud and kubectl, it's a good idea to set up a local kubernetes cluster.  The minikube
project is one way to do this.

Once you install minikube, you then need to understand what a context is.  A context is the
environment that kubectl will be running commands against.  In essence, kubectl is a CLI tool that
sends REST commands to an API server running on a kubernetes master.  The master will receive these
commands and try to get to the desired state either imperatively (most kubectl commands) or
declaratively (through the `apply` command with config files as arguments).  The question then
becomes, which master is kubectl targeting, the minikube master, or a GKE master?

To answer this question, you can run this command:

```bash
kubectl config get-contexts
```

The one with the asterix next to it is the one kubectl is pointing to.  You can change this manually
with the command:

```bash
kubectl config use-context <name>
```

### Namespaces

A related concern is what namespace you are working with in kubectl.  By default, there is a
namespace named default (who woulda thunk it?).  Sometimes however you want to create an isolated
namespace.  To do this, you can create a namespace:

```
# Create a namespace
kubectl create namespace localdev

# Set the namespace on a context
kubectl config set-context --namespace=localdev minikube
```
