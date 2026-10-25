# k8-course

https://www.boot.dev/lessons/402f9141-74ef-4933-abb1-5be6404b4b83

## Setup:

### K8

```bash
brew install kubectl
kubectl version --client
```

### Minikube

```bash
brew install minikube
minikube version
minikube start --extra-config="apiserver.cors-allowed-origins=['http://boot.dev']"
minikube stop
minikube delete
```

### BootDev

```bash
bootdev run -s
```

The CLI will run these checks:

```
    minikube status
        Expecting stdout to contain all of:
            Running
            Configured
        Expecting stdout to contain none of:
            Stopped
```

### Deploy an Image

```bash
kubectl create deployment synergychat-web --image=docker.io/bootdotdev/synergychat-web:latest
kubectl get deployments
```

### Viewing Deployments

To make sure the deployment was successful, run:

```bash
kubectl get deployments
```

### Viewing Pods

To view the pods created by the deployment, run:

```bash
kubectl get pods
```

You should see something like this:

```bash
NAME                                   READY   STATUS    RESTARTS   AGE
synergychat-web-679cbcc6cd-cq6vx       1/1     Running   0          20m
```

### Port Forwarding

To forward the port of your pod to your local machine, run:

```bash
kubectl port-forward PODNAME 8080:8080
```

_Be sure to replace PODNAME with your pod's name. In my case, it was synergychat-web-679cbcc6cd-cq6vx._

### Accessing the Webpage

Next, **open your browser and navigate** to http://localhost:8080. You should see a webpage titled "SynergyChat"!

_Keep in mind, that we haven't configured all the resources the page needs yet, so the forms won't work, but the page should load._

**Run and submit** the CLI tests.

```bash
bootdev run
bootdev run -s
```

### Edit the Deployment

To make changes to your deployment, you can **edit** it using:

```bash
kubectl edit deployment synergychat-web
```

_This is the configuration of your deployment. Under the `spec` section, you should see the replicas field set to `1`._

**Change it to `2`, save the file, and close the editor.**

```yaml
spec:
  ...
  replicas: 2
  ...
```

**Run `kubectl get pods` again:**

```bash
kubectl get pods
```

_You should see two pods now!_

### Print the logs

The logs are what the container is printing to `stdout` of your older pod.

```bash
kubectl logs PODNAME
```

**Kill that older pod:**

```bash
kubectl delete pod PODNAME
```

_This might take several seconds to complete:_

**Get a list of your running pods again:**

```bash
kubectl get pods
```

_The deleted pod is replaced by a new one._

## Unique IP Addresses

Every Pod in a Kubernetes cluster has a unique internal-to-k8s IP address. By giving each Pod a unique IP, Kubernetes simplifies communication and service discovery within the cluster. Pods within the same Node or across different Nodes can easily communicate.

All the resources inside a k8s cluster are **virtualized**. So, the IP address of a Pod is not the same as the IP address of the Node it's running on. It's a virtual IP address that is **only accessible from within the cluster**.

### Assignment

**Run this command** to get a "wide" output of your pods:

```bash
kubectl get pods -o wide
```

It gives a few more columns of information, including the IP address of each Pod. _Notice that each Pod has a unique IP address!_

Next, **run**:

```bash
kubectl proxy
```

_This will start a proxy server on your local machine, probably on `127.0.0.1:8001`. Assuming that's the host, navigate to http://127.0.0.1:8001/api/v1/namespaces/default/pods in your browser. You should see a big nasty JSON blob that describes the pods that you have running._

**Run and submit** the CLI tests.

```bash
bootdev run
bootdev run -s
```

_The tests will make sure that the proxy server is accessible and that it's returning information about the current Pods in your cluster._

## Deployments

A Deployment provides **declarative updates** for **Pods** and **ReplicaSets**.

You describe your desired state in a Deployment, and the Deployment Controller's job is to make the current state match the desired state. You declare your hopes and dreams, and it's Kubernetes' job to make them come true.

