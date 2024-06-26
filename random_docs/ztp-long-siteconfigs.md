*Just testing a possible bug on ZTP Siteconfig generator (kustomize plugin)*

# Testing very long ZTP Siteconfigs

Just trying to demonstrate a possible bug on [very long ZTP Siteconfigs](https://issues.redhat.com/browse/OCPBUGS-23194), caused for a bug on [Kustomization](https://github.com/kubernetes-sigs/kustomize/issues/5480).

After reproducing the bug, we will build an upstrem kustomize binary with Andreas Karis solution.

Testing environment OCP4.14, ZTP4.14, Openshift GitOps  (ArgoCD 2.8.4), Kustomize 

# Reproducing the bug

## Get Kustomize and Siteconfig generator plugin


Siteconfig generator plugin can be extracted from any 'ztp-site-generator' container image.

```bash
> mkdir /tmp/ztp-kustomize-plugin
> ZTP_SITE_GENERATOR_IMG=registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.14.2-6
> podman cp $(podman create --name policgentool --rm ${ZTP_SITE_GENERATOR_IMG}):/kustomize/plugin/ran.openshift.io /tmp/ztp-kustomize-plugin/
> ls /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig 
/tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig
```
For the current environment, the kustomize verion is 5.1.0: 

```bash
> oc -n openshift-gitops rsh openshift-gitops-repo-server-86f579fbcf-4dxbg
Defaulted container "argocd-repo-server" out of: argocd-repo-server, copyutil (init), kustomize-plugin (init), policy-generator-install (init)
sh-4.4$ kustomize version
v5.1.0

```

Or in my laptop:

```bash
> kustomize version
v5.2.1
```
In principle, this should not be very relevant. The bug should be, mostly, in all versions. Or the latest ones. 

## Create a long Siteconfig

Creating a very long Siteconfig with 100 SNOs.

We start with a Siteconfig with just one SNO:

```yaml
apiVersion: ran.openshift.io/v1 
kind: SiteConfig
metadata:                                                                                                                                                                                                                                                     
  name: "site-multicluster-sno"                                                                                                                                                                                                                               
  namespace: "site-multicluster-sno"
spec:
  baseDomain: "sno.hpecloud.org"
  pullSecretRef:
    name: "assisted-deployment-pull-secret"
  clusterImageSetNameRef: "img4.12.3-x86-64-appsub"
  sshPublicKey: "ssh-rsa AAAAB3NzaC1yc2EAAA.....q5EMHgo
pyiYV7swJEnFzAUzaiu8DP1FYNJyocRvp6AZpbIlyFoabyq+o2yn2Fhny6gs= jgato@provisioner.el8k.hpecloud.org"
  clusters:
  - clusterName: "sno1"
    networkType: "OVNKubernetes"
    clusterLabels:
      logical-group: "mb-du-sno"
      common: "true"
      site: "sno1"
    clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
    serviceNetwork:
      - 172.30.0.0/16
    machineNetwork:
      - cidr: "10.19.10.64/26"
    nodes:
      - hostName: "sno1"
        bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
        bmcCredentialsName:
          name: "sno1-bmc-secret"
        bootMACAddress: "52:54:00:ff:b8:5"
        bootMode: "UEFI"
        rootDeviceHints:
          deviceName: "/dev/sda"
```

We create a bunch of clusters for the 'spec.clusters':

```bash
> for i in `seq 1 100` ; do
cat <<EOF >> clusters-section.yaml
- clusterName: "sno${i}"
  networkType: "OVNKubernetes"
  clusterLabels:
    logical-group: "mb-du-sno"
    common: "true"
    site: "sno"
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  machineNetwork:
    - cidr: "10.19.10.64/26"
  nodes:
    - hostName: "sno${i}"
      bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/  Systems/00000000-0000-0000-0000-000000000001"
      bmcCredentialsName:
        name: "sno-bmc-secret"
      bootMACAddress: "52:54:00:ff:b8:5"
      bootMode: "UEFI"
      rootDeviceHints:
        deviceName: "/dev/sda"
EOF
done

```

We use jq to use the bunch of clusters to our short Siteconfig to have a very long Siteconfig (of whatever number of SNOs included/replicated):

```bash
> yq '.spec.clusters = load("clusters-section.yaml")' short-siteconfig-one-sno.yaml  > long-siteconfig.yaml
> cat long-siteconfig.yaml | grep clusterName | wc -l
100

```


## Run the plugin locally

 > Using kustomize 5.2.1 and site generator plugin 4.14.2
 
Once you have extracted the Siteconfig generator plugin and you have kustomize installed, you can run the generator over a local Siteconfig.

First, add your Siteconfig to a kustomization.yaml:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

generators:
  - short-siteconfig-one-sno.yaml
```

Run the plugin, this time  with a normal working Siteconfig:

```
> export KUSTOMIZE_PLUGIN_HOME=/tmp/ztp-kustomize-plugin/
> kustomize build ./ --enable-alpha-plugins                                                                                 

apiVersion: v1                                                                                                                                                                                                                                                
kind: Namespace                                                                                                                
metadata:                                                                                                                      
  annotations:                                                                                                                 
    argocd.argoproj.io/sync-wave: "0"                                                                                          
    ran.openshift.io/ztp-gitops-generated: '{}'                                                                                
  labels:                                                                                                                      
    name: sno5                                                                                                                 
  name: sno5                                                                                                                                                                                                                                                  
---                                                                                                                            
apiVersion: v1                                                                                                                 
data:                                                                                                                          
  01-container-mount-ns-and-kubelet-conf-master.yaml: |                                                                        
    apiVersion: machineconfiguration.openshift.io/v1                                                                           
    kind: MachineConfig                                                                                                                                                                                                                                       
    metadata:                                                                                                                                                                                                                                                 
        annotations:                                    
<REDACTED>
```

Plugin starts creating all the CRs correctly.

## Reach the limit

In my environment, I reached the error creating a Siteconfig (according to the previous example) with 213 SNOs.

```
# total n. of clusters
> cat long-siteconfig.yaml | grep clusterName | wc -l                                                          
213
```

The number of SNOs is not the only reason, so dont take it as the main problem, it is just a reference. It would be reached with less SNOs, if each one would contain more information. In this case, 199 SNO, with about 22 lines per cluster, and 4698 lines in total for the Siteconfig.

```
# lines per Cluster
> yq '.spec.clusters[0]' long-siteconfig.yaml  | wc -l
22
# total n. lines siteconfig
> cat long-siteconfig.yaml | wc -l
4698

```
It fails:
```
> export KUSTOMIZE_PLUGIN_HOME=/tmp/ztp-kustomize-plugin/
> kustomize build ./ --enable-alpha-plugins
Error: failure in plugin configured via /tmp/kust-plugin-config-208601320; fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long: fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long


```

If I reduce the number of clusters to 212,

```bash
> cat long-siteconfig.yaml | grep clusterName | wc -l                                                          
212
> cat long-siteconfig.yaml | wc -l                                                          
4676

```

it works correctly. 
```bash
> kustomize build ./ --enable-alpha-plugins | more
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    ran.openshift.io/ztp-gitops-generated: '{}'
  labels:

<REDACTED>
```

## Get the patched kustomize

The patch was already [merged](https://github.com/kubernetes-sigs/kustomize/pull/5510) into master. So, I just cloned the repo (01/03/2024) and build it:
```
> cd kustomize
> CGO_ENABLED=0 GO111MODULE=on go build     -ldflags="-s -X sigs.k8s.io/kustomize/api/provenance.version=5.3.1-jgato \
    -X sigs.k8s.io/kustomize/api/provenance.buildDate=${DATE}"
> ./kustomize version
5.3.1-jgato
```

Back to the 213 SNOs to reach the limit, but now with my compiled kustomize:

```bash
> cat long-siteconfig.yaml | grep clusterName | wc -l                                                          
213
> /home/jgato/Projects-src/github/kustomize/kustomize/kustomize build ./ --enable-alpha-plugins | more
# Warning: 'bases' is deprecated. Please use 'resources' instead. Run 'kustomize edit fix' to update your Kustomization automatically.
W0301 11:17:27.142616  335629 execplugin.go:194] KUSTOMIZE_PLUGIN_CONFIG_STRING exceeds hard limit of 131071 characters, the environment variable will be omitted
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    ran.openshift.io/ztp-gitops-generated: '{}'
  labels:
<REDACTED>
```

We have the warning:

```bash
W0301 11:17:27.142616  335629 execplugin.go:194] KUSTOMIZE_PLUGIN_CONFIG_STRING exceeds hard limit of 131071 characters, the environment variable will be omitted
```

but worked.

Even going very high on number of SNOs:

```bash
> cat long-siteconfig.yaml | grep clusterName | wc -l                                                          
1000
> cat long-siteconfig.yaml | wc -l
22012

```

worked correctly.

# Reaching the limit for MultiNode clusters

Above there is one example about reaching the limit for a Siteconfig with many SNOs cluster. Now, we will test something similar, but one cluster with many nodes.

At the end the limit is the same, because it is actually a matter of bytes, or lines (which is not so accurate but is simpler to understand). Here we want to try to find the limit on nodes inside a MultiNode cluster, to find out how many nodes you can have. Again, it is not accurate, because it is really about bytes. But, considering a base MultiNode cluster definition of about 68 lines:

```yaml
apiVersion: ran.openshift.io/v1
kind: SiteConfig
metadata:
  name: "site-cluster-mutinode"
  namespace: "site-multicluster-mutinode"
spec:
  baseDomain: "multinode.hpecloud.org"
  pullSecretRef:
    name: "assisted-deployment-pull-secret"
  clusterImageSetNameRef: "img4.12.3-x86-64-appsub"
  sshPublicKey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCei2UnkBB9g9DPhu4fpMFKmrlhR9UIYYPet61WF3qr6Rp2LkxEhZtbRk6tZjaiVXo/Ff6rsayyoEy86bPCE+4/Kl+3V/KueKW2fgxz/tg1uLiDerWj8+J8KAGJ8TsBAl3cWYYQtHxlwCnyPSmspWB/UegNTd+0cHhkPiTYd6wyl.....n2Fhny6gs= jgato@provisioner.el8k.hpecloud.org"
  clusters:
  - clusterName: "cluster-example"
    networkType: "OVNKubernetes"
    clusterLabels:
      logical-group: "multinode"
      common: "true"
      site: "cluster-example"
    clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
    apiVIP: 10.19.10.80
    ingressVIP: 10.19.10.70
    serviceNetwork:
      - 172.30.0.0/16
    additionalNTPSources:
      - 0.rhel.pool.ntp.org
      - 1.rhel.pool.ntp.org
    extraManifests:
      searchPaths:
        - extra-manifests/
        - custom-manifests/
      filter:
        inclusionDefault: include
    nodes:
      - hostName: "master01"
        role: master
        bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
        bmcCredentialsName:
          name: "bmc-secret"
        bootMACAddress: "52:54:00:ff:b8:a1"
        bootMode: "UEFI"
        rootDeviceHints:
          deviceName: "/dev/sda"
        crTemplates:
          NMStateConfig: "NMStates/nmstate-node.yaml"
      - hostName: "master02"
        role: master
        bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
        bmcCredentialsName:
          name: "bmc-secret"
        bootMACAddress: "52:54:00:ff:b8:a2"
        bootMode: "UEFI"
        rootDeviceHints:
          deviceName: "/dev/sda"
        crTemplates:
          NMStateConfig: "NMStates/nmstate-node.yaml"
      - hostName: "master03"
        role: master
        bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
        bmcCredentialsName:
          name: "bmc-secret"
        bootMACAddress: "52:54:00:ff:b8:a3"
        bootMode: "UEFI"
        rootDeviceHints:
          deviceName: "/dev/sda"
        crTemplates:
          NMStateConfig: "NMStates/nmstate-node.yaml"    
