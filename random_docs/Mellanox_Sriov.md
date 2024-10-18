# Mellanox issue with secureboot and the Sriov Operator

Openshift 4.16 cluster with Sriov Operator configuring all the Sriov VFs:

```bash
$ oc -n openshift-sriov-network-operator get sriovnetworknodestates.sriovnetwork.openshift.io -n openshift-sriov-network-operator 
NAME                            SYNC STATUS   DESIRED SYNC STATE   CURRENT SYNC STATE   AGE
htworker01.core.e5gc.bos2.lab   Succeeded     Idle                 Idle                 20d
htworker02.core.e5gc.bos2.lab   Succeeded     Idle                 Idle                 20d
htworker03.core.e5gc.bos2.lab   Succeeded     Idle                 Idle                 20d

```

and detected and configured VFs:

```yaml
$> oc -n openshift-sriov-network-operator get sriovnetworknodestates.sriovnetwork.openshift.io -n openshift-sriov-network-operator  htworker03.core.e5gc.bos2.lab -o yaml

apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodeState
metadata:
...
  name: htworker03.core.e5gc.bos2.lab
  namespace: openshift-sriov-network-operator
...
spec:
  interfaces:
  - mtu: 3040
    name: ens2f0
    numVfs: 4
    pciAddress: 0000:0d:00.0
    vfGroups:
    - deviceType: netdevice
      isRdma: true
      mtu: 3040
      policyName: sriovleftdpdkmellanox
      resourceName: sriovleftdpdkmellanox
      vfRange: 0-3
...
...
status:
  interfaces:
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: 7a:30:40:84:aa:d7
      mtu: 3040
      name: ens2f0v0
      pciAddress: 0000:0d:00.2
      vendor: 15b3
      vfID: 0
    - deviceID: "1018"
      driver: mlx5_core
      mac: 5e:10:50:b8:87:66
      mtu: 3040
      name: ens2f0v1
      pciAddress: 0000:0d:00.3
      vendor: 15b3
      vfID: 1
...

```

Actually, this VF resources are exported and available to Openshift cluster:

```yaml
$ oc get nodes htworker03.core.e5gc.bos2.lab -o yaml
apiVersion: v1
kind: Node
metadata:
  name: htworker03.core.e5gc.bos2.lab

spec:
  providerID: baremetalhost:///openshift-machine-api/htworker03.core.e5gc.bos2.lab/8025f381-e1f0-4102-8059-f00a6cce7c2f
  taints:
  - effect: NoSchedule
    key: high-throughput
    value: "true"
status:
...
...
  allocatable:
    cpu: "124"
    ephemeral-storage: "862123650053"
    hugepages-1Gi: 12Gi
    hugepages-2Mi: "0"
    memory: 1034734232Ki
    openshift.io/sriovleftdpdkmellanox: "8"
    openshift.io/sriovrightdpdkmellanox: "8"
    pods: "250"
  capacity:
    cpu: "128"
    ephemeral-storage: 936629116Ki
    hugepages-1Gi: 12Gi
    hugepages-2Mi: "0"
    memory: 1056320152Ki
    openshift.io/sriovleftdpdkmellanox: "8"
    openshift.io/sriovrightdpdkmellanox: "8"
    pods: "250"

...
...

```

SecureBoot is disabled, so the Sriov Operator can work perfectly configuring Mellanox cards.

```bash
$ oc debug node/htworker03.core.e5gc.bos2.lab
Starting pod/htworker03coree5gcbos2lab-debug-pn2l6 ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.82.73
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# mokutil --sb-state
SecureBoot disabled

```
But, if we enable it and restart the server:

```bash
$ oc debug node/htworker03.core.e5gc.bos2.lab
Starting pod/htworker03coree5gcbos2lab-debug-4kr2h ...
To use host binaries, run `chroot /host`
chroot /host
Pod IP: 192.168.82.73
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1#  mokutil --sb-state
SecureBoot enabled
```

looking into the node yaml:

```yaml
status:
  addresses:
  - address: 192.168.82.73
    type: InternalIP
  - address: htworker03.core.e5gc.bos2.lab
    type: Hostname
  allocatable:
    cpu: "124"
    ephemeral-storage: "862123650053"
    hugepages-1Gi: 12Gi
    hugepages-2Mi: "0"
    memory: 1034734252Ki
    openshift.io/sriovleftdpdkmellanox: "0"
    openshift.io/sriovrightdpdkmellanox: "0"
    pods: "250"
```

The Sriov VFs are not configured. 
And the operator shows:

```bash
$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState
NAME                            SYNC STATUS   DESIRED SYNC STATE   CURRENT SYNC STATE   AGE
htworker01.core.e5gc.bos2.lab   Succeeded     Idle                 Idle                 20d
htworker02.core.e5gc.bos2.lab   Succeeded     Idle                 Idle                 20d
htworker03.core.e5gc.bos2.lab   InProgress    Idle                 Idle                 20d

$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState htworker03.core.e5gc.bos2.lab -o jsonpath='{.status.lastSyncError}{"\n"}'
mellanox device detected when in lockdown mode

$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState htworker03.core.e5gc.bos2.lab -o jsonpath='{.status.syncStatus}{"\n"}'
Failed

```

So, we [hit this know bug](https://access.redhat.com/solutions/7073932). SRIOV Operator, with the Mellanox plugin and SecureBoot enabled cannot work together.

The workaround