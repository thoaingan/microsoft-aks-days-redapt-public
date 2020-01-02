# Application Example

Start by taking a look at the application in the `voting` directory of this folder.

## Setting Up Redis

Since redis is a dependency of our app, we should deploy it first. The recommended way is via `StatefulSet`

We can map our understanding of the `docker-compose.yml` we developed in our local environment, to our understanding of this primitive in Kubernetes.

```
$ cat docker-compose.alt.yml

version: '3.4'

services:
  voting:
    build:
      context: .
      dockerfile: voting/Dockerfile
    depends_on:
      - redis
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SERVICE_SETTINGS_PATH=servicesettings_alt.json
    ports: 
      - '8000:80'
  redis:
    image: "redis:3.2-alpine"
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
volumes:
  redis_data:
    driver: "local"
```


Our deployment for the `voting`/`redis` container:
```
$ cat favorite-beer-redis-statefulset.yml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: favorite-beer-redis
spec:
  serviceName: "favorite-beer-redis"
  replicas: 1
  selector:
    matchLabels:
      app: favorite-beer-redis
  template:
    metadata:
      labels:
        app: favorite-beer-redis
    spec:
      containers:
      - name: favorite-beer-redis
        image: redaptcloud/redis:3.2-alpine
        command: ["redis-server"]
        args: ["--appendonly", "yes"]
        ports:
        - containerPort: 6379
          name: redis
```


Notable Differences:
1. The deployment metadata is included, and controller information are also included. 
2. The containers section is populated from the `redis` section, and command is split into command and args.
3. `ports` becomes more defined, since we prefer to name ports in Kubernetes, and we have more options available.
4. We have not included any information about volumes, getting mounted to the container, yet.

### Persistent Volumes

While this implementation could be used, if we are concerned about keeping this data, for the life of our cluster, we will need to create a PersistentVolume.

##### MiniKube

**NOTE** Local storage is not compatible with Dynamic Provisioning of volumes, so we take a different approach for minikube.

On minikube, we can experiment with this by creating a host path volume, that we will use for our volume claim.

```
$ minkube ssh
minikube> mkdir /data/redis-data
```

```
$ cat favorite-beer-redis-pv.yml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: favorite-beer-redis-pv
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  capacity:
    storage: 1Gi
  hostPath:
    path: /data/redis-data/
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: favorite-beer-redis-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

$ kubectl apply -f favorite-beer-redis-pv.yml
```

```
$ cat favorite-beer-redis-statefulset.yml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: favorite-beer-redis
spec:
  serviceName: "favorite-beer-redis"
  replicas: 1
  selector:
    matchLabels:
      app: favorite-beer-redis
  template:
    metadata:
      labels:
        app: favorite-beer-redis
    spec:
      containers:
      - name: favorite-beer-redis
        image: redaptcloud/redis:3.2-alpine
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumes:
    - name: redis-date
      persistentVolumeClaim:
       claimName: favorite-beer-redis-pvc
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - favorite-beer-redis

$ kubectl apply -f favorite-beer-redis-statefulset.yml
```

Notable Differences:

1. A PersistentVolume and a PVC were created, specifying a `hostPath` that we want to mount our data to on our minikube node. The claim will consume the volume, when it is created.
2. In our set, we reference the claim in the volumes section, and then specify the path in our container to mount it to, in the `volumeMounts` section.
2. Anti-Affinity - Added to prevent collisions with local hostpaths for minikube users, but for AKS, it can help satisfy availability, requirements if the pods are not colocated. The caveat being, you will need to have at least as many nodes, as pods you desire in the stateful set. In minikube, this is a single node cluster.

##### AKS

In AKS, we can store our data in a Premium Managed Disk really easily, with Dynamic Provisioning.

```
$ cat favorite-beer-redis-statefulset.yml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: favorite-beer-redis
spec:
  serviceName: "favorite-beer-redis"
  replicas: 1
  selector:
    matchLabels:
      app: favorite-beer-redis
  template:
    metadata:
      labels:
        app: favorite-beer-redis
    spec:
      containers:
      - name: favorite-beer-redis
        image: redaptcloud/redis:3.2-alpine
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      storageClassName: managed-premium
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi

$ kubectl apply -f favorite-beer-redis-statefulset.yml
```

Notable Differences:

1. In our set, we have specified a `volumeClaimTemplate` that will tell the AKS cloud provider to create a Managed Disk and attach it to designated nodes for each pod that is created. We reference the claim template name and mount it to our containers data path, in the `volumeMounts` section.

### Exposing this StatefulSet

Since this is a service that recieve traffic internally or otherwise over the net, we will need to expose an endpoint. This is where the Kubernetes `Service` comes into play. As you may remember, when we were discussing the primitives there are 3 types of services each more complex than the last, building up to `LoadBalancer`.

We can define a service for our stateful set, and in this case we will choose `type: ClusterIP`. Applying this resource to our cluster, will create an internal (to the kubernetes cluster) IP that can be reached via the service name. We don't have any need to access our redis instance, outside of services running on Kubernetes, so no need to go further.

```
$ cat favorite-beer-redis-service.yml

apiVersion: v1
kind: Service
metadata:
  name: favorite-beer-redis
spec:
  selector:
    app: favorite-beer-redis
  ports:
  - name: redis
    port: 6379
    protocol: TCP
    targetPort: 6379
  type: ClusterIP

$ kubectl apply -f favorite-beer-redis-service.yml
```

## Setting Up Our Web App (Favorite Beer)

