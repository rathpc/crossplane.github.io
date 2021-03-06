---
title: Stacks
toc: true
weight: 6
---

# Defining Infrastructure Environments with Stacks

Though defining your infrastructure as reusable classes and being able to
dynamically provision resources with the same configuration makes for a much
better workflow, it is still a fair amount of work to set up a service catalog,
especially when secure connectivity is required between numerous infrastructure
components. For example, a workload in a remote Kubernetes cluster may want to
communicate with a database that is not exposed over the public internet.
Depending on the provider being used, this can involve VPCs, subnetworks,
security groups and more. Frequently, setting up these networking components is
a repeated task that may take place in multiple regions and accounts.

Crossplane *Stacks* allow you to group a collection of managed resources and
classes into a single unit that can be installed into your cluster as a
[CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).
Let's take a look at installing a minimal stack for commonly used GCP resources.


## Installing and Using the Minimal GCP Stack

[Minimal GCP](https://github.com/crossplane/stack-minimal-gcp) is a Crossplane
stack that includes the following managed resources:

* `Network`
* `Subnetwork`
* `GlobalAddress`
* `Connection`

Because these are managed resources, they will be created immediately (i.e.
static provisioning). The following resource classes will also be created. They
are configured with references to the networking resources above so when we
dynamically provision resources using these classes they will be created in the
provisioned `Network`, `Subnetwork`, etc.

* `GKEClusterClass`
* `CloudSQLInstanceClass`
* `CloudMemorystoreInstanceClass`

The Minimal GCP stack will also create a `Provider` resource for us, so we can
go ahead and delete the one we have been using:

```
kubectl delete provider.gcp.crossplane.io gcp-provider
```

Now, similar to how we installed the GCP provider at the beginning, we can
install a Crossplane stack with a `ClusterStackInstall`. Create the a file named
`stack-min-gcp.yaml` with the following content:

```yaml
apiVersion: stacks.crossplane.io/v1alpha1
kind: ClusterStackInstall
metadata:
  name: minimal-gcp
  namespace: crossplane-system
spec:
  package: crossplane/stack-minimal-gcp:v0.3.0
```

Then create it in your cluster:

```
kubectl apply -f stack-min-gcp.yaml
```

Creating this resource does not actually cause any of the resources listed above
to be created. Instead it creates a
[CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
in your cluster that allows you to repeatedly create instance of the environment
defined by the stack. To create an instance of the Minimal GCP stack, create a
file named `my-min-gcp.yaml` with the following content:

```yaml
apiVersion: gcp.resourcepacks.crossplane.io/v1alpha1
kind: MinimalGCP
metadata:
  name: my-min-gcp
spec:
  region: us-west2
  projectID: <your-project-id> # replace with the project ID you created your Provider with earlier
  credentialsSecretRef:
    name: gcp-account-creds
    namespace: crossplane-system
    key: credentials
```

Then create the instance:

```
kubectl apply -f my-min-gcp.yaml
```

Crossplane will immediately create the managed resources and classes that are
part of the Minimal GCP stack.

Now that we have general set of infrastructure and classes defined in our
cluster, it is time to deploy some applications. In the [previous section](workload.md), we bundled resources into a `KubernetesApplication` and
created it in the control cluster. This is useful for applications that are
deployed infrequently and are not widely distributed, but can be a burden for
someone who is not familiar with the architecture to manage. In the [next section](app.md) we will explore how Crossplane *Applications* make it possible
to package and distribute your configuration, including managed services that an
application consumes.

## Clean Up

The resources created in this section will be used in the next one as well, but
if you do not intend to go through the next section and would like to clean up
the resources created in this section, run the following command:

```
kubectl delete -f my-min-gcp.yaml
```
