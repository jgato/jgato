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
  sshPublicKey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCei2UnkBB9g9DPhu4fpMFKmrlhR9UIYYPet61WF3qr6Rp2LkxEhZtbRk6tZjaiVXo/Ff6rsayyoEy86bPCE+4/Kl+3V/KueKW2fgxz/tg1uLiDerWj8+J8KAGJ8TsBAl3cWYYQtHxlwCnyPSmspWB/UegNTd+0cHhkPiTYd6wylgmbBi9MWOAISkXOLWUjsOmKUKiTL
kfX2VWqvkk8BH2/blHp0xCSZ2NWifc+VmCvz+M36mj0aRF5dEmfdy+wg7m9wi2/Hq59+NLGBef3kKjBnj0A/K0wFfT0ufi03OkztDOY7Y0xxIkl8Bi/Hof4rDlfKVKA9hcMSo3TY2o0asmTTXUhGZ/FVuZcIZpULOFMXKUR3oKeqnr/dff32IHVwgYb8n8C5zUepWu7tVUKnvxZ0Gwajy1Ru+xjrlROFT+761faJHmG5Ev/EdwKHkXHq5EMHgo
pyiYV7swJEnFzAUzaiu8DP1FYNJyocRvp6AZpbIlyFoabyq+o2yn2Fhny6gs= jgato@provisioner.el8k.hpecloud.org"
  clusters:
  - clusterName: "sno5"
    networkType: "OVNKubernetes"
    clusterLabels:
      logical-group: "mb-du-sno"
      common: "true"
      site: "sno5"
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
          name: "sno5-bmc-secret"
        bootMACAddress: "52:54:00:ff:b8:5"
        bootMode: "UEFI"
        rootDeviceHints:
          deviceName: "/dev/sda"
```

We create a bunch of clusters for the 'spec.clusters':

```bash
> for i in `seq 1 100` ; do
cat <<EOF >> clusters-section.yaml
clusterName: "sno${i}"
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
    bmcAddress: "redfish-virtualmedia://10.19.10.71:6443/redfish/v1/Systems/00000000-0000-0000-0000-000000000001"
    bmcCredentialsName:
      name: "sno-bmc-secret"
    bootMACAddress: "52:54:00:ff:b8:5"
    bootMode: "UEFI"
    rootDeviceHints:
      deviceName: "/dev/sda
EOF
done

```

We use jq to add the bunch of clusters to our short Siteconfig to have a very long Siteconfig (of whatever number of SNOs included/replicated):

```bash
> yq '.spec.clusters += loadstr("clusters-section.yaml")' short-siteconfig-one-sno.yaml  > long-siteconfig.yaml
> cat long-siteconfig.yaml | grep clusterName | wc -l
201

```


## Run the plugin locally

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

In my environment, I reached the error creating a Siteconfig (according to the previous example) with 199 SNOs.

```
# total n. of clusters
> cat long-siteconfig.yaml | grep clusterName | wc -l                                                          
199
```

The number of SNOs would not be the only reason. It would be reached with less SNOs, if each one would contain more information. In this case, 150 SNO, with about 20 lines per cluster, and 4391 lines in total for the Siteconfig.

```
# lines per Cluster
> yq '.spec.clusters[0]' long-siteconfig.yaml  | wc -l
22
# total n. lines siteconfig
> cat long-siteconfig.yaml | wc -l
4391

```
Try:
```
> kustomize build ./ --enable-alpha-plugins                                                          
# Warning: 'bases' is deprecated. Please use 'resources' instead. Run 'kustomize edit fix' to update your Kustomization automatically.
Error: failure in plugin configured via /tmp/kust-plugin-config-105501965; fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long: fork/exec /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/siteconfig/SiteConfig: argument list too long

```

If I reduce the number of clusters to 198,

```bash
> cat long-siteconfig.yaml | grep clusterName | wc -l                                                          
198
> cat long-siteconfig.yaml | wc -l                                                          
4369

```

## Get the patched kustomize
