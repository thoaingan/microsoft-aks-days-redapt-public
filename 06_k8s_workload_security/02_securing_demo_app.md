# Securing Our Demo Application

Lets try to apply one or more of the security controls/knobs to our **favorite beer** app

>Hardening is an iterative process --> at some point, adding a control will break our application

our objective here is to add as many controls without code changes

## Hardening Application Deployment

* Reduce K8S API Server access permissions
* Run the database as non-root user
* Drop container Linux Capabilities
* ReadOnly file system (Immutable Container)

```yaml
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
      # Disable Kubernetes API Server Access - even to the discovery APIs
      automountServiceAccountToken: false
      securityContext:
        fsGroup: 2000      
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
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              add:
              - NET_BIND_SERVICE
              drop:
              - all
            #runAsNonRoot: true
            #runAsUser: 10001
            #runAsGroup: 10001          
            # Enable this while you carefully examine the application functionality
            #readOnlyRootFilesystem: true

          ports:
          - containerPort: 80
            name: http
            protocol: TCP
          
          livenessProbe:
            httpGet:
              path: /
              port: http
            
          readinessProbe:
            httpGet:
              path: /
              port: http
            
          resources:
            limits:
              cpu: 100m
              memory: 128Mi
            requests:
              cpu: 100m
              memory: 128Mi
            
          volumeMounts:
          - name: favorite-beer-config-volume
            mountPath: /etc/config
      volumes:
        - name: favorite-beer-config-volume
          configMap:
            name: favorite-beer
```

## Hardening our Application Database (Redis)

* Reduce K8S API Server access permissions
* Run the database as non-root user
* Drop container Linux Capabilities
* ReadOnly file system (Immutable Container)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: favorite-beer-redis
spec:
  serviceName: favorite-beer-redis
  replicas: 1
  selector:
    matchLabels:
      app: favorite-beer-redis
  template:
    metadata:
      labels:
        app: favorite-beer-redis
    spec:
      # Disable Kubernetes API Server Access - even to the discovery APIs
      automountServiceAccountToken: false
      securityContext:
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 2000       
      containers:
      - name: favorite-beer-redis
        image: redaptcloud/redis:3.2-alpine
        imagePullPolicy: IfNotPresent

        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          runAsNonRoot: true         
          # Enable this while you carefully examine the application functionality
          readOnlyRootFilesystem: true        

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
```
