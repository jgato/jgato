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

## Trying the workaround
*Thanks to Yuval Kashtan who created the WA, I am must testing it*

Disable again secureboot.

We take one of the cards/port:

```bash
$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState htworker03.core.e5gc.bos2.lab -o jsonpath='{.spec.interfaces[0]}' | jq
{
  "mtu": 3040,
  "name": "ens2f0",
  "numVfs": 4,
  "pciAddress": "0000:0d:00.0",
  "vfGroups": [
    {
      "deviceType": "netdevice",
      "isRdma": true,
      "mtu": 3040,
      "policyName": "sriovleftdpdkmellanox",
      "resourceName": "sriovleftdpdkmellanox",
      "vfRange": "0-3"
    }
  ]
}

```

Take not on `pciAddress` and `numVfs`.

ssh to this node using the proper `sriov-network-config-daemon-*`, and lets manually configure the VFs, due to SRIOV-Mellanos will fail:

```bash
$ oc -n openshift-sriov-network-operator rsh sriov-network-config-daemon-48dfl
sh-5.1# mstconfig -d 0000:0d:00.0 set SRIOV_EN=1 NUM_OF_VFS=4

Device #1:
----------

Device type:    ConnectX5       
Name:           MCX516A-CCA_Ax  
Description:    ConnectX-5 EN network interface card; 100GbE dual-port QSFP28; PCIe3.0 x16; tall bracket; ROHS R6
Device:         0000:0d:00.0    

Configurations:                                      Next Boot       New
         SRIOV_EN                                    True(1)         True(1)         
         NUM_OF_VFS                                  4               4               

 Apply new Configuration? (y/n) [n] : y
Applying... Done!
-I- Please reboot machine to load new configurations.

```

before rebooting, we disable the Mellanox plugin:

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovOperatorConfig
metadata:
  creationTimestamp: "2024-09-27T13:00:24Z"
  generation: 5
  name: default
  namespace: openshift-sriov-network-operator
  resourceVersion: "62910922"
  uid: 0b01bc34-4d0a-407b-b1bd-76e3f7d9dd3d
spec:
  configDaemonNodeSelector:
    node-role.kubernetes.io/ht100gb: ""
  enableInjector: true
  enableOperatorWebhook: true
  logLevel: 0
  disablePlugins:
  - mellanox

```

and reboot enabling secureboot again.

```bash
$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState htworker03.core.e5gc.bos2.lab -o jsonpath='{.status.syncStatus}{"\n"}'
Succeeded