The second microservice in our stack is a stateless service. We want to be able to scale this independently, in other words launch as many pods as necessary, to handle our load. The best Kubernetes primitive to choose for this scenario is a `Deployment`, which will manage `ReplicationController`s for each rollout, which will manage the number of pods we have running.

We can map our understanding of the `docker-compose.yml` we developed in our local environment, to our understanding of this primitive in Kubernetes.

```
$ cat docker-compose.alt.yml

version: '3.4'

services:
  voting:
    build:
      context: .
      dockerfile: voting/Dockerfile
    depends_on:
      - redis
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - SERVICE_SETTINGS_PATH=servicesettings_alt.json
    ports: 
      - '8000:80'
  redis:
    image: "redis:3.2-alpine"
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
volumes:
  redis_data:
    driver: "local"
```

Our deployment for the `voting`/`favorite-beer` app container.
```
$ cat favorite-beer-deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: favorite-beer
  labels:
    app: favorite-beer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: favorite-beer
  template:
    metadata:
      labels:
        app: favorite-beer
    spec:
      containers:
      - name: favorite-beer
        image: redaptcloud/favorite-beer:v1
        env:
        - name: REDIS_HOST
          value: "favorite-beer-redis"
        - name: REDIS_PORT
          value: "6379"
        - name: SERVICE_SETTINGS_PATH
          value: "servicesettings_alt.json"
        ports:
        - name: http
          containerPort: 80
```

Notable Differences:

1. The deployment metadata is included, and controller information are also included. 
2. The containers section is populated from the `voting` section, with `image` replacing `build`.
3. `depends_on` is a docker-compose construct, that is excluded, but considered during deployment strategy. It tells us we will need to deploy our redis endpoint first, when we actually apply this to Kubernetes.
4. `environment` becomes `env` under the containers spec, and takes a different input format.
5. We have defined `REDIS_HOST` as `favorite-beer-redis`, as this is the `Service` name for our redis deployment, in the last section.
6. `ports` becomes more defined, since we prefer to name ports in Kubernetes, and we have more options available.

### Configuration

That implementation is great, if we want to use one of the `servicesettings.json` files that is packaged within the application. However, we exposed these things to be able to update our settings at deploy time, so we can create a deployment that uses configmaps and volume mounts with custom configuration.

Consider if we define a configmap that includes a `servicesettings.json` with a few additional beverages, that looks like this:

```
$ cat favorite-beer-configmap.yml

apiVersion: v1
kind: ConfigMap
metadata:
  name: favorite-beer
data:
  servicesettings.json: |
    {
      "ServiceSettings": {
        "Beers": ["Red Hook", "Pyramid", "Great Divide", "New Belgium"]
      }
    }

$ kubectl apply -f favorite-beer-configmap.yml
```

We can then add to and reference this in our previous deployment:
```
$ cat favorite-beer-deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: favorite-beer
  labels:
    app: favorite-beer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: favorite-beer
  template:
    metadata:
      labels:
        app: favorite-beer
    spec:
      containers:
      - name: favorite-beer
        image: redaptcloud/favorite-beer:v1
        env:
        - name: REDIS_HOST
          value: "favorite-beer-redis"
        - name: REDIS_PORT
          value: "6379"
        - name: SERVICE_SETTINGS_PATH
          value: "/etc/config/servicesettings.json"
        ports:
        - name: http
          containerPort: 80
        volumeMounts:
        - name: favorite-beer-config-volume
          mountPath: /etc/config
      volumes:
        - name: favorite-beer-config-volume
          configMap:
            name: favorite-beer

$ kubectl apply -f favorite-beer-deployment.yml
```

What is happening here:
1. We define a volume in the volumes section called `favorite-beer-config-volume` which is referencing our `configMap` by name `favorite-beer`.
2. We are creating a mount to the container in the `volumeMounts` section, referencing the volume we defined, and the path within the container we want to mount to.
3. After these steps are performed, our container should have files that relate back to the confimap data keys, with the contents being their value. In this case `ls /etc/config/` would show `servicesettings.json`, so we update the `SERVICE_SETTINGS_PATH` environment variable pointing to this location.

Later when this app is deployed to Kubernetes, we will deploy `favorite-beer-configmap.yml` before or at the same time as `favorite-beer-deployment.yml`.

### Exposing this Deployment

Since this is a service that recieves traffic internally or otherwise over the net, we will need to expose an endpoint. This is where the Kubernetes `Service` comes into play. As you may remember, when we were discussing the primitives there are 3 types of services each more complex than the last, building up to `LoadBalancer`.

We can define a service for our deployment. If you already have a cluster running in AKS, you can speciy `type: LoadBalancer`, however on minikube you should use `NodePort`. This process executes iptables rules on every node, in the cluster for the NodePort defined. If not defined, it will choose a random available port, within a specific range. 

On AKS, with `LoadBalancer` this will additionally configure an Azure Load Balancer attached to every node in your cluster, with traffic routed to this NodePort. You could then setup your DNS servers to point to that Load Balancer, for a complete setup.

```
$ cat favorite-beer-service.yml

apiVersion: v1
kind: Service
metadata:
  name: favorite-beer
spec:
  selector:
    app: favorite-beer
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 31650
  type: LoadBalancer

$ kubectl apply -f favorite-beer-service.yml

$ kubectl get svc
NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
favorite-beer         LoadBalancer   10.0.47.83     13.64.173.174   80:31650/TCP   26m
```

Note the LoadBalancer IP (13.64.173.174, in this example) Azure assigned to the Azure Load Balancer it attached to your AKS nodes. Put whichever IP Azure assigned for _your_ Load Balancer into your browser and you should see the "Favorite Beer" app working.

