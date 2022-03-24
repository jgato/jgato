# Understanding how ACM works with the Red Hat installer of Openshift

ACM (Advanced Cluster Management) allows to user Kubernetes in order to deploy and manage other Kubernetes cluster. Where the first Kubernetes cluster is called Hub (Management) cluster and the deployed clusters are Spoke (Managed) Clusters.

![](assets/2022-03-24-14-48-11-image.png)

This facilitates the creation/import/deployment of clusters, but also to control the life-cycle of each one. 

To manage the infrastructure and the deployment process there exists different providers that allows you deploy different versions of Kubernetes and providers like AWS, GCP, etc.

In this case, we want to deploy Openshift, instead of just a Kubernetes distribution. Red Hat [Openshift Installer](https://github.com/openshift/installer) allows to deploy Openshift with different providers like AWS, GCP, Azure, Baremetal, etc. 

Hive is an Openshift Operator which, making use of the Openshift Installer, can provision Openshift Installations. But Hive can also works as part of ACM as a new provider: which enables the deployment of Openshift clusters directly from the ACM usual workflows. 

Very summarized, when a new ClusterDeployment Resource (with the cluster specification) came into ACM, Hive creates a Pod with the Openshift Installer and starts the installation about ClusterDeployment info.

But OpenShift Installer is not the only way of install Openshift. The [Assisted Installer](https://cloud.redhat.com/blog/ask-an-openshift-admin-office-hour-the-openshift-assisted-installer) provides a newer and easier way of deploying Openshift. With the limitation of the infrastructure has to exist, and cannot be created by the installer. Actually, it is focused on BareMetal Installations. 

This new methodology is also available through ACM, as an extension of the Hive Operator. The Assisted Service is available as an extension of Hive, taking place instead of Openshift Installer, and providing an Assisted Installer. 

## Hive and Assisted Installer (or Agent Install)

AI as an extension on Hive, and this is installed with an :

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

Which deploys two services: the assisted-service (with all the logic) and the assisted-image-service.

```bash
$> oc -n open-cluster-management get deployment | grep assisted
assisted-image-service                  1/1     1            1           51d
assisted-service                        1/1     1            1           51d
```

The assisted-image-service is in charge of providing an ArchOS live iso which will bill start the installation.  An ArchOS ISO is about 1GB which could create different problems when booting from a BMC's virtual media, or maybe, connectivity would be not good enough between the BMC and the ISO. So, the rootfs is extracted from the ISO, and the mini-ISO (about 100MB) is provided from the assisted-image-service, facilitating BMCs to boot from that ISO. 

Later, when new cluster is created, the assisted-service will customize an iso, for each host, accordingly to the cluster and hosts configuration. 

For example, it will include authorized-keys and NMStateConfig with the network from InfraEnv object.

```yaml
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

So images are customized (and stored) for each host that will be created. Customization about: apikey, networkconfiguartion, pull-secrets,etc. Everything need it during the installation. 

The next step, it is to put these images into the BMC VirtualMedia of each host. And here it comes another component [Metal3](https://metal3.io/) and a new Kubernetes Resource, BareMetalHost.

Here an example of a BaremetalHost example:

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

## Metal3

Metal3: It is in charge of configuring the VirtualMedia with the customized ISO and to boot the server to install the installation. But the BMC will not download the ISO from the assisted-image-service. 

Metal3 creates a kind of pods/caches with the ISOs. The URL to download the ISO from the Assiste-Image-Service is not too appropriate for some BMCS. Something like: "https://assisted-image-service-open-cluster-management.apps.el8k.hpecloud.org/images/3ffa8e57-bb4b-4c01-97c4-a34d5fca6bc0?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiIzZmZhOGU1Ny1iYjRiLTRjMDEtOTdjNC1hMzRkNWZjYTZiYzAifQ.UvGpeBtGMQAuvQ2SHCtuWEPJJm_1KR5RS5Mxe3jvoV07nURo-EEWgsl8l9k-gU1wvZIVS9cuSKz-wHHJc0w&arch=x86_64&type=minimal-iso&version=4.9". 

This urls would not work with some BMCs: url too long, not finishing with .iso or params and api-keys in the url.

So, Metal3 caches shorter and more appropriate URLs.

```bash
oc -n openshift-machine-api get pods -l baremetal.openshift.io/cluster-baremetal-operator=metal3-image-cache
NAME                       READY   STATUS    RESTARTS   AGE
metal3-image-cache-kd4wv   1/1     Running   2          51d
metal3-image-cache-mvn94   1/1     Running   3          51d
metal3-image-cache-rmn4h   1/1     Running   5          51d
```

And finally we have all the pieces.

Metal3 mounts the BMC VirtualMedia with these shorter ISOs, and these bots with the customized mini-iso.

## Booting the servers

The server boots with the mini-iso and the custom configuration. Inmediatly, it downlads the rootfs from the url pointed by the AgentServiceConfig. This image is about 900MB, but now we have an OS in charge and not a BMC. So there are less limitations. 

When the full ISO created and running, the server will try to register itself with the Assisted Service. In order to identify the new server, this will use an api-key, also customized together with the ISO. If the registering is oka, it will download an AI Agent that will start the installation. And you will see in the ACM GUI, how the servers are added to the cluster, and how these progress during the insallation.

During the installation, the communication is always from the Server Agent -> Assisted Service. And the Assisted service is orchestration all the installation, it generates the bootstrap/master/workers images, and makes the full installation. Similar to what it does the Openshift Installer.

Finished the installation:

![](assets/2022-03-24-17-02-38-image.png)

## All the Resources that manage a Cluster Deployment

During the previous sections everything has been summarized and simplified. Here a list of all the Resources to be created.

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