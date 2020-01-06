# Application Logging

In the previous sections, we enabled logging/monitoring capability. Lets take a look at our logging and metrics for the sample application, using features integrated with AKS.

## Enable Azure Insights

In order to enable insights, you need to create a cluster role and role binding to allow the service to interact with the cluster.

From this directory, connected to your cluster, run:

`kubectl apply -f enable_insights.yml`

Click into the `Insights` tab and then on the "Containers" tab in the main pane.

Locate the "Pod" using the pod column, and then locate our workload container. You can see, from this dashboard, the quick view of metrics collected.

## Accessing the Container Logs

Click the item you found, in the previous section, related to your wokload.

On the right, you will see a pane, that includes a dropdown, which says "View in Analytics". Click this dropdown, and change it to "View Container Logs".

