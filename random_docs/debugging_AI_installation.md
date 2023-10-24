*Disclaimer: This is not an official Red Hat documentation. Just our experience working on this kind of issues. There are thousands of combinations and cases that we cannot cover on this tutorial. But, it would help you with some tips on you debugging issues creating clusters. The work done here used OCP 4.13, ACM 2.7 and ZTP 4.13*

# Debugging clusters deployment with Openshift Assisted Installer and baremetal servers

Openshift Assisted Installer is an installation method for different Openshift cluster's flavours. The Assisted Installer can be used in different ways, like the [AI SaaS](https://docs.openshift.com/container-platform/4.13/installing/installing_on_prem_assisted/installing-on-prem-assisted.html), with a cloud platform that helps you to manage your cluster's deployment. Or, it can be used on premises, running on your own Management/Hub Openshift cluster.

In this tutorial, we will focus on this last option. When you have your own, on premises, Openshift Managment cluster. The cluster's deployment are done by the [Assisted Installer from the multicluster engine for Kubernetes](https://cloud.redhat.com/blog/multicluster-engine-and-assisted-installer-integration-deploy-and-expand-clusters-easily). This way of deploying cluster is also known as ZTP (Zero Touch Provisioning). 

We will cover  some tips to do debugging during the installation. Depending on if you are using ZTP, or ZTP GitOps. Two similar installation methods that make usage of different CustomResources.

It is important to differentiate when the installation failed. The AI starts the installation booting a RHCOS Discovery ISO, that basically, collects information from each node, contacts the AI and validates the installation. This can be known as the Discovery phase. After that, it happens the first nodes reboot. This time, it will use a RHCOS installed operative system, that will proceed with the real cluster installation. We could say, this is the Installation phase.

# Why debugging is complex

In this kind of installations you usually will make use of **remote baremetal servers**. If something goes wrong, it is difficult to debug:
 * By default, RHCOS creates an unique `core` user without password. So you dont have a kind of `root`user. But can login with an ssh key.
 * If the network configuration fails, and this will be your main headache, you cannot ssh there. 
 * The Discovery ISO, by default, it is a minimal ISO to facilitate booting from it with a BMC. Just after booting, starts downloading the full iso. If the network fails, it wont download it. You will have a minimal ISO, not a fully OS loaded, neither ssh, or other similar tools.
 * You dont have physical access, so you will use a virtual console on the BMC of the baremetal server. This virtual consoles, usually, responds pretty slow.

What are we going to try to do in this tutorial?
 * To do the Discovery phase with a full ISO, instead of the default minimal one. This can be only considered for debugging. Full ISOs are not good friends on BMC and its virtual media. It is not recommended, or allowed, to download 1GB ISO per each server, on a BMC network. This is not an expected behaviour. Some BMCs would not allow that, or maybe the connection is not good enough to download this size and boot with the virtual media. Advantages of that:
   * We boot the server with a full OS. All the tools are there, it does not need to download a big `rootfs`, etc
   * If there are network problems, at least, it will not get stuck downloading the whole OS
 * To get a root console. This is our main objective. To use the BMC Virtual Console and get a root console.
 
The combination of a `root` console, a whole OS installed will help us a lot on debugging. Then, how to fix the different possible problems is not covered here. But you will have the tools to proceed. Usually, you are reading this because of some kind of networking problems. So, with the root console and the OS installed, you can check what is failing about the network.

Next sections will help you how to get both advantages: full ISO and root console. Consider, full ISO as optional. Maybe, in your scenario you cannot boot with a full ISO anyway. 

## Debugging Discovery phase


### Enable the full ISO 

*This is an experimental feature of the Assisted Installer, how to enable it it would change. More info [here](https://github.com/openshift/assisted-service/blob/master/docs/operator.md#specifying-environmental-variables-via-configmap)*

Basically, we enable a feature that allows to enable different parameters of the Assisted Installer from a ConfigMap.

This ConfigMap will enable the Full ISO provisioning. Notice, this will affect any new deployment (disable this after your debugging sessions)

```yaml
apiVersion: v1     
kind: ConfigMap                                                                
metadata:                                                                      
  name: my-assisted-service-config                                             
  namespace: multicluster-engine                                               
data:                                                                          
  ISO_IMAGE_TYPE: full-iso                                                                                            
```

Then enable AI to use this CM:

```bash
$> oc annotate --overwrite AgentServiceConfig agent unsupported.agent-install.openshift.io/assisted-service-configmap=my-assisted-service-config
```

Then, we restart the AI service:

```
> oc rollout restart deployment/assisted-service -n multicluster-engine
deployment.apps/assisted-service restarted

```

After that, you can check BMH (Host Inventory CRs) and how it is using a full iso:

```bash
> oc -n el8k-ztp-1 get bmh master-0.el8k-ztp-1.hpecloud.org -o yaml | grep url

      url: https://assisted-image-service-multicluster-engine.apps.el8k.hpecloud.org/images/ec54f476-6535-4672-b7b2-b06265bb79db?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJlYzU0ZjQ3Ni02NTM1LTQ2NzItYjdiMi1iMDYyNjViYjc5ZGIifQ.7OynkZF4LFUnEVa6ju1d7cSWRIrzqVZBp_tTZLvjFSrmCmjdscVTUixUaSGt9AEKNONFZ3iC0H8MyJsky4fncg&arch=x86_64&type=full-iso&version=4.13 

> curl  -s -k -L -I "https://assisted-image-service-multicluster-engine.apps.el8k.hpecloud.org/images/ec54f476-6535-4672-b7b2-b06265bb79db?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJlYzU0ZjQ3Ni02NTM1LTQ2NzItYjdiMi1iMDYyNjViYjc5ZGIifQ.7OynkZF4LFUnEVa6ju1d7cSWRIrzqVZBp_tTZLvjFSrmCmjdscVTUixUaSGt9AEKNONFZ3iC0H8MyJsky4fncg&arch=x86_64&type=full-iso&version=4.13" | grep content-length

content-length: 1119879168
```

The new ISO is bigger than 1GB, that would cause problems on some BMCs. This is why this is optional.

### Enable a root console using the Host Inventory resources

To enable the `root` console is done by the Infraenv CR. This CR affects to all the hosts in the cluster we are debugging. After enabling it, it will reboot all the hosts, to make them boot again in the Discovery Phase. But this time, all of them will have `root` console on tty9. This is not a big issue, because you will use this feature when something is going wrong creating a cluster. So you will enable it, debug, and when everything is fixed, re-trigger the installation without this option. Dont forget to disable it, or you will have a huge free `root`access. Neither use this on a working/created cluster.

```yaml
> oc -n el8k-ztp-1 edit infraenv el8k-ztp-1

<REDACTED>                                     
spec:                                                                          
<REDACTED>                                     
  kernelArguments:                                                             
  - operation: append                                                          
    value: systemd.debug-shell=1                                               
  - operation: append                                                          
    value: console=tty1              
<REDACTED>                                     
```

`systemd.debug-shell=1` will open the `root` console on tty9. The host will reboot with this kernel param. During booting process of the baremetal server, you can use the virtual console and 'Alt+F9' to switch the console. 

![](assets/root-console.gif)

I tried this with an ILO BMC, other providers would have different behaviour about how you switch ttyX.


`console=tty1` will output all the logs to the tty1. So, you can work on tty9 as `root`not dealing with flood of logs. 



### Enable a root console using ZTP GitOps resources

Using ZTP GitOps resources (Siteconfigs) you cannot enable the `root`console. Or at least, in the way we did it above. 

It should be needed to implement a new param in the Siteconfig definition, that would end-up filling the proper param in the Infraenv CR from the Host Inventory.


## Debugging Installation phase


### Enable a root console using the Host Inventory resources
[ToDo]

### Enable a root console using ZTP GitOps resources

[ToDo]