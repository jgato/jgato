# Virtual SNO Plan



This is just a [kcli]() plan that helps you to create an empty virtual machine. I usually use this plan to create virtual machines linked with a simulated BMC using Sushy. In this same repository, there is a tutorial about [how to use Sushy](https://github.com/jgato/jgato/blob/main/random_docs/Sushy_Redfish_BMC.md) to emulate a VM and boot servers with an ISO.

The plan creates a VM with some characteristics to run an SNO (Single Node Openshift):

* 8 CPUs would be enough, but it creates 16, to play with NUMA configurations. Setting 6 CPUs by NUMA node

* 32GB RAM

* It also allows to create two different NUMAS, dividing CPUs and Memory. It also creates one extra NIC for each NUMA

The idea of this (optional) NUMA configuration, it is to later play with different operators that configures the performance of the SNO. But it is not something mandatory, it could be used as a regular VM. 



## Enabling NUMA and pinning CPUs

If you enable this parameter you have to configure some parameters in the plan.



About CPUs, check the CPUS in the host in order to configure this accordingly. In this example:

```bash
$ lscpu  | grep NUMA
NUMA node(s):        1
NUMA node0 CPU(s):   0-47

```

My host only has one NUMA node with 48 CPUS, so I ping/divide the last 16CPUs in the plan (vcpus are linked to the hostcpus)

```yaml
  cpupinning:
   - vcpus: 0-7
     hostcpus: 40-47
   - vcpus: 8-15
     hostcpus: 32-39

```

The 3 created network interfaces, the main one, and the ones added to each NUMA are using an existing baremetal bridge network in the host. So, the three interfaces will use the same network (in my case, with internet access). You would need to configure this accordingly to your networking

```bash
$ kcli list networks
Listing Networks...
+--------------+---------+------------------+------+---------+------+
| Network      |   Type  |       Cidr       | Dhcp |  Domain | Mode |
+--------------+---------+------------------+------+---------+------+
| baremetal    | bridged |  10.19.10.64/26  | N/A  |   N/A   | N/A  |
| default      |  routed | 192.168.122.0/24 | True | default | nat  |
| provisioning | bridged |  fd00:1101::/64  | N/A  |   N/A   | N/A  |
+--------------+---------+------------------+------+---------+------+

```

```
   - name: baremetal
     type: e1000e
     numa: 0
     vfio: true
   - name: baremetal
     numa: 1
     type: e1000e
     vfio: true

```

## Deploying an OS

I usually use this to deploy an SNO, actually, using a ZTP methodology. Or, a Red Hat Assisted Installer to deploy an SNO.

Once this is deployed you can check the configuration:


