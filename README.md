# Kubernetes Notes

Course: https://www.udemy.com/course/docker-kubernetes-the-practical-guide/
## Overview
Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/

## Installation
https://kubernetes.io/docs/tasks/tools/


* Install `kubectl`, allows you to run commands against Kubernetes clusters.
* Install `minikube`, tool that lets you run Kubernetes locally.

## Run
>
> minikube
> 
To initialize `minikube` you need a Virtual Machine with Kubernetes to emulate a Cluster, so you can use some of this alternatives: 
- `minikube start --driver=docker` (require Docker)
- `minikube start --driver=virtualbox` (require virtualbox)
- `minikube start` (using default with docker)

To check if it's working
```
minikube status
```
You can see a localhost web app to monitor your **minikube** cluster and see deployments, services, pods and more:
```
minikube dashboard
```
Now you can connect to minikube cluster with `kubectl` cli.
# Imperative use
> See example directory: `/action-example`

## Deployments

To create a deployment from an image:
```
kubectl create deployment [deployment-name] --image=[image name]
```
> image must be from image registry (i.e. Docker hub)

To list all deployments:
```
kubectl get deployments
```

To list all pods:
```
kubectl get pods
```
## Services

To expose the deployment on a port, create a service using:
```
kubectl expose deployment [deployment-name] --type=LoadBalancer --port=8080 
```
You can see the services using:
```
kubectl get services
```
> Note: to expose on a public IP (in your host, minikube) you cant use:
> ```
> minikube service [deployment-service]
> ```
> But in macOs I had some troubles with minikube, so a workaround could be:
> ```
> minikube tunner
> ```
> See: https://minikube.sigs.k8s.io/docs/handbook/accessing/

## Scale
To scale your app (increase numbers of pods/containers) you can use:
```
kubectl scale deployment/[deployment-name] --replicas=3
```

## Update (set)

To update a container inside a pod:
```
kubectl set image deployment/[deployment-name] [container-name]=[image name:tag]
```
> Note: is important to have a different tag to update successfully

## Rollout

To know the status of a deployment:
```
kubectl rollout status deployment/[deployment-name]
```
To rollout an update (for a possible failure):
```
kubectl rollout undo deployment/[deployment-name]
```
To see history:
```
kubectl rollout history deployment/[deployment-name] --revision=3
```

## Delete
To delete a deployment or service:
```
kubectl delete [deployment-name]
kubectl delete [service-name]
```

# Dlecarative use

> See example directory: `/action-example`

Using a YAML or JSON file, you can configure you cluster.


For deployments (deployment.yaml):
> https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: second-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: second-app
      tier: backend
  template:
    metadata:
      labels:
        app: second-app
        tier: backend
    spec:
      containers:
        - name: second-node
          image: alejomartinez8/kub-[deployment-name]
        # - name: example
        #   image: example

```
For services (services.yaml):
> https://kubernetes.io/docs/concepts/services-networking/service/

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: second-app
  ports:
    - protocol: 'TCP'
      port: 8080
      targetPort: 8080
  type: LoadBalancer

```
Use the `apply` option to execute declarative files:
```
kubectl apply -f=deployment.yaml -f=service.yaml
```
> Note: if you need to update a config file reflected on the cluster, just executes again `kubectl apply -f filename.yaml`, and it changes everythinig (i.e. updating the image of container)