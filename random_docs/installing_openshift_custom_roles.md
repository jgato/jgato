# Deploying Openshift with custom roles on nodes

Above Openshfit 4.14, using an ACM Hub and Assisted installer, you can deploy Openshift clusters setting roles on each nodes on installation time. Usually, you can only assign Master or Workers, but now, it is possible to add other roles.

The interesting thing on this is, that nodes are created directly with the configuration on that role. Here, it is important to understand how configurations works on Openshift and RHCOS (unmutable OS) with the MachineConfig Operator. But this is not covered on this article. 

Here, we will just check this new feature, inspecting a node during its installation. And checking that it fetches the ignition (configuration) from the custom role, instead of the usual worker or master.

Other important thing here, because the configuration (regarding the role) is applied on day-0, you save the usual reboot that happens when you change roles on day-2 operations. This is a ver important feature to consider.

In this case, we will take a worker, that is also deployed with the role 'ht'. The inspection is done after the first reboot, when RHCOS has been writing and it fetches the configuration, in this case, for 'ht'.

## Advanced details

How the changing roles on day-0 work? How the configurations are applied during the cluster installation depending on the role?

We have pointed out that assigning roles on day-0 is based on a new Assisted Installer feature. Because we apply the configuration on day-0, we save the usual reboot that happens when you change the role (MCO rendering and applying a new configuration). 

During the installation, nodes download the ignition file corresponding to their final role, even if they join the cluster (initially) as a worker/master, because this is the only way you can join a Kubernetes cluster. How is the process:
The node start their installation as usual, actually, as if it were a worker. Step 1 on our root document. 

When the node has its first reboot, it will fetch an ignition file with its configuration. At this moment, it fetches the ignition file for the new role (in our previous example, as ht role):

```bash
[root@worker-0 core]# journalctl | grep -i "fetched"
Jan 04 11:11:44 localhost.localdomain ignition[927]: fetched referenced config at https://10.19.10.80:22623/config/ht with SHA512: ec99b34e6e04e05bc0d8675391c1df077c85ef141c01a162708854eaedd6f345f3b8278815e53df1c1dbee5f9f745269adecaaad221f026955291128dad0688c
Jan 04 11:11:44 localhost.localdomain ignition[927]: fetched base config from "system"
Jan 04 11:11:44 localhost.localdomain ignition[927]: fetched user config from "system"
Jan 04 11:11:44 localhost.localdomain ignition[927]: fetched base config from "system"
Jan 04 11:11:44 localhost.localdomain ignition[927]: fetched user config from "system"
Jan 04 11:11:44 localhost.localdomain ignition[927]: fetched referenced user config from "/config/ht"
``` 

In an scenario where you are not using this new Assisted Installer feature, you would see something like fetching `config/worker`. 

In any case, it joins the Kubernetes/Openshift cluster as worker/master. You cannot avoid that on Kubernetes.

When the cluster is working, and the MCO is available, it will detect this node belongs to the MCP of ‘ht’ nodes. The MCO will move the node to the proper MCP and triggers to apply the new configuration. But this configuration is the ‘ht’ ignition that was already fetched. Therefore, it does not need to reboot. 

```bash
$ oc get nodes
NAME                           	STATUS   ROLES                     	master-0.el8k-ztp-2.hpecloud.org   Ready	control-plane,master,worker   master-1.el8k-ztp-2.hpecloud.org   Ready	control-plane,master,worker   master-2.el8k-ztp-2.hpecloud.org   Ready	control-plane,master,worker   worker-0.el8k-ztp-2.hpecloud.org   Ready	ht,worker
```

So, it is like: joining first as a worker but already with the ‘ht’ configuration. When the role changes, there are no changes to be applied. And you save a reboot.

```bash
$ oc debug node/worker-0.el8k-ztp-2.hpecloud.org

Starting pod/worker-0el8k-ztp-2hpecloudorg-debug ...
To use host binaries, run `chroot /host`
chroot /host
last reboot
Pod IP: 10.19.10.110
If you don't see a command prompt, try pressing enter.
sh-5.1# last reboot
reboot   system boot  5.14.0-284.30.1. Thu Jan  4 11:13   still running
reboot   system boot  5.14.0-284.25.1. Thu Jan  4 11:11 - 11:13  (00:01)

```

The two only reboots we see, are the unavoidable ones that happens in any installation. There were no extra reboots that will happen if you do the role assignment on day-2


To make the AssistedInstaller use this feature, the BMH object needs to be annotated with the new label (in this case a role) of the node:

```yaml
metadata:
  annotations:
<REDACTED>
    baremetalhost.metal3.io/detached: assisted-service-controller
    bmac.agent-install.openshift.io.node-label.node-role.kubernetes.io/ht: "" 
<REDACTED>
```

But this will only add the label to the node. If you want the node to pull the ignition from the ht MCP, from the very beginning, you have to indicate the AssistedInstaller in the following way:

```yaml
metadata:
  annotations:
<REDACTED>
    baremetalhost.metal3.io/detached: assisted-service-controller
    bmac.agent-install.openshift.io.node-label.node-role.kubernetes.io/ht: "" 
    bmac.agent-install.openshift.io/machine-config-pool: ht
<REDACTED>
``````

Now, when the installation is done, the node is labeled with the role ht, but it is also pointed to use the ht MCP from the very beginning. It will pull, during the installation, the ignition file for an ht node, instead of a worker.


