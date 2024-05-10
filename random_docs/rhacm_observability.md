
# RHACM Cluster monitoring

Before going with the real topic of this document, lets understand briefly how Cluster Monitoring works in any cluster with Openshift.

![](assets/rhacm_observability_20240510112717083.png)

The main component in the Cluster Monitoring is Prometheous. Prometheous will scrape metrics of any service (defined by ServiceMonitoring) that exports a `/metrics` interface returning a set of different metrics. It has it own time series database, but it is not intended to persist or to keep data for long period. ServiceMonitoring resources points Prometheous to where (And when) scrape metrics.

But how to scale, query and persist all the data? Thanos is in charge of that. You can directly query Prometheous with PromQL, but remember Prometheous is not intended to persist data. Instead of that, Prometheous sends the data to a Thanos Querier, that centralizes the data from different Prometheous. We have the default Openshift Prometheous (scraping node-exporter, kubelet, etc), but we could have another Prometheous for user's workloads that will export their own metrics. The data is sent from Prometheous to Thanos using the Remote-Write protocol. 

When we have enabled RHACM Multicluster Observability, now we not only have the local Prometheous, we will have the Cluster Monitoring Prometheous from other Openshift spoke clusters. 

![](assets/rhacm_observability_20240510113434516.png)

RHACM Multicluster Observability provides (at Hub level ) a centralized Thanos Querier, and Thanos Receiver, that will collect all the metrics.

Therefore, every Cluster Montoring (including the local one in the hub cluster) gathers metrics with their local Prometheous. Enabling Multicluster Observability will deploy a way of forwarding of metrics to the Hub central Observability components. Including the Hub Cluster Monitoring that will forward this metrics. 

Finally, the Hub Multicluster Observability provides its own Grafana instance to show all the metrics from all the cluster (including their local ones).

# RHACM Multicluster Observality

RHACM Observability is an operator that enables a set of RHACM add-ons and controllers. 

When enabled, the management cluster creates a new NS `open-cluster-management-addon-observability` with an operator, grafana and thanos component to collect (and forward) records and alerts (metrics). 

When enabled, every spoke cluster will have installed a new add-on called `addon-observability-controller` that deploys a controller/operator and a collector. These will forward all the default (and custom created) alerts and records managed by the local Cluster Monitoring Operator. 