```

and I see the VFs configured:

```yaml
status:
  interfaces:
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: ae:4e:18:29:0e:ef
      mtu: 3040
      name: ens2f0v0
      pciAddress: 0000:0d:00.2
      vendor: 15b3
      vfID: 0
    - deviceID: "1018"
      driver: mlx5_core
      mac: 56:41:17:99:ab:1b
      mtu: 3040
      name: ens2f0v1
      pciAddress: 0000:0d:00.3
      vendor: 15b3
      vfID: 1
    - deviceID: "1018"
      driver: mlx5_core
      mac: aa:2c:58:5f:00:41
      mtu: 3040
      name: ens2f0v2
      pciAddress: 0000:0d:00.4
      vendor: 15b3
      vfID: 2
    - deviceID: "1018"
      driver: mlx5_core
      mac: d6:93:f2:10:7e:bd
      mtu: 3040
      name: ens2f0v3
      pciAddress: 0000:0d:00.5
      vendor: 15b3
      vfID: 3
    deviceID: "1017"
    driver: mlx5_core
    eSwitchMode: legacy
    linkSpeed: 100000 Mb/s
    linkType: ETH
    mac: b8:3f:d2:e4:ab:76
    mtu: 3040
    name: ens2f0
    numVfs: 4
    pciAddress: 0000:0d:00.0
    totalvfs: 4
    vendor: 15b3
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: ba:0d:db:ab:bf:61
      mtu: 3040
      name: ens2f1v0
      pciAddress: 0000:0d:00.6
      vendor: 15b3
      vfID: 0
    - deviceID: "1018"
      driver: mlx5_core
      mac: 0a:80:42:13:ff:34
      mtu: 3040
      name: ens2f1v1
      pciAddress: 0000:0d:00.7
      vendor: 15b3
      vfID: 1
    - deviceID: "1018"
      driver: mlx5_core
      mac: d2:79:ec:16:25:e6
      mtu: 3040
      name: ens2f1v2
      pciAddress: 0000:0d:01.0
      vendor: 15b3
      vfID: 2
    - deviceID: "1018"
      driver: mlx5_core
      mac: ca:9f:3b:7e:f4:0e
      mtu: 3040
      name: ens2f1v3
      pciAddress: 0000:0d:01.1
      vendor: 15b3
      vfID: 3
    deviceID: "1017"
    driver: mlx5_core
    eSwitchMode: legacy
    linkSpeed: 100000 Mb/s
    linkType: ETH
    mac: b8:3f:d2:e4:ab:77
    mtu: 3040
    name: ens2f1
    numVfs: 4
    pciAddress: 0000:0d:00.1
    totalvfs: 4
    vendor: 15b3
  - deviceID: "1593"
    driver: ice
    eSwitchMode: legacy
    linkSpeed: 25000 Mb/s
    linkType: ETH
    mac: 30:3e:a7:03:15:80
    mtu: 9000
    name: eno12399
    pciAddress: "0000:22:00.0"
    totalvfs: 64
    vendor: "8086"
  - deviceID: "1593"
    driver: ice
    eSwitchMode: legacy
    linkSpeed: -1 Mb/s
    linkType: ETH
    mac: 30:3e:a7:03:15:81
    mtu: 1500
    name: eno12409
    pciAddress: "0000:22:00.1"
    totalvfs: 64
    vendor: "8086"
  - deviceID: "1593"
    driver: ice
    eSwitchMode: legacy
    linkSpeed: 25000 Mb/s
    linkType: ETH
    mac: 30:3e:a7:03:15:80
    mtu: 9000
    name: eno12419
    pciAddress: "0000:22:00.2"
    totalvfs: 64
    vendor: "8086"
  - deviceID: "1593"
    driver: ice
    eSwitchMode: legacy
    linkSpeed: -1 Mb/s
    linkType: ETH
    mac: 30:3e:a7:03:15:83
    mtu: 1500
    name: eno12429
    pciAddress: "0000:22:00.3"
    totalvfs: 64
    vendor: "8086"
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: 5a:3c:ad:77:81:e8
      mtu: 3040
      name: ens7f0v0
      pciAddress: 0000:b5:00.2
      vendor: 15b3
      vfID: 0
    - deviceID: "1018"
      driver: mlx5_core
      mac: 6a:c8:8c:5a:7e:44
      mtu: 3040
      name: ens7f0v1
      pciAddress: 0000:b5:00.3
      vendor: 15b3
      vfID: 1
    - deviceID: "1018"
      driver: mlx5_core
      mac: d2:29:c9:7f:7c:4d
      mtu: 3040
      name: ens7f0v2
      pciAddress: 0000:b5:00.4
      vendor: 15b3
      vfID: 2
    - deviceID: "1018"
      driver: mlx5_core
      mac: 16:49:92:c3:a5:6f
      mtu: 3040
      name: ens7f0v3
      pciAddress: 0000:b5:00.5
      vendor: 15b3
      vfID: 3
    deviceID: "1017"
    driver: mlx5_core
    eSwitchMode: legacy
    linkSpeed: 100000 Mb/s
    linkType: ETH
    mac: b8:3f:d2:b7:35:a6
    mtu: 3040
    name: ens7f0
    numVfs: 4
    pciAddress: 0000:b5:00.0
    totalvfs: 4
    vendor: 15b3
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: 5e:1d:34:41:1d:12
      mtu: 3040
      name: ens7f1v0
      pciAddress: 0000:b5:00.6
      vendor: 15b3
      vfID: 0
    - deviceID: "1018"
      driver: mlx5_core
      mac: 3a:d6:52:69:fb:3c
      mtu: 3040
      name: ens7f1v1
      pciAddress: 0000:b5:00.7
      vendor: 15b3
      vfID: 1
    - deviceID: "1018"
      driver: mlx5_core
      mac: de:5f:8d:45:82:bd
      mtu: 3040
      name: ens7f1v2
      pciAddress: 0000:b5:01.0
      vendor: 15b3
      vfID: 2
    - deviceID: "1018"
      driver: mlx5_core
      mac: c6:d9:5a:cb:19:73
      mtu: 3040
      name: ens7f1v3
      pciAddress: 0000:b5:01.1
      vendor: 15b3
      vfID: 3
    deviceID: "1017"
    driver: mlx5_core
    eSwitchMode: legacy
    linkSpeed: 100000 Mb/s
    linkType: ETH
    mac: b8:3f:d2:b7:35:a7
    mtu: 3040
    name: ens7f1
    numVfs: 4
    pciAddress: 0000:b5:00.1
    totalvfs: 4
    vendor: 15b3


