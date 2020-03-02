# Troubleshoot Networking

In Kubernetes there is potential for networking differences between nodes and between namespaces; nodes could have different subnets, firewall rules on the nic, etc. where namespaces can be isolated with NetworkPolicy resources.

In addition to the NetworkPolicy, Kubernetes also runs an internal dns server to resolve internal DNS names. If your pods are configured to use this internal dns for resolution "ClusterFirst", and it goes into a failure or its capacity is exceeded, resolving even external addresses can become an issue.

The biggest challenge debugging an application running in a Kubernetes environment, is that the underlying containers running your workload are typically minified such that cli diagnostic tools, such as curl are not present.

Suppose you have pod that you suspect has a networking/connection error and you are trying to resolve. For example, we see an error that says "Unable to connect to Database - Timeout" The best practice approach would be to isolate various parts of the stack and try to narrow down on the cause.

First, lets identify the namespace and the node name of the afflicted pod. Namespace you should know, but if you dont, its probably in the default namespace. You will need the know the namespace, to describe the pod. If its default `-n` can be omitted.

`kubectl get namespaces`

`kubectl desribe pod POD_NAME -n NAMESPACE`

In this output, you will see `NODE`

```
Node:         YOUR_NODE_NAME/X.X.X.X
```

If you are connecting to a database, as is the case here, it makes sense to build/find a container that contains the mysql command line utility. With the kubectl cli tool, we can quickly run a pod on the same node, with some default configuration, and connect via the `sh` command.

`kubectl run --generator=run-pod/v1 tmp-mysql-shell --rm -i --tty --image mysql:5.7 --overrides='{ "spec": { "nodeSelector": { "kubernetes.io/hostname": "YOUR_NODE_NAME" } } }' -- /bin/sh`

```
If you don't see a command prompt, try pressing enter.

# mysql --host HOST --user USER --password=mypassword
```

For troubleshooting or inspecting, the [Netshoot](https://github.com/nicolaka/netshoot) container and documentation is a good place to start, when looking for a specific tool.
