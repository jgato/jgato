# Debugging monitoring stack part. II

In this chapter we will play with some tunings trying to get better performance on the monitoring stack.

This is just testing to dig into the Prometheus/node_exporter performance. It is nothing recommended/official/supported. Just testing.

## Set the monitoring operator to unmanaged

Monitoring stack is managed by an Openshift Operator, so, anything not supported by the Operator configuration, cannot be changed. We set the operator to unmanaged, so it will no monitor any of its CRs. This make the operator and cluster unsupported. This is why this tutorial does not cover any kind of recommended procedure. Just testing.

Edit the ClusterVersion to set unmanaged the operator:

```yaml
apiVersion: v1
items:
- apiVersion: config.openshift.io/v1
  kind: ClusterVersion
  metadata:
    creationTimestamp: "2022-10-10T09:23:42Z"
    generation: 6
    name: version
    resourceVersion: "1043553"
    uid: e454f9c4-4009-4a87-b32f-85209881f683
  spec:
    channel: stable-4.10
    clusterID: c8943a89-be9c-47b9-af4b-f1d84f032ca4
    desiredUpdate:
      version: 4.10.9
    overrides:
    - group: apps/v1
      kind: Deployment
      name: cluster-monitoring-operator
      namespace: openshift-monitoring
      unmanaged: true
    upstream: https://api.openshift.com/api/upgrades_info/v1/graph
```

And now we can scale down the operator:

```bash
> oc -n openshift-monitoring scale deployments cluster-monitoring-operator --replicas=0
deployment.apps/cluster-monitoring-operator scaled

> oc -n openshift-monitoring scale deployments prometheus-operator --replicas=0
deployment.apps/prometheus-operator scaled
> oc -n openshift-monitoring get deployments
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
cluster-monitoring-operator   0/0     0            0           26h
kube-state-metrics            1/1     1            1           26h
openshift-state-metrics       1/1     1            1           26h
prometheus-adapter            2/2     2            2           25h
prometheus-operator           0/0     0            0           26h
telemeter-client              1/1     1            1           25h
thanos-querier                2/2     2            2           25h

```

## 

## Burning the cluster

In the previous tutorial we have learn how to burn the cluster
