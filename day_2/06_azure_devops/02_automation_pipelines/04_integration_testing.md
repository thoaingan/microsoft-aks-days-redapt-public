# Integration Testing MicroServices

With Kubernetes on IaaS such as AKS, it becomes reliably feasible to bring up entire stacks of micro-services in a temporary namespace, executing an integration or load test suite against the dynamically provisioned load balancers, and have it report status back to the team.

As mentioned in the previous sections, if you are performing stress tests, it makes sense to perform them in a seperate cluster that is an analog to production. 

## Using Kubernetes to run Tests

Kubernetes has the concept of a Job, which consists of a running container. This Job could contain the source code (including the test suite) and instead of running the web server, the command/entrypoint for the job could be the test suite run commands. Unit tests can be run this way fairly easily. 

With integration tests, you may need to configure them to discover the dependent microservices, either via env vars or mounted configmap/secrets. Because the job is running in the cluster, ideally within the same namespace as the services themselves, it will be able to resolve any services using internal service names.

For many application frameworks, the configuration of the running workload and the configuration for the test run, is very similar, which means you can leverage the existing deployment pod spec, for the job pod spec.

These jobs can be applied to the cluster, during a pipeline run, whether its Jenkins or ADO. In that pipeline, youd apply the job, tail the pods logs, and then check the Job status upon completion, to determine next steps.