### Why Deleting a Pod Doesn't Feel Like a Deletion

Remember when we had you delete a pod, only to see that a new pod was created in its place? It's kinda like chopping heads off of a hydra.

That's because the desired state described in our Deployment says we want 2 pods running at all times. When we delete one, the Deployment Controller sees that the current state doesn't match the desired state, so it creates a new pod to make them match again.

In other words, **you don't manage individual pods directly** when using Deployments. You manage the desired state, and Kubernetes takes care of the rest. This is a key concept in Kubernetes: you declare what you want, and the system works to make it true.

**Take a look at the YAML file for your current deployment in the CLI:**

```bash
kubectl get deployment synergychat-web -o yaml
```

**Edit the deployment and change the number of replicas from 2 to 10:**

```bash
kubectl edit deployment synergychat-web
```

**Make sure you've got 10 pods running:**

```bash
kubectl get pod
```

**Keep using `kubectl get pod` to check on your pods until all 10 show "1/1" under "READY". Once they are, run:**

```bash
kubectl proxy
```

**Run and submit the CLI tests.**

```bash
bootdev run
bootdev run -s
```

## Replica Sets

A [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) maintains a stable set of replica Pods running at any given time. It's the thing that makes sure that the number of Pods you want running is the same as the number of Pods that are actually running.

You might be thinking, "I thought that's what a Deployment does." Well...yes.

A Deployment is a higher-level abstraction that manages the ReplicaSets for you. You can think of a Deployment as a wrapper around a ReplicaSet. Here's the rub:

You will probably never use ReplicaSets directly. I just need to mention what they are because you'll hear the term thrown around, and might even see them referenced in logs and such.

### Look at Your Replica Sets

Let's take a look at the ReplicaSets that are running in your cluster:

```bash
kubectl get replicasets
```

_Just like with pods, notice that we never directly created the replica set. We created a deployment, and the deployment created the replica set._

## YAML Config

Kubernetes resources are primarily configured using YAML files. We've used the kubectl edit command to edit resources in the cluster on-demand, but let's inspect our deployment's YAML file a bit more closely.

First, download a copy of your deployment's YAML file and save it in your current directory:

```bash
kubectl get deployment synergychat-web -o yaml > web-deployment.yaml
```

Then open it in your text editor. There are **5 top-level fields** in the file:

- `apiVersion`: apps/v1 - Specifies the version of the Kubernetes API you're using to create the object (e.g., apps/v1 for Deployments).
- `kind`: Deployment - Specifies the type of object you're configuring
- `metadata` - Metadata about the deployment, like when it was created, its name, and its ID
- `spec` - The desired state of the deployment. Most impactful edits, like how many replicas you want, will be made here.
- `status` - The current state of the deployment. You won't edit this directly, it's just for you to see what's going on with your deployment.

Inside your editor, **change the number of replicas to 3 and save the file**.
_Notice that you're just editing a file on your machine! It won't yet have any effect on the deployment in your cluster._

To apply the changes, **run**:

```bash
kubectl apply -f web-deployment.yaml
```

You should get a warning that lets you know that you're missing the **last-applied-configuration annotation**. That's okay! we got that warning because we created this deployment the quick and dirty way, by using kubectl create deployment instead of creating a YAML file and using `kubectl apply -f`.

However, because we've now updated it with `kubectl apply`, the annotation is now there, and we won't get the warning again.

**Download the YAML file again** and take a look at it. You should see the annotation now.

**Apply the configuration a second time**, you won't get the warning. Save this YAML file in a git repo for this course! We'll be making more configuration files. Kubernetes is an "infra-as-code" tool, so it's important to keep your configuration files in a git repo.

Finally, **start the proxy server** (if it's not already running):

```bash
kubectl proxy
```

**Run and submit the CLI tests:**

```bash
bootdev run
bootdev run -s
```