Requirements:
 * RHACM and MCE operators installed
 * Storage system:
   * For this example an Storage bucket S3 compatible. At the bottom there is an small section to create an S3 compatbile endpoing using ODF. 
   * Other supported cloud storage systems [available](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.3/html-single/observability/index#prerequisites-observability)
   
Resources consumption:
 * At hub level: 2701mCPU and 11972Mi to manage about 5 spoke clusters. More [here](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.10/html-single/observability/index#observability-pod-capacity-requests)

 * At spoke level: there is no documentation about it. But looking to the pods deployed (actually 2, `endpoint-observability-operator` and `metrics-collector-deployment`), this seems to request 2mCPU and 50Mi (`endpoint-observability-operator`) and 10mCPU and 100Mi(`metrics-collector-deployment`). Notice there are no limits sets, that would be something interesting. 
 
> All the work here done on OCP4.15 RHACM Hub. The Cluster Observability Operator is a Technology Preview feature only.

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

You can check how `managedcluster` CR of every spoke cluster contains now the `addon-observability-controller`:

```yaml
> oc get managedcluster sno4 -o jsonpath='{.metadata.labels}' | jq
{
  "feature.open-cluster-management.io/addon-cluster-proxy": "available",
  "feature.open-cluster-management.io/addon-config-policy-controller": "available",
  "feature.open-cluster-management.io/addon-governance-policy-framework": "available",
  "feature.open-cluster-management.io/addon-observability-controller": "available",
  "feature.open-cluster-management.io/addon-work-manager": "available",
<REDACTED>
}

```

In the spoke you will see the add-on:
```bash
> oc -n open-cluster-management-addon-observability get pods
NAME                                              READY   STATUS    RESTARTS   AGE
endpoint-observability-operator-78b7ddc74-wl8vb   1/1     Running   0          121m
metrics-collector-deployment-5956f9f8b-bjjmt      1/1     Running   0          121m

```


## Quick overview of spoke cluster's metrics

Prometheous metrics can be classified as [records](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)` or [alerts](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).

Once the Observability is enabled you will have a quick overview on the RHACM overview section:

![](assets/rhacm_observability_20240509110910810.png)

Clicking on each alert you will be forwarded to Grafana dashboard:

![](assets/rhacm_observability_20240509111008585.png)

Using Grafana observability dashboard (https://grafana-open-cluster-management-observability.apps.your-domain) you can also check some other metrics and dashboards:

![](assets/rhacm_observability_20240509111110717.png)

We can browse any of the Prometheous (from the Cluster Monitoring Operator) alert rule per spoke. For example, all the alerts in the hub sno4:

![](assets/rhacm_observability_20240509154535606.png)

The different collected, by default alerts, are:
 * HighOverallControlPlaneMemory
 * NodeClockNotSynchronising
 * PrometheusNotConnectedToAlertmanagers
 * UpdateAvailable
 * ViolatedPolicyReport
 * Watchdog

Or any of the Promethous exported record rules. Like the cluster memory usage on spoke vsno7:

![](assets/rhacm_observability_20240509154905599.png)

All these screenshots captured from the hub. 


### Playing with Alerts

For example, we can check ACM Policies violations in the last minutes:

![](assets/rhacm_observability_20240509162039652.png)

If we fix this Policy violation and we make it compliant:

![](assets/rhacm_observability_20240509170727025.png)




## Which metrics logs are collected per each spoke

who exports these metrics, the Openshift Cluster Monitoring Operator, the MultiCluster Observability Operator? Makes sense to have both? Could I disable CMO and only have the MultiCluster Observability Operator (with their own metrics/alerts)?


## Disabling an spoke from observability

You only have to label as disabled the `managedcluster` CR of the spoke:

```bash
[hub]> oc label managedcluster vsno5 observability=disabled
[hub]> > oc get managedcluster vsno5 -o jsonpath='{.metadata.labels}' | jq
{
  "feature.open-cluster-management.io/addon-cluster-proxy": "available",
  "feature.open-cluster-management.io/addon-config-policy-controller": "available",
  "feature.open-cluster-management.io/addon-governance-policy-framework": "available",
  "feature.open-cluster-management.io/addon-work-manager": "available",
  "observability": "disabled",
<REDACTED>
}

```

When `observability: disabled` the `addon-observability-controller` is no longer available. 

Inmediatly, you can observe how the spoke delete the addon:

```bash
[spoke]> oc -n open-cluster-management-addon-observability get pods -w
NAME                                               READY   STATUS    RESTARTS   AGE
endpoint-observability-operator-6c96c49c67-p69js   1/1     Running   0          147m
metrics-collector-deployment-69cfc56858-brbgc      1/1     Running   0          147m
metrics-collector-deployment-69cfc56858-brbgc      1/1     Terminating   0          148m
metrics-collector-deployment-69cfc56858-brbgc      0/1     Terminating   0          148m
endpoint-observability-operator-6c96c49c67-p69js   1/1     Terminating   0          148m
endpoint-observability-operator-6c96c49c67-p69js   0/1     Terminating   0          148m

```



## Real consumption of observability on spokes



# Creating an Storage bucket with ODF/CEPH

If you already have ODF in your claster (in my case with an internal CEPH cluster), you can create an object bucket through and object bucket claim.

![](assets/observability-2_20240508121803110.png)

The object bucket claim allows you to use on of your available StorageClasses:

![](assets/observability-2_20240508122016315.png)

Once the claim is created, an new object bucket is available with all the S3 endpoint and credentials: