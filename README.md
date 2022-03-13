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


# Volumes

Documentation https://kubernetes.io/docs/concepts/storage/volumes/

## Background

**Kubernetes supports many types of volumes**. A Pod can use any number of volume types simultaneously. Ephemeral volume types have a lifetime of a pod, **but persistent volumes exist beyond the lifetime of a pod**. When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. For any kind of volume in a given pod, data is preserved across container restarts.



> Check example directory: `/volume-example`
### `emptyDir`

> Specify to container the volume with `volumeMounts`, and create de volume in Kubernetes with `volumes` using a volume type (i.e. `emptyDir`)

Example deployment with volume:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
      - name: story-container
        image: alejomartinez8/kub-data-demo:2
        # Specify the volume for container
        volumeMounts:
          - mountPath: /app/story # The path in the container
            name: story-volume # Name of volume to connect
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 3000
      # Create the volume in Kubernetes
      volumes:
        - name: story-volume
          emptyDir: {}
```
> Note: An `emptyDir` volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running on that node. As the name says, the `emptyDir` volume is initially empty. All containers in the Pod can read and write the same files in the `emptyDir` volume, though that volume can be mounted at the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the `emptyDir` is deleted permanently.

### `hostPath`
With `emptyDir` we can't share data between replicas, so if we scale the deployment and one pod crashes the next **Pod** can't access to **previous** volume, so we can use another volume type like `hostPath`.

Example:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
      - name: story-container
        image: alejomartinez8/kub-data-demo:2
        # Specify the volume for container
        volumeMounts:
          - mountPath: /app/story # The path in the contianer
            name: story-volume # The name of volume
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 3000
      # Create the volume in Kubernetes
      volumes:
        - name: story-volume
          hostPath:
            path: /data # the directory to share
            type: DirectoryOrCreate # If it doesn't exist, then create it
```
> Note: this only works if the deployment is in the same Node
## Persistent Volumes - `PV`

A `PersistentVolume(PV)` is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but **have a lifecycle independent** of any individual Pod that uses the PV.

> Note: `PV` has types of volumes similar to `Volume`. 

For create a new PV we must create a new resource called `PersistentVolume`, see `host-pv.yaml` file:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem # Filesystem or Block
  accessModes:
    - ReadWriteOnce # read-write volume by a single node
  hostPath:
    path: /data
    type: DirectoryOrCreate
```

## Persistent Volume Claim - `PVC`
A `PersistentVolumeClaim(PVC)` is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources.

To connect with the Persistent Volume, we must create a new resource called `PersistentVolumeClaim`, see file `host-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-pvc
spec:
  volumeName: host-pv # The PV name in host-pv.yaml
  accessModes:
    - ReadWriteOnce
  resources:
   requests:
     storage: 1Gi # The max of Pod capacity
```

Then, we can consume the `PVC` in the `deployment.yaml` file:
```yaml
  volumes:
    - name: story-volume
      persistentVolumeClaim:
        claimName: host-pvc
```