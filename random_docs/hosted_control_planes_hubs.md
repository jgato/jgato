
# Creating a Hub as an HostedControlPlane with Openshift virtualization

>Warning: this is a quick tutorial on how to create a Hub with a cluster created with HostedControlPlane and Openshift Virtualization

Some concepts about what we want to do:
 * Management cluster: basically an Openshift cluster that manages other clusters.
 * A hub: it is basically a management cluster with an specific configuration or combination of operators. Like the [Telco Hub](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/scalability_and_performance/telco-hub-ref-design-specs)
 * HostedControlPlane (HCP) or Hypershift: you create new clusters, where the usual control planes as pods on a management cluster without the need for dedicated virtual or physical machines for each control plane. And the workers can be created from different providers. In the case of this tutorial: virtual machines.
 * Openshift Virtualization to provide the infrastructure as VMS
 
So, the architecture will be:

```
┌──────────────────────────────────────────────────┐
│       Regular Compact Management Cluster         │
│   (Telco Hub + OCP Virtualization + MetalLB)     │
└─────────────────────┬────────────────────────────┘
                      │
            HCP + OpenShift Virtualization
                      │
                      ▼
          ┌──────────────────────────────┐
          │      Hosted Compact Cluster  │
          └────┬──────────┬──────────┬───┘
               │          │          │
               ▼          ▼          ▼
         ┌─────────┐ ┌─────────┐ ┌─────────┐
         │  sno1   │ │  sno2   │ │  sno3   │
         └─────────┘ └─────────┘ └─────────┘
         (baremetal) (baremetal) (baremetal)
```
 
 The regular compact management cluster it is just a our usual Telco Hub + Openshift Virtualization + Metallb
 
# Configure the Management Cluster

Management cluster is composed by the usual hub operators:
 * ACM/MCE
 * local storage operator
 * ODF
 * Openshift Gitops
 * TALM
In addition for this case:
 * Openshift Virtualization
 * Metallb
 
How to configure all these operators (hub) is not covered in this tutorial.

In addition I created a Metallb IPAddressPool that will be used by the hosted clusters created by HCP.

```
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metallb
  namespace: metallb-system
spec:
  addresses:
    - 10.6.77.100-10.6.77.120
    - '2620:0052:0009:164d:0000:0000:0000:1111 - 2620:0052:0009:164d:0000:0000:0000:ffff'
  autoAssign: true
  avoidBuggyIPs: false
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - metallb

```

Becareful with the ranges to avoid automatic assigements with IPs already in use. This is why I use ranges that I am sure I am not using.

