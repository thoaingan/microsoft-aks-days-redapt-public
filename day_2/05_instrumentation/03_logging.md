# Logging

When a container is running, its generally following a process which is writing to STDOUT. Any logs that are written to STDOUT are captured on the node, where the container is running.

In AKS, the OMSAgent pods are mounting special Kubernetes host directories, so they have read access to all of the logs on the host, where each OMSAgent pod is running. 

## Additional Forwarding

Even with Azure Insights enabled, your organization may want to forward these logs to another centralized location. A typical pattern is to deploy fluentd as a DaemonSet, and pass in configuration that includes your destination as a target.

There are various kubernetes plugins for fluentd, to pre-process the data prior to forwarding. If you are interested more in the approach, check out the helm chart for `fluentd`.

https://github.com/helm/charts/tree/master/stable/fluentd

If you want a simple solution that can be deployed within the cluster, checkout the `fluentd-elasticsearch`chart.

https://github.com/helm/charts/tree/master/stable/fluentd-elasticsearch

Kibana can also be deployed, to act as the UI for elasticsearch, this is all effictively known as `EFK` or `ELK` (Logstash in place of fluentd) stack.

## Best Practices

