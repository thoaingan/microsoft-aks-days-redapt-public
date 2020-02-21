# Kuberneres Security

## Cluster Surfaces

![k8s-security-framework](images/what-to-secure.png)

>* Service Mesh Control Plane - *Optional*
>* Kubernetes Control Plane - AKS does the heavy lifting
>* Worker Nodes - container runtime (Moby on AKS), CSI, CNI
>* Cluster Infra - Operators, Dashboard, Cloud Provider
>* Workloads - infra workloads, app workloads

## Cloud Native & Kubernetes Security Framework

![k8s-security-framework](images/k8s-security-framework.png)

>* Source Code Scan (@CI)
>* Image Scan (@CI)
>* Cluster Scan (@CD) - Azure DevOps
>* Resource Admission Control (@Admission)
>* K8S Audit Security Monitoring (@Runtime)
>* K8S Workload Runtime Monitor (@Runtime)

## Workload Security

1. [Pod Security](01_pod_security.md)
2. [Securing Our Demo App](02_securing_demo_app.md)