```
The base Siteconfig only contains the Master to make it as much as correct as possible. Workers will add many later.

And, each worker node definition will have about 10 lines

```yaml
        - hostName: "node98"
          bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
          bmcCredentialsName:
            name: "bmc-secret"
          bootMACAddress: "52:54:00:ff:b8:aa"
          bootMode: "UEFI"
          rootDeviceHints:
            deviceName: "/dev/sda"
          crTemplates:
            NMStateConfig: "NMStates/nmstate-sno5.yaml"
```

Notice that network configuration is managed externally as recommended [here](https://access.redhat.com/solutions/7058930). This separates the networks definitions apart from the Siteconfig, to make the length of Siteconfigs more manageable.

We use previous example script to build a long Siteconfig with as many nodes as we desired. With an script such as:

```bash
rm clusters-node.yaml
rm NMStates/nmstate-node-*.yaml
for i in `seq 1 ${1}` ; do
cat <<EOF >> clusters-node.yaml
    - hostName: "node${i}"
      role: worker
      bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
      bmcCredentialsName:
        name: "bmc-secret"
      bootMACAddress: "52:54:00:ff:b8:aa"
      bootMode: "UEFI"
      rootDeviceHints:
        deviceName: "/dev/sda"
      crTemplates:
        NMStateConfig: "NMStates/nmstate-node-${i}.yaml"