Releted to HCP, configure the management cluster as explained [here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/hosted_control_planes/deploying-hosted-control-planes#hcp-virt-prereqs_hcp-deploy-virt).

# Create the Hosted Compact Cluster

## Using HCP cli

From the management cluster, download the [HCP cli](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/hosted_control_planes/preparing-to-deploy-hosted-control-planes#hcp-cli-console_hcp-cli)
An hcp cluster is created with hcp cli

```
> hcp create cluster kubevirt   --name hcp-3   --node-pool-replicas 3   
	\--pull-secret /home/jgato/.config/containers/auth.json   
	\--memory 16Gi   --cores 8   --etcd-storage-class=ocs-storagecluster-cephfs   
	\--arch amd64   --release-image quay.io/openshift-release-dev/ocp-release:4.21.0-multi
	\--enable-cluster-capabilities baremetal
	\--cluster-cidr 10.132.0.0/14 
	\--cluster-cidr fd03::/48
	\--service-cidr 172.31.0.0/16 
	\--service-cidr fd04::/112

```

Dual stack is working correctly:

> curl https://[2620:52:9:164d::]:6443/readyz -k
ok

> curl https://10.6.77.102:6443/readyz -k
ok

and the network is properly  configured:

```yaml
> oc get network cluster -o yaml
apiVersion: config.openshift.io/v1
kind: Network
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    release.openshift.io/create-only: "true"
  creationTimestamp: "2026-04-17T10:15:36Z"
  generation: 2
  labels:
    hypershift.openshift.io/managed: "true"
  name: cluster
  ownerReferences:
  - apiVersion: config.openshift.io/v1
    kind: ClusterVersion
    name: version
    uid: 086d7308-0a85-40a3-95fe-4668d4b76e1d
  resourceVersion: "1630"
  uid: e310b368-e55d-4d80-b61a-6afd9427e9f9
spec:
  clusterNetwork:
  - cidr: 10.132.0.0/14
    hostPrefix: 23
  - cidr: fd00::/48
    hostPrefix: 64
  networkDiagnostics:
    mode: ""
    sourcePlacement: {}
    targetPlacement: {}
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.31.0.0/16
  - fd00:fd::/112

```

Even if the management and hosted cluster are correctly configured with dual-stack, the IPv6 address of the hosted cluster will not work. More details on the bug [here](https://redhat.atlassian.net/browse/OCPBUGS-84303).

## Using Manifests

[ToDo]

# Configure Hosted Cluster

## Install the hub usual operators

Install:
 * ACM/MCE
 * Openshift GitOps
 * TALM
 * Metallb
ODF is not needed, because it is exported from the management cluster. Neither the local-storage operator


## Configuring the baremetal operator

Create different services to make metal3 (from hcp-3) containers externally accessible.

In the hosted cluster create:

```
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
  namespace: openshift-machine-api
spec:
  type: NodePort
  selector:
    baremetal.openshift.io/cluster-baremetal-operator: metal3-state
  ipFamilies:   
  - IPv4                                  
  - IPv6                                  
  ipFamilyPolicy: PreferDualStack
  ports:
  - name: http
    protocol: TCP
    port: 6180
    targetPort: 6180
    nodePort: 30701
  - name: vmedia-https
    protocol: TCP
    port: 6183
    targetPort: 6183
    nodePort: 30702
  - name: ironic
    protocol: TCP
    port: 6385
    targetPort: 6385
    nodePort: 30703
```


In a Hosted Control Plane setup, Ironic runs inside a guest cluster (hcp-3), not on a node directly reachable by baremetal servers. So there has to be some indirection — a LoadBalancer or NodePort — to bridge the physical server network to the pod network.
Therefore, in the management cluster create LoadBalancer: (configure the nodepool and create it in the NS of the hosted cluster):

```
apiVersion: v1
kind: Service
metadata:
  name: lbsvc
  namespace: clusters-hcp-3
spec:
  type: LoadBalancer
  ipFamilies:   
    - IPv4                                  
    - IPv6                                  
  ipFamilyPolicy: PreferDualStack 
  allocateLoadBalancerNodePorts: false
  ports:
  - name: http
    protocol: TCP
    port: 6180
    targetPort: 30701
  - name: httpd
    protocol: TCP
    port: 6183
    targetPort: 30702
  - name: ironic
    protocol: TCP
    port: 6385
    targetPort: 30703
  selector:
    hypershift.openshift.io/nodepool-name: hcp-3

```


In the hosted cluster the Provisioning:

```
apiVersion: metal3.io/v1alpha1
kind: Provisioning
metadata:
  name: provisioning-configuration
spec:
  provisioningNetwork: Disabled
  externalIPs:
  - 10.6.77.104
  - 2620:52:9:164d::1112
  watchAllNamespaces: true  
```

Use the external IP from the created lbsvc (LoadBalancer, from the management cluster)

```bash
> oc -n clusters-hcp-3 get svc lbsvc
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP                        PORT(S)                      AGE
lbsvc   LoadBalancer   172.30.145.99   10.6.77.104,2620:52:9:164d::1112   6180/TCP,6183/TCP,6385/TCP   21s


```


Graphically, this is what we are trying to do:

```

    MANAGEMENT CLUSTER (hub-2)
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   namespace: clusters-hcp-3                                     │
   │  ┌──────────────────────────────────────────────────────────┐   │
   │  │                                                          │   │
   │  │   lbsvc (LoadBalancer)                                   │   │
   │  │   external IP: 10.6.77.102                               │   │
   │  │   ports: 6180 → 30701 / 6183 → 30702 / 6385 → 30703     │   │
   │  │                                                          │   │
   │  └───────────────────────┬──────────────────────────────────┘   │
   │                          │ selector: nodepool hcp-3             │
   └──────────────────────────┼──────────────────────────────────────┘
                              │ routes to NodePorts on hcp-3 workers
                              ▼
    HOSTED CLUSTER (hcp-3) — worker nodes
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   namespace: openshift-machine-api                              │
   │  ┌──────────────────────────────────────────────────────────┐   │
   │  │  nodeport-svc                                            │   │
   │  │  30701 (http) / 30702 (vmedia-https) / 30703 (ironic)   │   │
   │  └───────────────────────┬──────────────────────────────────┘   │
   │                          │                                      │
   │  ┌───────────────────────▼──────────────────────────────────┐   │
   │  │  metal3 pod                                              │   │
   │  │                                                          │   │
   │  │   metal3-ironic   (10.128.1.130)                         │   │
   │  │   ├─ external_http_url:     https://10.6.77.102:6183     │   │
   │  │   └─ external_callback_url: https://10.6.77.102:6385     │   │
   │  │                                                          │   │
   │  │   /shared/html/redfish/boot-<uuid>.iso  ◄── ISO built    │   │
   │  │                  │          with callback URL embedded    │   │
   │  └──────────────────┼───────────────────────────────────────┘   │
   │                     │                                           │
   └─────────────────────┼───────────────────────────────────────────┘
                         │
            1) BMC fetches ISO via https://10.6.77.102:6183
                         │
                         ▼
    PHYSICAL NETWORK (10.6.x.x)
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   sno4 — HPE ProLiant e910                                      │
   │   BMC: 10.6.75.176                                              │
   │                                                                 │
   │   ┌─────────────────────────────────────────────────────────┐   │
   │   │  boots inspection ISO via Redfish VirtualMedia          │   │
   │   │                                                         │   │
   │   │  IPA (Ironic Python Agent) starts                       │   │
   │   │                                                         │   │
   │   │  2) calls back to external_callback_url                 │   │
   │   │     https://10.6.77.102:6385  ──────────────────────────┼───┼──► lbsvc
   │   │                                                         │   │    → NodePort 30703
   │   └─────────────────────────────────────────────────────────┘   │    → metal3-ironic :6385
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘


```

Now we have BMO configured.

```
> oc -n openshift-machine-api get pod
NAME                                                        READY   STATUS    RESTARTS   AGE
cluster-baremetal-operator-hostedcluster-5669fc4f78-jgwqk   1/1     Running   0          23m
metal3-5fdfd6795b-msxrz                                     3/3     Running   0          8m55s
metal3-baremetal-operator-f9884ddb-wbpt6                    1/1     Running   0          8m55s
metal3-image-customization-c99865f56-jb6zl                  1/1     Running   0          8m52s

```

The assisted service is running:

```
> oc -n multicluster-engine get pod assisted-service-fc976457f-2xfbx
NAME                               READY   STATUS    RESTARTS        AGE
assisted-service-fc976457f-2xfbx   2/2     Running   200 (20m ago)   16h

```

Siteconfig operator has been enabled in the MCH:

```
> oc -n open-cluster-management get pod siteconfig-controller-manager-55499c95b-wrnzv
NAME                                            READY   STATUS    RESTARTS      AGE
siteconfig-controller-manager-55499c95b-wrnzv   1/1     Running   3 (13h ago)   17h

```

Now, you hosted hub, should be able to allow your usual ZTP/GitOps deployments.

## Enabling observability on a hcp hub

The hosted cluster does not have ODF installed. It just uses like ODF pass-through from the management cluster, to have the StorageClass:

```
> oc get sc
NAME                                   PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
kubevirt-csi-infra-default (default)   csi.kubevirt.io   Delete          Immediate           true                   27h
```

So, the object's bucket storage is created in the management cluster, and accessed "remotely" from the hosted cluster. The hosted cluster (hcp-3) lives inside the management cluster, as it was another cluster. So, it access the management cluster "remotely".

```
> cat <<EOF | oc apply -f -
---
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: thanos-hcp-3
  namespace: open-cluster-management-observability
spec:
  bucketName: observability-hcp-3
  storageClassName: openshift-storage.noobaa.io

EOF
```

In order to get the S3 bucket credentials inspect the content of the secret called thanos within the namespace open-cluster-management-observability.

```
> oc -n open-cluster-management-observability get secret thanos-hcp-3 -o yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: dFBMak9.....WUY=
  AWS_SECRET_ACCESS_KEY: MFZkNlB...E4yL1ExcQ==
kind: Secret
metadata:
  creationTimestamp: "2026-04-14T14:13:05Z"
  finalizers:
  - objectbucket.io/finalizer
  labels:
    app: noobaa
    bucket-provisioner: openshift-storage.noobaa.io-obc
    noobaa-domain: openshift-storage.noobaa.io
  name: thanos-hcp-3
  namespace: open-cluster-management-observability
  ownerReferences:
  - apiVersion: objectbucket.io/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: ObjectBucketClaim
    name: thanos-hcp-3
    uid: 40ff696c-f859-4b6f-8239-5403603480b4
  resourceVersion: "152678738"
  uid: 0f47584f-c1c6-4f8b-be5f-4752fc85cef0
type: Opaque

```

Notice: they key/access are base64 encoded. When using them in the secret, decode first.

To access externally (hcp-3) we have to the s3 external route:


```
> oc get route s3 -n openshift-storage
NAME   HOST/PORT                                                            PATH   SERVICES   PORT       TERMINATION       WILDCARD
s3     s3-openshift-storage.apps.hub-2.el8k.se-lab.eng.rdu2.dc.redhat.com          s3         s3-https   reencrypt/Allow   None
```

Now, in the hosted cluster we enable MultiClusterHub Observability.

```bash
# Create the NS
> oc create ns open-cluster-management-observability
namespace/open-cluster-management-observability created

# This secrets contains the S3 connection to the created
# bucket
>  cat <<EOF | oc apply -f -
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
      bucket:  "observability-hcp-3"
      endpoint: "s3-openshift-storage.apps.hub-2.el8k.se-lab.eng.rdu2.dc.redhat.com:443"
      insecure: false
      http_config:
        insecure_skip_verify: true
      access_key: "tPLjO...Yln6YF="
      secret_key: "0Vd6Pc...8HN2/Q1q"
EOF

# create a pull-secret
> DOCKER_CONFIG_JSON=`oc extract secret/pull-secret -n openshift-config --to=-`

> oc create secret generic multiclusterhub-operator-pull-secret \
    -n open-cluster-management-observability \
    --from-literal=.dockerconfigjson="$DOCKER_CONFIG_JSON" \
    --type=kubernetes.io/dockerconfigjson
```

and finally, we create the MCO:

```
> cat <<EOF | oc apply -f -
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
  annotations:
    mco-disable-alerting: "true"
spec:
  advanced:
    retentionConfig:
      blockDuration: 2h
      deleteDelay: 48h
      retentionInLocal: 24h
      retentionResolutionRaw: 15d
  enableDownsampling: false
  observabilityAddonSpec:
    enableMetrics: true
    interval: 300
  storageConfig:
    alertmanagerStorageSize: 10Gi
    compactStorageSize: 100Gi
    metricObjectStorage:
      key: thanos.yaml
      name: thanos-object-storage
    receiveStorageSize: 25Gi
    ruleStorageSize: 10Gi
    storeStorageSize: 25Gi
    storageClass: kubevirt-csi-infra-default
EOF
```

The MCO Dashboard:

![](assets/hosted_control_planes_hubs_20260414170739782.png)

# ZTP GitOps

This tutorial is not intended to explain on how to use ZTP/GitOps to deploy new spoke clusters. In this case, using the hosted cluster that is also a hub.

I have tested working correctly the following:
 * Hosted Hub IPv4 and spoke IPv4
 * Hosted Hub dual-stack and