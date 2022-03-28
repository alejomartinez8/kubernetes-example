# Kubernetes Notes

Course: https://www.udemy.com/course/docker-kubernetes-the-practical-guide/
## Overview
Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

See more: https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/

## Installation
https://kubernetes.io/docs/tasks/tools/


* Install `kubectl`, allows you to run commands against Kubernetes clusters.
* Install `minikube`, tool that lets you run Kubernetes locally.

## Minikube
minikube is local Kubernetes, focusing on making it easy to learn and develop for Kubernetes.

All you need is Docker (or similarly compatible) container or a Virtual Machine environment.

See more: https://minikube.sigs.k8s.io/docs/start/

To initialize `minikube` you need a Virtual Machine with Kubernetes to emulate a Cluster, so you can use some of this alternatives: 
- `minikube start --driver=docker` (require Docker)
- `minikube start --driver=virtualbox` (require virtualbox)
- `minikube start` (using default with docker)

To check if it's working
```
minikube status
```
You can see a `localhost` web app to monitor your `minikube` cluster and see deployments, services, pods and more:
```
minikube dashboard
```
Now you can connect to minikube cluster with `kubectl` cli.
# Imperative use (Kubectl)
Kubernetes provides a command line tool for communicating with a Kubernetes cluster's control plane, using the Kubernetes API.
> For the next commands see example directory: `/action-example`

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

To expose a `deployment` on a port, create a service using:
```
kubectl expose deployment [deployment-name] --type=LoadBalancer --port=8080 
```
You can see the services using:
```
kubectl get services
```
> **Note**: to expose on a public IP (in your host-minikube) you cant use:
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

Using a YAML or JSON file, you can configure you cluster.


For deployments (deployment.yaml):

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
> Documentation: https://kubernetes.io/docs/concepts/workloads/controllers/deployment

For services (services.yaml):

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
> Documentation:  https://kubernetes.io/docs/concepts/services-networking/service/

Then, sse the `kubectl apply` option to execute declarative files:
```
kubectl apply -f=deployment.yaml -f=service.yaml
```

## Update
If you need to update a config file reflected on the cluster, just executes again:
```
kubectl apply -f=filename.yaml
```

## Delete
Delete a deployement or service using:
```
kubectl delete -f=filename.yaml
```

# Volumes
## Background
**Kubernetes supports many types of volumes**. A Pod can use any number of volume types simultaneously. Ephemeral volume types have a lifetime of a pod, **but persistent volumes exist beyond the lifetime of a pod**. When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. For any kind of volume in a given pod, data is preserved across container restarts.

> NOTE: The volumes are independent of the containers but not of the Pods, so if the Pod is destroyed the data could be lost (lifetime of Pod). The volumes are in parallel with container inside the Pod.

See more (https://kubernetes.io/docs/concepts/storage/volumes/)

Let see some types:
### `emptyDir`

> Connect a container with a volume using `volumeMounts:`, and create some volumes in Kubernetes with `volumes:` tag, adding a volume type inside (i.e. `emptyDir`)

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
        # Connect the volume
        volumeMounts:
          - mountPath: /app/story # The path in the container
            name: story-volume # Name of volume to connect
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 3000
      # Create the volumes in the Pod
      volumes:
        - name: story-volume
          emptyDir: {}
```
> NOTE: An `emptyDir` volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running on that node. As the name says, the `emptyDir` volume is initially empty. All containers in the Pod can read and write the same files in the `emptyDir` volume, though that volume can be mounted at the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the `emptyDir` is deleted permanently.

### `hostPath`
A hostPath volume mounts a file or directory from the host node's filesystem into your Pod. This is not something that most Pods will need, but it offers a powerful escape hatch for some applications.

> NOTE: With `emptyDir` we can't share data between replicas (Pods), so if we scale the deployment and one pod crashes the next **Pod** can't access to **previous** volume, so we can use another volume type like `hostPath`.

Example:
```yaml
      volumes:
        - name: story-volume
          hostPath:
            path: /data # the directory to share
            type: DirectoryOrCreate # If /data doesn't exist, then create it
```
> Note: this only works if the deployment is in the same Node
## Persistent Volumes - `PV`

A `PersistentVolume(PV)` is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but **have a lifecycle independent** of any individual Pod that uses the PV.


For create a new PV we must create a new resource called `PersistentVolume`, see `host-pv.yaml` file like:
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
A `PersistentVolumeClaim(PVC)` is a request for storage by an user. It's similar to a Pod. Pods consume node resources and PVCs consume PV resources (Remember that PV are in the same level like Nodes).

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

## Environment vars
You can set/user ENV var inside the deployments:
```yaml
  containers:
    - name: story-container
    image: alejomartinez8/kub-data-demo:3
    env:
      - name: STORY_FOLDER
      value: 'story'
```

## Config Map
A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

```yaml
    env:
      - name: STORY_FOLDER
        valueFrom:
          configMapKeyRef:
            name: data-store-env
            key: folder
```

# Network

Networking is a central part of Kubernetes, but it can be challenging to understand exactly how it is expected to work. There are 4 distinct networking problems to address:

1. Highly-coupled container-to-container communications: this is solved by Pods and localhost communications.
2. Pod-to-Pod communications.
3. Pod-to-Service communications.
4. External-to-Service communications.

See more: https://kubernetes.io/docs/concepts/cluster-administration/networking

> In the following examples see the `/network-example` folder, where `/user-api` and `/tasks-api` consume `/auth-api`, and `/frontend` consumes `/tasks-api`.

### Container-to-container

You must create the containers in the same POD (in the same deployment file), and it allows you to use `localhost`:
```yaml
 # ... users-deployment.yaml
      containers:
        - name: users
          image: alejomartinez8/kub-demo-users:latest
          env:
            - name: AUTH_ADDRESS
              value: localhost
        - name: auth
          image: alejomartinez8/kub-demo-auth:latest
```

### Pod-to-Pod
You must create the containers in differents PODs (in different deployment files):

```yaml
  # ... auth-deployment.yaml
    containers:
      - name: auth
        image: alejomartinez8/kub-demo-auth:latest
```
You can specify the type of service using `ClusteIP` if need to set it not public:
```yaml
# ... auth-service.yaml
  spec:
    selector:
      app: auth
    type: ClusterIP
    ports:
      - protocol: TCP
        port: 80
        targetPort: 80
```

To consume this service, here we have several options:

1. The services create a default `env` var using the format `[SERVICE_NAME]_SERVICE_HOST`, so for example with `auth-service` could be like  `AUTH_SERVICE_SERVICE_HOST`, in this case is not necessary to set any `env` var on deployments files.

So the address is in the application could be like: `http://${process.env.AUTH_SERVICE_SERVICE_HOST}`.

2. Using `IP address` of services:

```yaml
  # ... users-deployment.yaml
      containers:
        - name: users
          image: alejomartinez8/kub-demo-users:latest
          env:
            - name: AUTH_ADDRESS
              value: "10.99.104.252"
```

To get the IP address of services use:
```
kubectl get services
```

3. Using `{name-service}.{name-space}` dynamically:

```yaml
  # ... users-deployment.yaml
      containers:
        - name: users
          image: alejomartinez8/kub-demo-users:latest
          env:
            - name: AUTH_ADDRESS
              value: "auth-service.default"
```
To get the name spaces use:
```
kubectl get namespaces
```
So the address in the application could be like: `http://${process.env.AUTH_ADDRESS}`

## Example Multiple Deployments

Setup `auth-api`:
```yaml
  # ... auth-deployment.yaml
    containers:
      - name: auth
        image: alejomartinez8/kub-demo-auth:latest
```

Setup `users-api` consuming `users-api` to get token:

```yaml
# ... users-deployment.taml
    containers:
      - name: users
        image: alejomartinez8/kub-demo-users:latest
        env:
          - name: AUTH_ADDRESS
            value: "auth-service.default"
```

Setup `tasks-api` consuming `auth-api`, to pass the token:
```yaml
  # ... tasks-deployment.yaml
    containers:
      - name: tasks
        image: alejomartinez8/kub-demo-tasks:latest
        env:
          - name: AUTH_ADDRESS
            value: "auth-service.default"
          - name: TASKS_FOLDER
            value: tasks
```

Setup `frontend`:
```yaml
 # ... frontend-deployment.yaml
    containers:
      - name: frontend
        image: alejomartinez8/kub-demo-frontend:latest
```
**Alternative**: here we use a reverse proxy and setup the service `tasks-api` in `conf/ngix.conf` using the format `{name-service}.{name-space}`:

```
 location /api/ {
    proxy_pass http://tasks-service.default:8000/;
  }
```
So in the `frontend` application we can use the url `/api/tasks` with the `Bearer token`.