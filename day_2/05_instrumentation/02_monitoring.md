# Monitoring

## Cluster Resources

From the cluster perspective, each container is consuming resources, on the individual nodes in the node pool. It's up to the operator, or the cluster autoscaling mechanism to ensure that underlying nodes exist or can be created as pods become unschedulable. 

As a developer it would be your role to help operators understand what is the expected resource consumption. You can determine this by using the `top` commands we mentioned in the last page, during an average load test.

With these numbers, you can define resource requests that Kubernetes can use for scheduling workloads appropriately, given the nodes available and current workloads.

## APM

Chances are your favorite APM tool, which may integrate nicely with your code, has a Kubernetes implementation available. In general, most solutions provide yaml for a DaemonSet that you can deploy to your cluster, with your license key. With that pods running on every host, the workloads would be configured to forward metrics to the hosts (that its currently running on) APM Server port.

- DataDog
- SignalFX
- Stackify
- etc.