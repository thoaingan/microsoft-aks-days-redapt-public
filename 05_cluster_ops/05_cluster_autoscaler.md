# Cluster Auto Scaler

In general, the cluster autoscaler is responsible for scaling up and down the underlying node pool, so that additional pods can be included and infrastructure remains utilized. In an ideal setup, you can focus on running the right number of pods, with the correct resources, and let the cluster autoscaler manage the size of your node pool accordingly.

The cluster auto scaler is generally implemented by the IaaS provider, for cloud deployments. There are autoscalers that connect to on-premise vm provisioning systems, to scale out that way as well.

## AKS

For AKS, they have special instructions for deploying this to your cluster:
https://docs.microsoft.com/en-us/azure/aks/autoscaler

https://docs.microsoft.com/en-us/azure/aks/use-multiple-node-pools#scale-a-node-pool

Once deployed, you can create seperate node pools and set independent scale rules.

```
az aks nodepool update \
  --resource-group jm-rgp \
  --cluster-name jm-cluster \
  --name agentpool \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --node-count 3 \
  --max-count 5
```