# Running kubernetes

Here, we will show how to set up minikube and use kubectl

## Installing minikube

You can follow the directions to [install minikube here][-minikube]

### Using minikube

```bash
minikube start --vm-driver=kvm2
minikube status
```

## Creating deployment files

TODO: Go over all the deployment files we need


## Installing kubectl

You can follow the directions here to [install kubectl][-kubectl]

### Setting up kubectl

To get kubectl to use the minikube k8s enviroment by creating a new namespace, run this command:

```bash
kubectl config set-context --namespace=localdev minikube
```

### Kubectl commands

This command will expose an already running pod named khadga, and attach a LoadBalancer service
with a name of khadga-service

```bash
kubectl expose deployment khadga --type=LoadBalancer --name=khadga-service 
```

This command uses the kompose file, and will run the get command on everything in the kompose

```bash
kubectl get -k . 
```

Will display all the services running in the kubernetes environment (for the default namespace)

```bash
kubectl get service
kubectl get deployment
kubectl get pod
kubectl get pod -o wide 
```

Shows how to delete various kubernetes objects like Deployments or Services

```bash
kubectl delete deployment khadga
kubectl delete service khadga-service 
```

Uses the kompose file to apply all the config files.  Looks for a kompose file

```bash
kubectl apply -k . 
```

Creates a new namespace.  Namespaces isolate the kubernetes environment

```bash
kubectl create namespace localdev 
```
```bash
kubectl config get-contexts 
```

## Setting up gcloud

You can follow the directions here to [install the gcloud SDK][-gcloud-sdk].

Once the gcloud SDK is installed, you will also need to initialize it.

```
gcloud init
```

### kubectl commands

Allow gcloud to use docker

```bash
gcloud auth configure-docker
```

To set up auth for logging in, run this command.

```bash
gcloud auth login
```

To set the project in google cloud to work with, run this command


```bash
gcloud config set project khadga-dev
```

To show a list of images in GCR run this command, where the argument to repository is the URL for
the GCR repo (eg gcr.io, eu.gcr.io, etc), a slash, and then the name of the project

```bash
gcloud container images list --repository=gcr.io/khadga-dev
```

To show the tags of an image use this command

```bash
gcloud container images list-tags gcr.io/khadga-dev/khadga
```


```bash
docker-credential-gcloud configure-docker
```

To get credentials 

```bash
gcloud container clusters get-credentials standard-cluster-1 --zone us-central1-a --project khadga-dev
```

## General workflow for testing and deployment

- docker build your container images
- tag your images
- push the images to gcr.io
- Update the config file for the updated image tag


### For testing

- create a localdev namespace and use it
- Create the secret so that localdev knows how to use it
- patch the serviceaccount to use the imagePullSecrets key


For the first step, you need to create a new namespace if you haven't already:

```bash
kubectl create namespace localdev
```

If you have run this step before, you don't need to again.  You can go to step 3.

You can either append --namespace=localdev to all kubectl commands, or you can make it your default,
where the final argument is the name of your `context`.  To view your contexts:

```bash
kubectl config get-contexts
```

Then to set your namespace to the context you want:

```bash
kubectl config set-context --namespace=localdev minikube
```

Next, you can set up a new [secret key by following there directions here][-gcr-secret]

Then you need to patch your serviceaccount to use the key that you downloaded.

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gcr-json-key"}]}'
```

[-gcloud-sdk]: https://cloud.google.com/sdk/install
[-minikube]: https://kubernetes.io/docs/tasks/tools/install-minikube/
[-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[-gcr-secret]: https://blog.container-solutions.com/using-google-container-registry-with-kubernetes