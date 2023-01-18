# ACM components to deploy Openshift clusters with the Assisted Installer

Red Hat ACM (Advanced Cluster Management) allows Openshift/Kubernetes to deploy and manage other Kubernetes cluster, and your infrastructure as a pool of resources. ACM lives on a first Openshift/Kubernetes cluster, which  is called Hub (Management) cluster. From there all the deployed clusters are Spoke (Managed) Clusters.

> This document dont focuses on how you create and deploy your spoke clusters using RHACM. There are other [documents about it](https://github.com/jparrill/ztp-the-hard-way).
> 
> This document happens after you know how to create the spoke clusters. How the installation, internally, works. The different components inside RHACM that makes this possible. And some technical details. This would make you easier to dig dive into the platform. It  would also help you solving issues, knowing better in which step and which components are involved.

Red Hat ACM is based on the upstream project [Open Cluster Management](https://open-cluster-management.io/) together with other components/controllers/operators. ACM can be used to deploy different Kubernetes flavors, but here, we will focus on Openshift.  

![](assets/2022-03-24-14-48-11-image.png)

To manage the infrastructure and the deployment process there exists different "installers" that allows you deploy different versions of Kubernetes (in our case Openshift). The installer also cover the infrastructure improvising, with cloud providers like AWS, GCP and on premises, using Baremetal.

Some components installed together with RHACM 

* Hive provides API driven cluster provisioning, reshaping, deprovisioning and configuration at scale. It can be used with different providers to deploy your clusters. For example, it can use the [Openshift Installer](https://github.com/openshift/installer) to deploy clusters on AWS, Google Cloud, etc. In our case, it will make use of the [Assisted Installer service](https://github.com/openshift/assisted-service/), another alternative to deploy Openshift.

* Assisted Installer, which is the service that deploys the cluster based on an service living in your Hub cluster. It is a controller monitoring some resources that define a cluster creation. More about these resources [above](#Custom-resources-involved-in-the-process).
  
  Assisted Installer cannot create the Infrastructure and it is only limited to BareMetal installations.

* BareMetalOpertor/Metal3: The BMO is in charge of managing a new CRD called BareMetalHosts. It contains information about the BareMetal servers and its BMCs. So, it can inspect the hardware, boot/switchoff, provision boot images, etc.

The installation process will start, once you have created the needed [CRs](#Custom-resources-involved-in-the-process). In this picture, we can see a high level diagram about the different components interactions:

![](assets/2023-01-18-10-36-48-RHACM%20Components.png)

In the following sections, we will follow the installation process and the different components roles and interactions, in details. 

## Hive and Assisted Installer

Assisted Installer is implemented as an extension on Hive, and this is installed creating an 'AgentServiceConfig':

```yaml
apiVersion: agent-install.openshift.io/v1beta1                                    
kind: AgentServiceConfig                                                          
metadata:                                                                      
 name: agent                                                                   
spec:                                                                          
  databaseStorage:                                                             
    accessModes:                                                               
    - ReadWriteOnce                                                            
    resources:                                                                 
      requests:                                                                
        storage: 10Gi                                                          
  filesystemStorage:                                                           
    accessModes:                                                               
    - ReadWriteOnce                                                            
    resources:                                                                 
      requests:                                                                
        storage: 20Gi                                                          
  osImages:                                                                    
    - openshiftVersion: "4.9"                                                  
      version: "4.9.0"                                                                                                                                                                                                                                        
      url: "https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.9/4.9.0/rhcos-4.9.0-x86_64-live.x86_64.iso"
      rootFSUrl: "https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.9/4.9.0/rhcos-live-rootfs.x86_64.img"
      cpuArchitecture: "x86_64"  
```

The AgentServiceConfig contains a list of available 'openshiftVersions' and the corresponding images. The version in the ISOs, here, are the version of the boot installation ISO.  We can consider this as the version of the installer, and later, we will select the OCP version.  

The AgentServiConfig deploys two services: the assisted-service (with all the logic) and the assisted-image-service.

```bash
$> oc -n open-cluster-management get deployment | grep assisted
assisted-image-service                  1/1     1            1           51d
assisted-service                        1/1     1            1           51d
```

The assisted-image-service is in charge of providing a RHCOS live iso, which will boot the server, to starts the Agent that will make the OCP installation. 
A RHCOS ISO is about 1GB which could create different problems when booting from a BMC's virtual media, or maybe, connectivity would be not good enough between the BMC and the ISO. So, the rootfs is extracted from the ISO, and the mini-ISO (about 100MB) is provided from the assisted-image-service, facilitating BMCs to boot from that ISO. 

The assisted-service will customize the ISOs, for each host, accordingly to the cluster and hosts configuration. For example, it will include authorized-keys and NMStateConfig with the network from InfraEnv object.

```yaml
apiVersion: agent-install.openshift.io/v1beta1                                 
kind: InfraEnv                                                                 
metadata:                                                                      
  labels:                                                                      
    agentclusterinstalls.extensions.hive.openshift.io/location: westford    
#   networkType: dhcp                                                           
  name: sno1                                                      
  namespace: sno1                                                   
spec:                                                                          
  clusterRef:                                                                  
    name: sno1                                                    
    namespace: sno1                                                 
  pullSecretRef:                                                               
    name: pull-secret-sno1                                                  
  sshAuthorizedKey: ssh-rsa AAAAB....3aAhtyhe0SPmTMT+MCjT9R7/tw== jgato@redhat.com                                                 
  nmStateConfigLabelSelector:                                                  
    matchLabels:                                                               
      cluster-name: sno1   
```

So images are customized (and stored) for each host that will be created. 

The next step, it is to put these images into the BMC VirtualMedia of each host. And here it comes another component [BareMetalOperator](https://github.com/metal3-io/baremetal-operator) and a new Kubernetes Resource: BareMetalHost.

## BareMetalOperator/Metal3

The BareMetalOperator(BMO) provides the API for the BareMetalHost(BMH) Resource.  Metal3 provides the logic for managing the BareMetal servers. 

```yaml
apiVersion: metal3.io/v1alpha1  
kind: BareMetalHost  
metadata:  
 name: sno1  
 namespace: sno1  
 labels:  
 infraenvs.agent-install.openshift.io: sno1  
 annotations:  
 inspect.metal3.io: disabled  
 bmac.agent-install.openshift.io/role: "master"  
 bmac.agent-install.openshift.io/hostname: sno1  
spec:  
 automatedCleaningMode: disabled  
 bmc:  
 disableCertificateVerification: True  
 address: redfish-virtualmedia://[IP]/redfish/v1/Systems/1  
 credentialsName: sno1-bmc-secret  
 bootMACAddress: 94:40:c9:1f:bf:64  
 online: true
```

Metal3: It is in charge of configuring the VirtualMedia with the customized ISO and to boot the server to start the installation. 
In the previous section we have seen how the Assisted-Image-Service provides the different ISOs. But the BMC of the server will not download the ISO from the assisted-image-service. 

Metal3 creates a kind of pods/cache with the custom mini-ISOs. The URL to download the ISO from the Assiste-Image-Service is not very appropriate for some BMCS. Assisted-image-service ISOs looks like: 'https://assisted-image-service-open-cluster-management.apps.el8k.hpecloud.org/images/3ffa8e57-bb4b-4c01-97c4-a34d5fca6bc0?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiIzZmZhOGU1Ny1iYjRiLTRjMDEtOTdjNC1hMzRkNWZjYTZiYzAifQ.UvGpeBtGMQAuvQ2SHCtuWEPJJm_1KR5RS5Mxe3jvoV07nURo-EEWgsl8l9k-gU1wvZIVS9cuSKz-wHHJc0w&arch=x86_64&type=minimal-iso&version=4.9'.

This urls would not work with some BMCs: url too long, not finishing with .iso or params and api-keys in the url.

So, Metal3 caches more appropriate, and shorter,  URLs.

```bash
oc -n openshift-machine-api get pods -l baremetal.openshift.io/cluster-baremetal-operator=metal3-image-cache
NAME                       READY   STATUS    RESTARTS   AGE
metal3-image-cache-kd4wv   1/1     Running   2          51d
metal3-image-cache-mvn94   1/1     Running   3          51d
metal3-image-cache-rmn4h   1/1     Running   5          51d
```

And finally we have all the pieces.

Metal3 mounts the BMC VirtualMedia with these shorter ISOs, and these boots with the customized mini-iso.

## Booting the servers and connecting to the Assisted Installer Service

The server boots with the mini-iso and the custom configuration. Immediately, it downloads the rootfs from the url pointed by the AgentServiceConfig. The rootfs plus the mini-iso provides a full OS. Remember: this is not the OCP ISO, this is just the ISO that boots the server for the installation.

The installation ISO contains also an Agent (Assisted Agent) with the information about: how to contact the Assisted Service in the hub cluster (urls, auth, etc).

## Assisted Service and Assisted Agent

That Assisted Agent starts running and calls back to the Assisted Service using https to "register". When that happens, the service creates the Agent Resource (CR) in the hub cluster. Here it is collected all the information provided by the Assisted Agent on each installing host. It also contains all the hardware inventory and the validations about memory, cpus, disks, etc. There is one Agent object, for each installing host.

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: Agent
metadata:
  name: 1988e4b2-c761-4149-bed1-4441fb15812c
  namespace: el8k-ztp-1
status:
  conditions:
  - lastTransitionTime: "2023-01-09T11:22:24Z"
    message: The Spec has been successfully applied
    reason: SyncOK
    status: "True"
    type: SpecSynced

  debugInfo:
    eventsURL: https://assisted-service-open-cluster-management.apps.el8k.hpecloud.org/api/assisted-install/v2/events?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJjNzFkNzYyZC1jNWExLTQ0MmQtYTExZC1jNGI3MTZjY2Y3ZWQifQ.AyZEEC8IfSPk3tvJwORK9upSLL8ZBd1qulAWf4FTgt2jvTf89MRw2otCmXcuqgffpUTME6SfdhH0YyBZRpIsIQ&host_id=1988e4b2-c761-4149-bed1-4441fb15812c
    logsURL: https://assisted-service-open-cluster-management.apps.el8k.hpecloud.org/api/assisted-install/v2/clusters/d4d6386f-27e6-4792-83d3-41c9664e6c59/logs?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbHVzdGVyX2lkIjoiZDRkNjM4NmYtMjdlNi00NzkyLTgzZDMtNDFjOTY2NGU2YzU5In0.O7a-vlu-LZetxSX-ENRtSShkqJITb8ZK00zzdtqpV6h4_WoTXD4O4BJ_6Uw8kFjiwQ5HE5E-FyGsaIRH1lDWpA&host_id=1988e4b2-c761-4149-bed1-4441fb15812c&logs_type=host
    state: added-to-existing-cluster
    stateInfo: Host has rebooted and no further updates will be posted. Please check
      console for progress and to possibly approve pending CSRs
  inventory:
    bmcAddress: 0.0.0.0
    bmcV6Address: ::/0
    boot:
      currentBootMode: uefi
    cpu:
      architecture: x86_64
      clockMegahertz: 2394
      count: 16
      flags:
      - fpu
      - vme
<REDACTED>
      - pku
      - ospke
      - avx512_vnni
      - md_clear
      - arch_capabilities
      modelName: Intel Xeon Processor (Cascadelake)
    disks:
    - byPath: /dev/disk/by-path/pci-0000:00:1f.2-ata-1
      driveType: ODD
      hctl: "0:0:0:0"
<REDACTED>
    hostname: localhost
    interfaces:
    - flags:
      - up
      - broadcast
      - multicast
      hasCarrier: true
      ipV4Addresses:
      - 10.19.10.111/26
      ipV6Addresses: []
      macAddress: 52:54:00:ff:b8:58
      mtu: 1500
      name: enp1s0
      product: "0x0001"
      speedMbps: -1
      vendor: "0x1af4"
    memory:
      physicalBytes: 34359738368
      usableBytes: 33691697152
    systemVendor:
      manufacturer: Red Hat
      productName: KVM
      virtual: true
 <REDACTED>
  progress:
    currentStage: Done
    installationPercentage: 100
    progressStages:
    - Starting installation
    - Installing
    - Writing image to disk
    - Waiting for control plane
    - Rebooting
    stageStartTime: "2023-01-09T11:25:57Z"
    stageUpdateTime: "2023-01-09T11:25:57Z"
  role: worker
  validationsInfo:
    hardware:
    - id: has-inventory
      message: Valid inventory exists for the host
      status: success
<REDACTED>
      status: success
    - id: compatible-with-cluster-platform
      message: Host is compatible with cluster platform baremetal
      status: success
    network:
    - id: connected
      message: Host is connected
      status: success
<REDACTED>
      message: Host has been configured with at least one default route.
      status: success
    - id: api-domain-name-resolved-correctly
      message: Domain name resolution is not required (managed networking)
      status: success
    - id: api-int-domain-name-resolved-correctly
      message: Domain name resolution is not required (managed networking)
      status: success
    - id: apps-domain-name-resolved-correctly
      message: Domain name resolution is not required (managed networking)
      status: success
    - id: dns-wildcard-not-configured
      message: DNS wildcard check is not required for day2
      status: success
    - id: non-overlapping-subnets
      message: Host subnets are not overlapping
      status: success

      status: success
```

During the installation, the communication is always from the Assisted Agent  -> Assisted Service. This is done periodically (1 minute), 

For the installation, the Agent will download the OCP ISO. The  OCP versions to install in your system, depends on the available 'clusterImageSet':

```bash
> oc get clusterimagesets.hive.openshift.io 
NAME                       RELEASE
img4.10.10-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.10-x86_64
img4.10.11-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.11-x86_64
img4.10.12-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.12-x86_64
img4.10.13-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.13-x86_64
img4.10.14-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.14-x86_64
img4.10.15-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.15-x86_64
img4.10.16-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.16-x86_64
img4.10.17-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.17-x86_64
img4.10.18-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.18-x86_64
img4.10.20-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.20-x86_64
img4.10.21-x86-64-appsub   quay.io/openshift-release-dev/ocp-release:4.10.21-x86_64
...
...
```

More about [clusterImageSet here](https://cloud.redhat.com/blog/red-hat-advanced-cluster-management-and-clusterimagesets). 

Which OCP version will be installed? The 'AgentClusterInstall' resource, created on the Hub, contains all the information about the cluster. Including which OCP ISO to use.

```bash
> oc -n el8k-ztp-1 get aci el8k-ztp-1 -o jsonpath={.spec.imageSetRef}
{"name":"img4.10.42-x86-64-appsub"}
```

When the Agent starts the installation, the Assisted Service will take the control on the different BMH resources on the Hub cluster. Remember that BMH resources contained the BMC information, that was used to boot the installation ISO. These BMHs were managed by the BMO. This is done to avoid conflicts or interference between the two controllers:

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    baremetalhost.metal3.io/detached: assisted-service-controller
```

This 'baremetalhost.metal3.io/detached' indicates that now, the BMH resource will take the responsibility on this resources, and the BMO will no longer be in charge of any reconciliation.


Once finished the installation, the Assisted Agent which does the installation on each host, finishes it work. It was running as a service, that has finished, and it will not do anything else. From that moment, there is no more communication between the Assisted Installer Service and the Agent.


![](assets/2022-03-24-17-02-38-image.png)

## Custom resources involved in the process



| Custom Resource       | Description                                                                                                                                                                  |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Namespace             | Namespace for the managed single node cluster.                                                                                                                               |
| BMCSecret CR          | Credentials for the host BMC.                                                                                                                                                |
| Image Pull Secret CR  | Pull secret for the disconnected registry.                                                                                                                                   |
| AgentClusterInstall   | Specifies the single node cluster’s configuration such as networking, number of supervisor (control plane) nodes, and so on.                                                 |
| ClusterDeployment     | Defines the cluster name, domain, and other details.                                                                                                                         |
| KlusterletAddonConfig | Manages installation and termination of add-ons on the ManagedCluster for ACM.                                                                                               |
| ManagedCluster        | Describes the managed cluster for ACM.                                                                                                                                       |
| InfraEnv              | Describes the installation ISO to be mounted on the <br>destination node that the assisted installer service creates. This is the final step of the manifest creation phase. |
| BareMetalHost         | Describes the details of the bare-metal host, including BMC and credentials details.                                                                                         |

# ZTP and platform integration of the Spoke cluster

When we install the cluster, the AI will include some configurations and integration points depending on the platform. Platforms like Baremetal, VSphere, None.

For example: when using Baremetal, UserManagedNetorking is False, and you dont need to configure the DNS. Also, the cluster is installed with the [BMO](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10/html/scalability_and_performance/managing-bare-metal-hosts) and MachineAPI. This integration would allow you, after installation, to manage the infrastructure (managing hosts, scaling up-down, etc).

In case of deploying clusters with ZTP, you will never manage the cluster infrastructure with BMO and MachineAPI. You will scale up-down using your ZTP SiteConfig, which is the only point of truth. You will create a new host on your SiteConfig, that will create the BMH object on the Hub cluster, that will be managed by the BMO, the customized ISOS will be created, and finally the Assisted Installer will do its work. 

In the time or writing this article, neither ZTP4.10, nor ZTP4.11, nor ZTP4.12 sets anything about the platform. So, by default, the AI:

* Install SNOs as None platform.

* Install Standard/Compact clusters as Baremetal platform, including the BMO.

The [AI feature of configuring the platform]([[release-ocm-2.6] MGMT-12317: Add capabilities entries to install-config by openshift-cherrypick-robot · Pull Request #4532 · openshift/assisted-service · GitHub](https://github.com/openshift/assisted-service/pull/4532)) with Kubernetes API (AgentClusterInstall) is only allowed on ACM2.6 and above.

So, installation platform is Baremetal, but, would make sense to dont need to install the BMO. A good idea from ZTP would be to select the platform as Baremetal but not requiring BMO. This requires also a new [feature to disable capabilities](https://github.com/openshift/assisted-service/pull/4532), also ACM2.6.

The cluster capabilities from ACM depends on the OCP deployed clusters. This is also a recent feature:

* In 4.11, you can use capabilities to remove the baremetal, marketplace, and samples operator.

* In 4.12 these are the capabilities that can be removed:  
  
      - CSISnapshot
      - Console
      - Insights
      - Storage
      - baremetal
      - marketplace
      - openshift-samples