*Disclaimer: This is not an official Red Hat documentation. Just our experience working on this kind of issues. There are thousands of combinations and cases that we cannot cover on this tutorial. But, it would help you with some tips on you debugging issues creating clusters.*

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
   * We boot the server with a full OS. All the tools are there, it does not need to download a big rootfs, etc
   * If there are network problems, at least, it will not get stuck downloading the whole OS
 * To get a root console. This is our main objective. To use the BMC Virtual Console and get a root console.
 
The combination of a root console, a whole OS installed will help us a lot on debugging. Then, how to fix the different possible problems is not covered here. But you will have the tools to proceed. Usually, you are reading this because of some kind of networking problems. So, with the root console and the OS installed, you can check what is failing about the network.

Next sections will help you how to get both advantages: full ISO and root console. Consider, full ISO as optional. Maybe, in your scenario you cannot boot with a full ISO anyway. 

## Debugging Discovery phase


### Enable the full ISO using the Host Inventory resources

### Enable the full ISO using ZTP GitOps resources


### Enable a root console using the Host Inventory resources

### Enable a root console using the Host Inventory resources