```

There is something I dont understand here. I see configured all the VFs on all the NICs. Not only the one configured manually with `mstconfig`. But, in my scenario, I started with all the VFs in all NICs already configured. Is this saved in the card? and so, when I restart, without the plugin, the configuration is expected to be still there? No matter if I did the configuration with `mstconfig` or Sriov Operator? 
If so, it is only important to do the configuration, then disable the plugin, and then you can restart with SecureBoot? This is my guess. So, we could just:
 * Configure everything as usual with Sriov Operator and ZTP Policies. This is ... saved to the card?
 * When finished, we disable the Plugin, reboot with secureboot.
 * The operator is no longer try to configure (or reconfigure) anything because it is disabled.
 * But it is already configured on the hardware from the first step
 * Secureboot is there, and the Operator is oka, because is not changing anything in the hardware.
 
This means that configuration is done inside the card? My knowledge on Sriov is not so deep. 

I would demonstrate this with:
 * I still have two htworkers with secureboot not enabled
 * There, I have not used `mstconfig`
 * With the Mellanox plugin disabled, I only have to enable secureboot there.
 * Check configuration is oka
 
 **Not tested at all: Create networks and pods using this VFs with the plugin disabled and Secureboot enabled.**


## Trying the workaround second phase

This time will not use the tool `mstconfig`. We will take a worker that was never used `mstconfig` there.

But all the htworker were already configured by the Sriov operator, before enabling Secureboot. That is our **usual scenario**. 

So, htworker02:
```bash
$ oc debug node/htworker02.core.e5gc.bos2.lab
Starting pod/htworker02coree5gcbos2lab-debug-6t4rw ...
To use host binaries, run `chroot /host`
chrootPod IP: 192.168.82.72
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1#  mokutil --sb-state
SecureBoot disabled
sh-5.1# 
exit
sh-5.1# 
exit

Removing debug pod ...
$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState htworker02.core.e5gc.bos2.lab -o jsonpath='{.status.syncStatus}{"\n"}'
Succeeded

```

Reboot and enable Secureboot on worker02.

```bash
$ oc debug node/htworker02.core.e5gc.bos2.lab
Starting pod/htworker02coree5gcbos2lab-debug-hrgwc ...
To use host binaries, run `chroot /host`
chroot /host
Pod IP: 192.168.82.72
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# mokutil --sb-state
SecureBoot enabled
sh-5.1# 
exit
sh-5.1# 
exit

Removing debug pod ...
[jgato@jump ~]$ oc -n openshift-sriov-network-operator get SriovNetworkNodeState htworker02.core.e5gc.bos2.lab -o jsonpath='{.status.syncStatus}{"\n"}'
Succeeded

```

and all the VFs configured, as previously created by the Sriov operator.

```yaml
status:
  interfaces:
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: 42:e3:a8:14:f8:26
      mtu: 3040
      name: ens2f0v0
      pciAddress: 0000:0d:00.2
      vendor: 15b3
      vfID: 0
...
...
    - deviceID: "1018"
      driver: mlx5_core
      mac: 2a:70:5e:49:35:bd
      mtu: 3040
      name: ens2f0v3
      pciAddress: 0000:0d:00.5
      vendor: 15b3
      vfID: 3
    deviceID: "1017"
    driver: mlx5_core
    eSwitchMode: legacy
    linkSpeed: 100000 Mb/s
    linkType: ETH
    mac: b8:3f:d2:e4:ae:5e
    mtu: 3040
    name: ens2f0
    numVfs: 4
    pciAddress: 0000:0d:00.0
    totalvfs: 4
    vendor: 15b3
 ...
 ...
  - Vfs:
    - deviceID: "1018"
      driver: mlx5_core
      mac: c2:3f:df:b1:d0:af
      mtu: 3040
      name: ens7f0v0
      pciAddress: 0000:b5:00.2
      vendor: 15b3
      vfID: 0
    - deviceID: "1018"
      driver: mlx5_core
      mac: aa:ea:9e:38:0e:e6
      mtu: 3040
      name: ens7f0v1
      pciAddress: 0000:b5:00.3
      vendor: 15b3
      vfID: 1
    - deviceID: "1018"
      driver: mlx5_core
      mac: fe:4b:d4:fb:59:ec
      mtu: 3040
      name: ens7f0v2
      pciAddress: 0000:b5:00.4
      vendor: 15b3
      vfID: 2
    - deviceID: "1018"
      driver: mlx5_core
      mac: 82:57:fd:5a:ff:9f
      mtu: 3040
      name: ens7f0v3
      pciAddress: 0000:b5:00.5
      vendor: 15b3
      vfID: 3
    deviceID: "1017"
    driver: mlx5_core
    eSwitchMode: legacy
    linkSpeed: 100000 Mb/s
    linkType: ETH
    mac: b8:3f:d2:e4:af:26
    mtu: 3040
    name: ens7f0
    numVfs: 4
    pciAddress: 0000:b5:00.0
    totalvfs: 4
    vendor: 15b3
 ...
 ...
  syncStatus: Succeeded


```

So, we can configure Sriov as our scenario, and just disable the plugin before enabling secureboot. This will make the VFs to be created correctly.
 
**Not tested at all: Create networks and pods using this VFs with the plugin disabled and Secureboot enabled.**
