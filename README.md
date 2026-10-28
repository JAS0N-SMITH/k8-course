# k8-course

https://www.boot.dev/lessons/402f9141-74ef-4933-abb1-5be6404b4b83

## Quick Restart Cheat Sheet

```bash
# 1. Start the cluster (CORS flag needed for boot.dev)
minikube start --extra-config="apiserver.cors-allowed-origins=['http://boot.dev']"
minikube status                      # expect: Running / Configured

# 2. Re-apply all manifests (safe to re-run)
kubectl apply -f api-configmap.yaml -f crawler-configmap.yaml
kubectl apply -f web-deployment.yaml -f api-deployment.yaml -f crawler-deployment.yaml
kubectl apply -f web-configmap.yaml
kubectl apply -f web-service.yaml -f api-service.yaml -f crawler-service.yaml

# 2b. Gateway API (Envoy Gateway must be installed; only needed again after `minikube delete`)
# helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.2.0 -n envoy-gateway-system --create-namespace
kubectl apply -f app-gatewayclass.yaml -f app-gateway.yaml
kubectl apply -f web-httproute.yaml -f api-httproute.yaml

# 3. Check everything is up
kubectl get deployments
kubectl get pods                     # wait for READY 1/1
kubectl get services
kubectl get configmaps
kubectl get gatewayclass,gateway,httproute   # Gateway should show PROGRAMMED=True

# 3b. Reach the gateway from your Mac (separate terminal, keep running)
minikube tunnel                      # gives the Gateway an external IP
# Map hostnames to the Gateway address in /etc/hosts:
#   <GATEWAY-IP>  synchat.internal synchatapi.internal
kubectl get gateway app-gateway      # ADDRESS column = <GATEWAY-IP>

# 4. Access / debug
kubectl port-forward PODNAME 8080:8080
kubectl logs PODNAME
kubectl describe pod PODNAME
kubectl proxy                        # http://127.0.0.1:8001/api/v1/namespaces/default/pods

# 5. Boot.dev CLI
bootdev run                          # run tests
bootdev run -s                       # run tests and submit

# Shutdown
minikube stop                        # keep cluster state
minikube delete                      # wipe everything
```

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

## API Service

We've deployed one service, and we've deployed multiple instances of it. Time to deploy a second service!

This service doesn't serve a webpage! It's a JSON API. It's the backend for our chat application. By deploying the API and configuring the front-end to talk to it, we'll have a functional chat application!

### Create a Deployment Configuration

Let's write a deployment from scratch.

Feel free to reference the k8s docs here as you go for examples of the proper structure.

1. [ ] Create a new file called `api-deployment.yaml`.
2. [ ] Add the `apiVersion` and `kind` fields. The apiVersion is apps/v1 and, since this is a deployment, the kind is Deployment.
3. [ ] Add a `metadata/name` field, and let's name our deployment synergychat-api for consistency.
4. [ ] Add a `metadata/labels/app` field, and also set it to synergychat-api. This will be used to select the pods that this deployment manages.
5. [ ] Add a `spec/replicas` field and let's set it to 1. We can always scale up to more pods later.
6. [ ] Add a `spec/selector/matchLabels/app` field and set it to synergychat-api. This should match the label we set in step 4.
7. [ ] Add a `spec/template/metadata/labels/app` field and set it to synergychat-api. Again, this should match the label we set in step 4. Labels are important because they're how Kubernetes knows which pods belong to which deployments.
8. [ ] Add a `spec/template/spec/containers` field. This actually contains a list of containers that will be deployed:
       _Note: A hyphen is how you denote a list item in YAML_
   1. [ ] Set the `name` of the container to synergychat-api.
   2. [ ] Set the `image` to bootdotdev/synergychat-api:latest. This tells k8s where to download the Docker image from.

I don't want to give you the YAML because I want you to type it out, but here's the equivalent JSON:

```json
{
  "apiVersion": "apps/v1",
  "kind": "Deployment",
  "metadata": {
    "name": "synergychat-api",
    "labels": {
      "app": "synergychat-api"
    }
  },
  "spec": {
    "replicas": 1,
    "selector": {
      "matchLabels": {
        "app": "synergychat-api"
      }
    },
    "template": {
      "metadata": {
        "labels": {
          "app": "synergychat-api"
        }
      },
      "spec": {
        "containers": [
          {
            "name": "synergychat-api",
            "image": "bootdotdev/synergychat-api:latest"
          }
        ]
      }
    }
  }
}
```

