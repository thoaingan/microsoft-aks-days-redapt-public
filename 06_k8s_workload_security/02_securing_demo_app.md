# Securing Our Demo Application

Lets try to apply one or more of the security controls/knobs to our **favorite beer**

>Hardening is an iterative process --> at some point, adding a control will break our application

our objective here is to add as many controls without code changes

## Hardening Application Deployment

* Reduce K8S API Server access permissions
* Run the database as non-root user
* Drop container Linux Capabilities
* ReadOnly file system (Immutable Container)

## Hardening our Application Database (Redis)

* Reduce K8S API Server access permissions
* Run the database as non-root user
* Drop container Linux Capabilities
* ReadOnly file system (Immutable Container)
