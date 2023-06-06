# Quickstart

This guid walks thorugh the process to install the DataStax Astra Provider.

To install and use this provider:

* Install Upbound Universal Crossplane (UXP) into your Kubernetes cluster.
* Install the Provider and apply a ProviderConfig.
* Create a managed resource in AWS with Kubernetes.

## Prerequisites

This quickstart requires:
* a Kubernetes cluster with permissions to create pods and secrets
* a host with `kubectl` installed and configured to access the Kubernetes
  cluster
* An AstraDB account with database
* AstraDB Credetials to access AstraDB DevOps API.

A step-by-step walk through of the required commands and descriptions on what the commands do.
Note: all commands use the current kubeconfig context and configuration and assuming Crossplane already installed.

## Install Provider 

Install the  provider into the Kubernetes cluster with a Kubernetes configuration file.
```
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-datastax-astra
spec:
  package: xpkg.upbound.io/zipstack/provider-datastax-astra:v0.0.1
```

Verify the provider installed with ```kubectl describe providers.pkg.crossplane.io``` and ```kubectl get providers```. This kubectl describe providers output is from an installed provider.

```
+ kubectl describe providers.pkg.crossplane.io provider-datastax-astra
Name:         provider-datastax-astra
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  pkg.crossplane.io/v1
Kind:         Provider
# Output truncated
Status:
  Conditions:
    Last Transition Time:  2023-05-31T08:17:39Z
    Reason:                HealthyPackageRevision
    Status:                True
    Type:                  Healthy
    Last Transition Time:  2023-05-31T08:17:25Z
    Reason:                ActivePackageRevision
    Status:                True
    Type:                  Installed
  Current Identifier:      xpkg.upbound.io/zipstack/provider-datastax-astra:v0.0.1
  Current Revision:        provider-datastax-astra-6d578b39ce3f
Events:
  Type    Reason                  Age                    From                                 Message
  ----    ------                  ----                   ----                                 -------
  Normal  InstallPackageRevision  25m (x304 over 5d20h)  packages/provider.pkg.crossplane.io  Successfully installed package revision
```

The INSTALLED value should be True. It may take up to 5 minutes for HEALTHY to report true.

```
+ kubectl get providers
NAME                      INSTALLED   HEALTHY   PACKAGE                                                         AGE
provider-aws              True        True      xpkg.upbound.io/upbound/provider-aws:v0.35.0                    9d
provider-datastax-astra   True        True      xpkg.upbound.io/zipstack/provider-datastax-astra:v0.0.1         5d22h
provider-k8s              True        True      xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.9.0   9d
```
If there are issues downloading and installing the provider the INSTALLED field is empty. Use kubectl describe providers for more information.

## Create a Kubernetes secret for AstraDB

The provider requires credentials to create and manage AstraDB resources.

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: astra-creds
  namespace: crossplane-system
type: Opaque
stringData:
  credentials: |
    {
      "token": "y0ur-t0k3n"
    }
EOF    
```
Replace token with Access token created in AstraDB with DB admin role.

## Create a ProviderConfig

Create a ProviderConfig Kubernetes configuration file to attach the AstraDB credentials to the installed official provider.

```
cat <<EOF | kubectl apply -f -
apiVersion: datastax-astra.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      name: astra-creds
      namespace: crossplane-system
      key: credentials
EOF      
```

The spec.secretRef describes the parameters of the secret to use.

namespace is the Kubernetes namespace the secret is in.
name is the name of the Kubernetes secret object.
key is the Data field from kubectl describe secret.

Apply this configuration with kubectl apply -f.

Verify the ProviderConfig with kubectl describe providerconfigs.

```
+ kubectl describe providerconfigs.datastax-astra.upbound.io
Name:         default
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  datastax-astra.upbound.io/v1beta1
Kind:         ProviderConfig
# Output truncated
Spec:
  Credentials:
    Secret Ref:
      Key:        credentials
      Name:       astra-credentials
      Namespace:  crossplane-system
    Source:       Secret
```

Note: the ProviderConfig install fails and Kubernetes returns an error if the Provider isn't installed.

## Create a managed resource

Create a managed resource to verify the provider is functioning.

This example creates a keyspace, which requires a globally unique name.
NOTE: Keyspace name should contain alphanumeric characters only, spaces and special characters are not allowed (use underscore instead of space)


```
cat <<EOF | kubectl apply -f -
apiVersion: keyspace.datastax-astra.upbound.io/v1alpha1
kind: Keyspace
metadata:
  name: test-keyspace
spec:
  forProvider:
    databaseId: <DATABASE ID>
    name: <my_keyspace>
  providerConfigRef:
    name: default
EOF    
```

Get the DATABASE ID from AstraDB console and replace it in above resource file.

Use ```kubectl get keyspaces.keyspace.datastax-astra.upbound.io``` to verify bucket creation.

```
+ kubectl get keyspaces.keyspace.datastax-astra.upbound.io
NAME      READY   SYNCED   EXTERNAL-NAME                                               AGE
tester1   True    True     7b860481-b0d9-4a56-83da-d7544910884a/keyspace/my_keyspace   64s
```
Upbound created the keyspace when the values READY and SYNCED are True. This may take up to 5 minutes.

If the ``READY`` or `SYNCED` are blank or `False` use `kubectl describe` to understand why.

## Delete the managed resource

Remove the managed resource by using `kubectl delete -f` with the same Bucket object file. Verify removal of the bucket with `kubectl get keyspaces.keyspace.datastax-astra.upbound.io`

```
+ kubectl delete keyspaces.keyspace.datastax-astra.upbound.io tester1
keyspace.keyspace.datastax-astra.upbound.io "tester1" deleted

+ kubectl get keyspaces.keyspace.datastax-astra.upbound.io
No resources found
```