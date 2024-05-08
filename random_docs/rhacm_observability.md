# RHACM Observality


Requirements:
 * RHACM and MCE operators installed
 * Storage system:
   * For this example an Storage bucket S3 compatible. At the bottom there is an small section to create an S3 compatbile endpoing using ODF. 
   * Other supported cloud storage systems [available](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.3/html-single/observability/index#prerequisites-observability)

## Install the operator

To install the operator you can mainly follow [this instructions](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.3/html-single/observability/index#enabling-observability).

Create a NS and copy there a `pull-secret`:

```bash
> oc create namespace open-cluster-management-observability
namespace/open-cluster-management-observability created
> DOCKER_CONFIG_JSON=`oc extract secret/pull-secret -n openshift-config --to=-`
# .dockerconfigjson
> oc create secret generic multiclusterhub-operator-pull-secret \
    -n open-cluster-management-observability \
    --from-literal=.dockerconfigjson="$DOCKER_CONFIG_JSON" \
    --type=kubernetes.io/dockerconfigjson
secret/multiclusterhub-operator-pull-secret created

```

Create the object storage credentials (some example at the bottom of the document):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: thanos-object-storage
  namespace: open-cluster-management-observability
type: Opaque
stringData:
  thanos.yaml: |
    type: s3
    config:
      bucket:  observability-bucket-e196c599-dfe6-413d-b23d-e671902b3ae7
      endpoint: rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc
      insecure: true
      access_key: 7I62........53VRG
      secret_key: NCQV........oRYX0VMp8H
```


Create the MultiClusterObservability:
```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  observabilityAddonSpec: {}
  storageConfig:
    storageClass: ocs-storagecluster-ceph-rbd
    metricObjectStorage:
      name: thanos-object-storage
      key: thanos.yaml

```

Use the `.spec.storageClass` to use a proper StorageClass to be used by the Observability pods.

If everything was oka:

```bash

```

# Creating an Storage bucket with ODF/CEPH

If you already have ODF in your claster (in my case with an internal CEPH cluster), you can create an object bucket through and object bucket claim.

![](assets/observability-2_20240508121803110.png)

The object bucket claim allows you to use on of your available StorageClasses:

![](assets/observability-2_20240508122016315.png)

Once the claim is created, an new object bucket is available with all the S3 endpoint and credentials: