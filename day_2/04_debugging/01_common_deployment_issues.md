# Common Deployment Issues

When deploying to Kubernetes it requires a bit of a different mindset for troubleshooting issues. As the vast majority of issues are going to be occuring, within a container. Ideally, that container's code has been unit tested and integration tested. In the ideal world, we can assume that any issue we are seeing is not related to the code, but potentially to the configuration or the environment the code is running in.

We will walk through some common steps to check the environment, before we escalate to a dev team for a code check/change. 

## Centralized Logging

Generally the first place to check would be a centralized logging system, which we will talk about in the next section. This will generally have the most specific error available to you. Typically, the application logs from a single failing pod would give some indicator of where the issue might lie. If the issue was a code problem, generally verbose logging can help determine that.

For a configuration error, it may complain about an invalid host, or credential/auth error. 

## Configuration Error

In the case of an issue that looks like a configuration error, we would want to check that the image, entrypoint/command, env vars, and volume mounts are defined correctly. You can do this by running `kubectl get pod POD_NAME` or `kubectl describe pod POD_NAME` on the pod in question, which will show any env vars and file mounts that are set at runtime.

If it references configmaps or secrets, you can `kubectl get configmap CONFIGMAP_NAME` or `kubectl get secret SECRET_NAME` with `-o yaml` to see the full spec.

Note: Secrets are base64 encoded, so to see real values would need to decode the data values.

In the event, that a secret or configmap does not exist, or a volume had issues mounting, that would appear in the pod events at the bottom of the `describe` output as an error message. If there are any readiness or liveness probes defined, you can also see any related failures in the pod events.

## Connect to the Container

If you have verified that the configuration is good, and the logs are indicating that something within the container is wrong, it makes sense to connect to the container and do some manual inspection.

You can effectively ssh into a container by using the `kubectl exec` command with the `sh` or `bash` entrypoint, for linux containers.

`kubectl exec -it POD_NAME -c CONTAINER_NAME -- sh`

From within this shell context, you can check various files, and manually run commands to probe/test the container for issues. This process assumes that the container is running and the process its following is not crashing/exiting early, but there is some other issue that needs checked.
