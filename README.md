# spire-openshift-federated-identities

Demonstrate how [SPIFFE](https://spiffe.io) and [SPIRE](https://spiffe.io/docs/latest/spire-about), when deployed on [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift) as part of [Red Hat Zero Trust Workload Identity Manager (ZTWIM)](https://www.redhat.com/en/technologies/cloud-computing/openshift/zero-trust-workload-identity-manager), can make use of Federated Identities on several Public Cloud Platforms.

This repository consists of the following components:

1. Deployment of Zero Trust Workload Identity Manager (ZTWIM) on OpenShift
2. Resources to support federating SPIFFE identities from OpenShift to [Microsoft Azure](https://azure.microsoft.com), [Amazon Web Services](https://aws.amazon.com) and [Google Cloud](https://cloud.google.com) platforms.

## Installation of Red Hat Zero Trust Workload Identity Manager

Utilize the following steps to deploy Red Hat Zero Trust Workload Identity Manager in an OpenShift cluster.

First, authenticate to the OpenShift cluster as an elevated user and set a few environment variables related to the cluster:

```shell
export OPENSHIFT_CLUSTER_APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{ .spec.baseDomain }')
export ZTWIM_OPERATOR_CHANNEL=$(oc get packagemanifests openshift-zero-trust-workload-identity-manager -o jsonpath='{ .status.defaultChannel }')
```

Next, render the Kustomization file for the ZTWIM Operator from the root of the repository

```shell
envsubst < templates/ztwim-operator-kustomization.template > install/ztwim/operator/kustomization.yaml
```

Deploy the ZTWIM Operator

```shell
oc apply -k install/ztwim/operator
```

Wait util the ZTWIM Custom Resource Definitions have been established

```shell
until oc wait crd/spireservers.operator.openshift.io --for condition=established --timeout 120s >/dev/null 2>&1 ; do sleep 1 ; done
```

Now, render the Kustomization file for the ZTWIM resources

```shell
envsubst < templates/ztwim-instance-kustomization.template > install/ztwim/instance/kustomization.yaml
```

Deploy the ZTWIM Custom Resources

```shell
oc apply -k install/ztwim/instance
```

Wait until ZTWIM has been fully deployed

```shell
until oc get zerotrustworkloadidentitymanager cluster -o jsonpath='{ .status.conditions[?(@.type=="Ready")].status }'=="True" >/dev/null 2>&1 ; do sleep 10; done;
```

## Workload Deployment

A sample workload is available in the [install/workload](install/workload) directory to demonstrate how to utilize federating identities across cloud providers with SPIFFE and SPIRE.

Apply the workload to the cluster from the root of the repository:

```shell
oc apply -f install/workload/
```

Wait until the application has been deployed:

```shell
oc wait -n spire-federated-identities --for=condition=Available deployment/workload-app --timeout=600s
```

With the application deployed, begin exploring how to utilize federated identities in each of the Public Cloud providers.
