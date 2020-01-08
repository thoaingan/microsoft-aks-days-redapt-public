# Application Logging

## Accessing Logs using Kubectl
First fetch the pod that you are interested in, and then you can isolate the logs. If there are multiple containers in the pod, you can also pass in the `-c` argument.

`kubectl get pods`

`kubectl logs -f favorite-beer-bd8897bff-fg8d6`

## Accessing the Container Logs via Azure Portal
In the previous sections, we enabled Azure logging/monitoring capability. Lets take a look at our logging and metrics for the sample application, using features integrated with AKS.

### Enable Azure Insights

In order to enable insights, you need to create a cluster role and role binding to allow the service to interact with the cluster.

From this directory, connected to your cluster, run:

`kubectl apply -f enable_insights.yml`

Click into the `Insights` tab and then on the "Containers" tab in the main pane.

Locate the "Pod" using the pod column, and then locate our workload container. You can see, from this dashboard, the quick view of metrics collected.

### Accessing the Container Logs

Click the item you found, in the previous section, related to your wokload.

On the right, you will see a pane, that includes a dropdown, which says "View in Analytics". Click this dropdown, and change it to "View Container Logs".

In this generated query, you can change the following section:


`| where ContainerName ~= 'XXXXXXXXXXXXXXXXXXXXXXX/favorite-beer'`


to


`| where ContainerName contains '/favorite-beer'`


To see all of the containers that match this name, in a single stream.