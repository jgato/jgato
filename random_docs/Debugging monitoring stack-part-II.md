# Debugging monitoring stack part. II

In this chapter we will play with some tuning trying to get better performance on the monitoring stack. Better performing because of we will configure it to make a less intensive work about collecting metrics from the different services.

This is just testing to dig into the Prometheus/node_exporter performance. It is nothing recommended/official/supported. Just a test experiment.

The experiment will consist on getting the following metrics:

* Get the pods more CPU consuming during the last 30m

* Get the '/system/slice' CPU consuming to have the global slice consumption and kubelet, during the last 30m

We will take this metrics with:

* A clean cluster and A cluster with a big number of pods running.
  
  * Default monitoring stack configuration
  
  * Double scrap time for the metrics for 'node_exporter'
  
  * Double all the scraping times

Why we make special interest on doubling 'node_exporter' scrape interval? 'node_exporter' takes metrics about Pods, Containers, Memory Used by these, etc. These metrics are extracted from 'kubelet' cAdvisory, so, 'kubelet' is very affected by Prometheus gathering metrics. 

From Prometheus there are many different scrape_intervals, most of them set to 15s. To increase this is not easy because configuration is managed by operators. If you are using an OpenShift Red Hat supported environment, you cannot change that without breaking the support. Changing these intervals would affect other Openshift metrics/alarms and also the auto scaling feature. It is out of the scope of this tutorial to find out which value is better.

How we are burning the cluster? In a previous tutorial I explained how I use kube-burner (https://github.com/jgato/jgato/blob/main/random_docs/Debugging%20monitoring%20stack%20on%20Openshift.md) 

Following we show the results of the experiment. At the end of the document I show how I have managed to 'hack' the monitoring stack to change the metrics.



# Hacking the monitoring stack to change scrap intervals

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

## Patching the monitoring stack

With the two operators down. We can start patching and changing scraping interval. 

The different jobs and intervals for monitoring are stored on Secret called 'prometheus-k8s':

```yaml
> oc get -n openshift-monitoring secrets prometheus-k8s -o jsonpath='{.data.prometheus\.yaml\.gz}'  | base64 -d | gunzip -c
global:
  evaluation_interval: 30s
  scrape_interval: 30s
  external_labels:
    prometheus: openshift-monitoring/k8s
    prometheus_replica: $(POD_NAME)
rule_files:
- /etc/prometheus/rules/prometheus-k8s-rulefiles-0/*.yaml
scrape_configs:
- job_name: serviceMonitor/openshift-apiserver-operator/openshift-apiserver-operator/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - openshift-apiserver-operator
  scrape_interval: 30s
  scheme: https
  tls_config:
    insecure_skip_verify: false
    server_name: metrics.openshift-apiserver-operator.svc
    ca_file: /etc/prometheus/configmaps/serving-certs-ca-bundle/service-ca.crt
    cert_file: /etc/prometheus/secrets/metrics-client-certs/tls.crt
    key_file: /etc/prometheus/secrets/metrics-client-certs/tls.key
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - source_labels:
    - job
    target_label: __tmp_prometheus_job_name
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_app
    - __meta_kubernetes_service_labelpresent_app
    regex: (openshift-apiserver-operator);true
  - action: keep
```

So lets get all the Prometheus configuration about jobs and intervals. By default all set to 30s. We can change that to 60s and store the whole configuration on a file.

```bash
> oc get -n openshift-monitoring secrets prometheus-k8s \
    -o jsonpath='{.data.prometheus\.yaml\.gz}' \
    | base64 -d \
    | gunzip -c \
    | sed 's/30s/60s/g' > /tmp/file.yaml
```

Now we have in this file all intervals set to 60s. Here an example of the job for scraping node_exporter. Notice there are many jobs. This one (node_expoerter) is the only one we change in our experiment. And finally we change all of them to 60s.

```yaml
- job_name: serviceMonitor/openshift-monitoring/node-exporter/0                
  honor_labels: false                                                          
  kubernetes_sd_configs:                                                       
  - role: endpoints                                                            
    namespaces:                                                                
      names:                                                                   
      - openshift-monitoring                                                   
  scrape_interval: 60s                                                                                                                                                                                                                                        
  scheme: https                                                                
  tls_config:                                                                  
    insecure_skip_verify: false                                                
    server_name: node-exporter.openshift-monitoring.svc                        
    ca_file: /etc/prometheus/configmaps/serving-certs-ca-bundle/service-ca.crt 
    cert_file: /etc/prometheus/secrets/metrics-client-certs/tls.crt            
    key_file: /etc/prometheus/secrets/metrics-client-certs/tls.key             
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token      
```

With the file and the intervals configured as desired, we zip and encode again:

```yaml
> cat /tmp/file.yaml | gzip | base64 -w 0
H4sIAAAAAAAAA+2dW5OjOJbH3+tTODrm.......Ck+Z1uPvw/zuRltydbAgA=
```

Edit again the secret and change the config file:



```yaml
> oc -n openshift-monitoring edit secrets prometheus-k8s

apiVersion: v1                                                                 
data:                                                                          
  prometheus.yaml.gz: H4sIAAAAAAAAA+2dW5OjOJbH3+tTODrm.......Ck+Z1uPvw/zuRltydbAgA=

```

This configuration is reflected into the Prometheus Pods:

```bash
> oc -n openshift-monitoring debug pod/prometheus-k8s-0 -- cat /etc/prometheus/config_out/prometheus.env.yaml | grep 'node-exporter/0'  -A 7 
Defaulting container name to prometheus.
Use 'oc describe pod/prometheus-k8s-0-debug -n openshift-monitoring' to see all of the containers in this pod.

Starting pod/prometheus-k8s-0-debug ...
- job_name: serviceMonitor/openshift-monitoring/node-exporter/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - openshift-monitoring
  scrape_interval: 60s

Removing debug pod ...
```

Now, we can configure our scrape intervals as we need for our experiments.

# Burning the cluster

In the [previous tutorial](https://github.com/jgato/jgato/blob/main/random_docs/Debugging%20monitoring%20stack%20on%20Openshift.md) we have learn how to burn the cluster

So we have the cluster as:

```bash
sh-4.4# crictl ps | wc -l
268
sh-4.4# crictl pods | wc -l
239
sh-4.4# ip link show 2>/dev/null | wc -l
458
```