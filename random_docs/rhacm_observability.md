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
> DOCKER_CONFIG_JSON=`oc extract secret/p	ull-secret -n openshift-config --to=-`
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
─> oc -n open-cluster-management-observability get pods
NAME                                                      READY   STATUS    RESTARTS   AGE
endpoint-observability-operator-748767778f-c67mr          1/1     Running   0          5m16s
metrics-collector-deployment-85ff445f49-zrldc             1/1     Running   0          5m16s
observability-alertmanager-0                              4/4     Running   0          5m13s
observability-alertmanager-1                              4/4     Running   0          4m41s
observability-alertmanager-2                              4/4     Running   0          4m6s
observability-grafana-67bdc6c8dc-79j5c                    3/3     Running   0          5m15s
observability-grafana-67bdc6c8dc-k445m                    3/3     Running   0          5m15s
observability-observatorium-api-974646b7c-6jz56           1/1     Running   0          5m15s
observability-observatorium-api-974646b7c-jbtzh           1/1     Running   0          5m15s
observability-observatorium-operator-7ff98677fc-cbp78     1/1     Running   0          5m15s
observability-rbac-query-proxy-5d67cd8c6d-b9zws           2/2     Running   0          5m14s
observability-rbac-query-proxy-5d67cd8c6d-rskw2           2/2     Running   0          5m14s
observability-thanos-compact-0                            1/1     Running   0          5m13s
observability-thanos-query-5c8f78747b-4bv8l               1/1     Running   0          5m14s
observability-thanos-query-5c8f78747b-5pzwd               1/1     Running   0          5m14s
observability-thanos-query-frontend-5f74c8d9b9-7xs5t      1/1     Running   0          5m14s
observability-thanos-query-frontend-5f74c8d9b9-mlkm7      1/1     Running   0          5m14s
observability-thanos-query-frontend-memcached-0           2/2     Running   0          5m10s
observability-thanos-query-frontend-memcached-1           2/2     Running   0          5m6s
observability-thanos-query-frontend-memcached-2           2/2     Running   0          5m4s
observability-thanos-receive-controller-8d45599cd-tdwvb   1/1     Running   0          5m13s
observability-thanos-receive-default-0                    1/1     Running   0          5m11s
observability-thanos-receive-default-1                    1/1     Running   0          4m58s
observability-thanos-receive-default-2                    1/1     Running   0          4m52s
observability-thanos-rule-0                               2/2     Running   0          5m12s
observability-thanos-rule-1                               2/2     Running   0          4m47s
observability-thanos-rule-2                               2/2     Running   0          4m30s
observability-thanos-store-memcached-0                    2/2     Running   0          5m9s
observability-thanos-store-memcached-1                    2/2     Running   0          5m5s
observability-thanos-store-memcached-2                    2/2     Running   0          5m2s
observability-thanos-store-shard-0-0                      1/1     Running   0          5m11s
observability-thanos-store-shard-1-0                      1/1     Running   0          5m11s
observability-thanos-store-shard-2-0                      1/1     Running   0          5m10s
uwl-metrics-collector-deployment-6fb9fc8ff7-pj2c6         1/1     Running   0          5m11s

```

# Creating an Storage bucket with ODF/CEPH

If you already have ODF in your claster (in my case with an internal CEPH cluster), you can create an object bucket through and object bucket claim.

![](assets/observability-2_20240508121803110.png)

The object bucket claim allows you to use on of your available StorageClasses:

![](assets/observability-2_20240508122016315.png)

Once the claim is created, an new object bucket is available with all the S3 endpoint and credentials: