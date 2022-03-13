# Volumes
## Background
**Kubernetes supports many types of volumes**. A Pod can use any number of volume types simultaneously. Ephemeral volume types have a lifetime of a pod, **but persistent volumes exist beyond the lifetime of a pod**. When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. For any kind of volume in a given pod, data is preserved across container restarts.

> NOTE: The volumes are independent of the containers but not of the Pods, so if the Pod is destroyed the data could be lost (lifetime of Pod). The volumes are in parallel with container inside the Pod.

For more info visit https://kubernetes.io/docs/concepts/storage/volumes/. Let see some types:
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