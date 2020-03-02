# Enable Azure Insights

We enabled Azure Insights earlier in an earlier section, this serves to explain a little more detail.

In the traditional Azure VM landscape, you would install the OMSAgent onto the machines, which allows them to report metrics to the Azure Monitor portal. 

With AKS, the story is similar, but its leveraging the concept of kubernetes DaemonSet to deploy an OMSAgent container to every host. This container and DaemonSet is optimized for grabbing logs, and metrics which get forwarded to Azure.

It gets the metrics from the Kubernetes metrics server API. In order to do this, we need to grant the permissions for the service to access the metrics endpoints.


```
1. In your command line, type `az login` and follow the instructions.
2. Run `az aks install-cli` to install `kubectl` into your local environment. **Note: If you are using Azure Cloud Shell, you can skip this step.**
3. Run `az aks get-credentials -n YourClusterName -g YourClusterRGP` with your flags to set the kubectl context.
```

With that context established, you can run the following commands in realtime, to fetch metrics directly from the kubernetes API. 

```
kubectl top pods
```

```
kubectl top nodes
```

To enable Azure Insights, apply the yaml file in this directory:

```
kubectl apply -f enable_insights.yaml
```

Lets take a look at the contents of this file, and what its doing.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
    name: containerHealth-log-reader
rules:
    - apiGroups: ["", "metrics.k8s.io", "extensions", "apps"]
      resources:
         - "pods/log"
         - "events"
         - "nodes"
         - "pods"
         - "deployments"
         - "replicasets"
      verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
    name: containerHealth-read-logs-global
roleRef:
    kind: ClusterRole
    name: containerHealth-log-reader
    apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: clusterUser
  apiGroup: rbac.authorization.k8s.io
```

As you can see, its establishing a ClusterRole granting clusterUser read access to various resources, including `pods/log` and `pods` in the `metrics.k8s.io` apiGroup.