EOF

cat <<EOF > NMStates/nmstate-node-${i}.yaml 
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    ran.openshift.io/ztp-gitops-generated: '{}'
  labels:
    app.kubernetes.io/instance: clusters
    nmstate-label: node${i}
  name: node${i}
  namespace: cluster-example
spec:
  interfaces:
    - name: "eno3"
      macAddress: "94:40:c9:1f:bf:8e" 
  config:
    interfaces:
      - name: eno3
        type: ethernet
        state: up
        ipv4:
          enabled: true
          address:
            - ip: 10.19.10.74
              prefix-length: 26
        ipv6:
          enabled: false
    dns-resolver:
      config:
        server:
          - 10.19.10.71
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 10.19.10.126
          next-hop-interface: eno3
EOF
done
```

To build a `clusters-node.yaml` file, and all the needed NMStates, that you can add to our previous base Siteconfig.

```
> yq '.spec.clusters[0].nodes += load("clusters-node.yaml")' short-siteconfig-multinode.yaml  > long-siteconfig.yaml
> cat long-siteconfig.yaml | grep hostName | wc -l
103
> cat long-siteconfig.yaml | wc -l
1168
```

Very easily we have the base of our Siteconfig with 68 lines and 100 nodes with 11 lines each.

Playing with the nodes generation we can find the limit using the not patched Kustomize:

The first time I reached the limit:

```bash
> kustomize build ./ --enable-alpha-plugins
Error: failure in plugin configured via /tmp/kust-plugin-config-1856370234; fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long: fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long
> cat long-siteconfig.yaml | grep hostName | wc -l
330

```
So 330 nodes on a Multinode of the previous conditions will hit the bug again. But, again, this depends on the really length of the Siteconfig. Previously on SNO we reached earlier, because that Siteconfig contain many clusters definition, one per SNO, plus the info of each nodes. Again, is more related to the bytes of the file that the number of nodes.

Lines would be more accurate, but still not fully accurate:

With 330 nodes we have 3665 lines that reach the limit
```bash
> cat long-siteconfig.yaml | wc -l
3665