### Create the Deployment

Run the following command to create the deployment:

```bash
kubectl apply -f api-deployment.yaml
```

Next, take a look at all the pods you have running now. You should see pods for the web service and a pod for the api service.

However, you might notice that the api pod isn't in a "ready" state. In fact, it should be stuck in a "CrashLoopBackOff" status. Oh no! We've created a thrashing pod!

Run:

```bash
kubectl get pods
```

```bash
kubectl proxy
```

Run and submit the CLI tests.

```bash
bootdev run
bootdev run -s
```

## Thrashing Pods

One of the most common problems you'll run into when working with Kubernetes is Pods that keep crashing and restarting. This is called "thrashing" and it's usually caused by one of a few things:

- The application recently had a bug introduced in the latest image version
- The application is misconfigured and can't start properly
- A dependency of the application is misconfigured and the application can't start properly
- The application is trying to use too much memory and is being killed by Kubernetes

### What Is “CrashLoopBackOff”?

When a pod's status is CrashLoopBackOff, that means the container keeps crashing (the program is exiting with a non-zero exit code, like 1).

Because Kubernetes is all about building self-healing systems, it will automatically restart the container. However, each time it tries to restart the container, if it crashes again, it will wait longer and longer in between restarts. That's why it's called a "backoff".

To fix a thrashing pod, you need to find out why it's crashing. We'll do that in a later lesson.

## Config Maps

There are several ways to manage environment variables in Kubernetes. One of the most common ways is to use ConfigMaps. ConfigMaps allow us to decouple our configurations from our container images, which is important because we don't want to have to rebuild our images every time we want to change a configuration value.

In a Dockerfile we can set environment variables like this:

```dockerfile
ENV PORT=3000
```

The trouble is, that means that everyone using that image will have to use port `3000`. It also means that if we want to change the port, we have to rebuild the image.

First, let's take a closer look at our crashing pod and try to figure out why it's crashing.

```bash
kubectl get pods
```

Copy the pod name and then get the logs:

```bash
kubectl logs <pod-name>
```

You should see that a specific environment variable is missing! Let's fix that.

Create a new file. Let's call it api-configmap.yaml. Add the following YAML to it:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: synergychat-api-configmap
data:
  API_PORT: "8080"
```

Next, apply the config map:

```bash
kubectl apply -f api-configmap.yaml
```

Now, we haven't yet connected the config map to our pod, so it should still be crashing. However, for now, let's just validate that the config map was created successfully:

```bash
kubectl get configmaps
```

Run:

```bash
kubectl proxy
```

Run and submit the CLI tests.

```bash
bootdev run 9eb61c75-e1c4-4e1c-9d57-984575974e0d
bootdev run -s 9eb61c75-e1c4-4e1c-9d57-984575974e0d
```

## Applying the Config Map

Now that we have a config map, we need to connect it to our deployment.
Assignment

Open up your api-deployment.yaml file. We're going to add a few things to it. Under the containers section, add the following to the first (and only) entry:

```yaml
env:
  - name: API_PORT
    valueFrom:
      configMapKeyRef:
        name: synergychat-api-configmap
        key: API_PORT
```

This tells Kubernetes to set the `API_PORT` environment variable to the value of the `API_PORT` key in the `synergychat-api-configmap` config map. Reference the official docs if you're confused about the structure of the yaml.

Next, apply the deployment. Hopefully, you remember the command for this by now.

```bash
kubectl apply -f api-deployment.yaml
```

Once it's applied, you should be able to take a look at the pods and see that a new API pod has been deployed and isn't crashing!

Let's forward the API pod's 8080 port to our local machine so we can test it out.

```bash
kubectl port-forward <pod-name> 8080:8080
```

Make sure it returns a 404 response when you hit the root:

```bash
curl http://localhost:8080
```

Run and submit the CLI tests.

```bash
bootdev run
bootdev run -s
```

## Config Maps Are Insecure

ConfigMaps are a great way to manage innocent environment variables in Kubernetes. Things like:

- Ports
- URLs of other services
- Feature flags
- Settings that change between environments, like DEBUG mode

However, they are not cryptographically secure. **ConfigMaps aren't encrypted**, and they can be accessed by anyone with access to the cluster.

If you need to store sensitive information, you should use **Kubernetes Secrets** or a third-party solution.
