# Logging

When a container is running, its generally following a process which is writing to STDOUT. Any logs that are written to STDOUT are captured on the node, where the container is running.

In AKS, the OMSAgent pods are mounting special Kubernetes host directories, so they have read access to all of the logs on the host, where each OMSAgent pod is running. 

Lets take a quick look.

`kubectl get pods -n kube-system`

`kubectl get pods -n kube-system omsagent-XXXXX -o yaml`

Take a look at the volumes section, which can tell us a bit about how this agent gets its information.

```
  volumes:
  - hostPath:
      path: /
      type: ""
    name: host-root
  - hostPath:
      path: /var/run
      type: ""
    name: docker-sock
  - hostPath:
      path: /etc/hostname
      type: ""
    name: container-hostname
  - hostPath:
      path: /var/log
      type: ""
    name: host-log
  - hostPath:
      path: /var/lib/docker/containers
      type: ""
    name: containerlog-path
  - hostPath:
      path: /etc/kubernetes
      type: ""
    name: azure-json-path
  - name: omsagent-secret
    secret:
      defaultMode: 420
      secretName: omsagent-secret
  - configMap:
      defaultMode: 420
      name: container-azm-ms-agentconfig
      optional: true
    name: settings-vol-config
  - name: omsagent-token-9r9xz
    secret:
      defaultMode: 420
      secretName: omsagent-token-9r9xz
```

## Additional Forwarding

Even with Azure Insights enabled, your organization may want to forward these logs to another centralized location. A typical pattern is to deploy fluentd as a DaemonSet, and pass in configuration that includes your destination as a target.

There are various kubernetes plugins for fluentd, to pre-process the data prior to forwarding. If you are interested more in the approach, check out the helm chart for `fluentd`.

https://github.com/helm/charts/tree/master/stable/fluentd

If you want a simple solution that can be deployed within the cluster, checkout the `fluentd-elasticsearch`chart.

https://github.com/helm/charts/tree/master/stable/fluentd-elasticsearch

Kibana can also be deployed, to act as the UI for elasticsearch, this is all effictively known as `EFK` or `ELK` (Logstash in place of fluentd) stack.

## Best Practices

When developing a microservice strategy, or migrating applications to containers, it's best to forward all logs to stdout stream. It will be more painful to implement a strategy that has multiple log files, that need to be collected and indexed part from the logs of the containerized process.

If you have containers that have custom log files, its best to put those log files into a volumeMount that can be shared with other containers in the pod. With that approach, you can use another conatiner to `tail` the log file to its stdout stream, which can be collected as well.