```
with 329 nodes, we have  3525 lines and it works:
```bash
> cat long-siteconfig.yaml | grep hostName | wc -l
332
> cat long-siteconfig.yaml | wc -l
3654
> kustomize build ./ --enable-alpha-plugins
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    ran.openshift.io/ztp-gitops-generated: '{}'
  labels:
    name: cluster-example
  name: cluster-example
---
apiVersion: v1
data:
  50-mcp-ht.yaml: |
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfigPool
    metadata:
        annotations:
            ran.openshift.io/ztp-gitops-generated: '{}'
        labels:
            pools.operator.machineconfiguration.openshift.io/ht: ""
        name: ht
    spec:
        machineConfigSelector:
            matchExpressions:
                - key: machineconfiguration.openshift.io/role
                  operator: In

<REDACTED>

```

this would help you to determine the limits aproximately. 

> More production environments will have longer base cluster definition (this does not affect too much) and longer nodes definitions (that would affect much more to this 332 nodes). Consider better the number of lines (even if still not 100% accurate)

# Fixed Kustomize

with a Kustomize fixed version we can go higher, here I just want to try how much is feasible until it takes to long to render all the content:

``` bash
> cat long-siteconfig.yaml | grep hostName | wc -l
330
> kustomize build ./ --enable-alpha-plugins > /dev/null 
Error: failure in plugin configured via /tmp/kust-plugin-config-2379186691; fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long: fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long

> time  /home/jgato/Projects-src/github/kustomize/kustomize/kustomize build ./ --enable-alpha-plugins > /dev/null 
W0415 17:43:31.642261  543526 execplugin.go:194] KUSTOMIZE_PLUGIN_CONFIG_STRING exceeds hard limit of 131071 characters, the environment variable will be omitted

real	0m0,634s
user	0m0,818s
sys	0m0,074s

```

Our previous limit can be fixed with the fixed Kustomize really quickly. Lets try more demanding scenarios:

```bash
> cat long-siteconfig.yaml | grep hostName | wc -l
1003
> time  /home/jgato/Projects-src/github/kustomize/kustomize/kustomize build ./ --enable-alpha-plugins > /dev/null 
W0415 17:45:04.965343  546339 execplugin.go:194] KUSTOMIZE_PLUGIN_CONFIG_STRING exceeds hard limit of 131071 characters, the environment variable will be omitted

real	0m3,061s
user	0m3,721s
sys	0m0,126s
```

Lets go ahead with 3003 nodes:

```bash
> cat long-siteconfig.yaml | grep hostName | wc -l
3003

> time  /home/jgato/Projects-src/github/kustomize/kustomize/kustomize build ./ --enable-alpha-plugins > /dev/null 
W0415 17:52:49.934709  575997 execplugin.go:194] KUSTOMIZE_PLUGIN_CONFIG_STRING exceeds hard limit of 131071 characters, the environment variable will be omitted

real	0m35,995s
user	0m38,164s
sys	0m0,318s

```

And something crazy: 

```bash
 cat long-siteconfig.yaml | grep hostName | wc -l
10003
┌─[🕙 17:46:32] :ztp-deployments/ZTP/HubClusters/el8k/SpokeClusters/ztp-gitops/gitop-repo/siteconfig-4.14 [ main]+109205/-3830 [!?] 
└──> time  /home/jgato/Projects-src/github/kustomize/kustomize/kustomize build ./ --enable-alpha-plugins > /dev/null 
W0415 17:46:38.591343  567203 execplugin.go:194] KUSTOMIZE_PLUGIN_CONFIG_STRING exceeds hard limit of 131071 characters, the environment variable will be omitted
^C

real	4m29,646s
user	4m40,072s
sys	0m1,014s

```
After 4 minutes I had to "control+c", but it is not reasonable or feasible to have 10K nodes. Just trying to reach the limits. The CPU of the laptop was not doing nothing so, most likely the plugin was....