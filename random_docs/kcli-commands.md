# Some useful commands

## Create an empty VM

```bash
kcli create vm -P uefi_legacy=true -P start=false -P nets=[default] \
    -P memory=8192 -P numcpus=2 -i centos7 agent1
```

## Create a VM with an downloaded iso

```bash
kcli download image centos7
kcli create vm -P uefi_legacy=true -P start=false -P nets=[default] \
    -P memory=8192 -P numcpus=2 -i centos7 agent1
```

## Create a VM to boot an SNO

```bash
kcli create vm SNO_vm -P memory=30720 -P numcpus=8 \
 -P disks=['{"size": 120}']  \
 -P http_proxy=http://192.168.207.9:3128 \
 -P iso=/tmp/discovery_image_test-sno-local-jgato.iso
```

## Wipe-out disks

List disks

```bash
$ kcli list disk
Listing disks...
+----------------------------------------------------------------------------------------+----------------------+----------------------------------------------------------------------------------------------------------------+
| Name                                                                                   |         Pool         |                                                      Path                                                      |
+----------------------------------------------------------------------------------------+----------------------+----------------------------------------------------------------------------------------------------------------+
| CentOS-7-x86_64-GenericCloud.qcow2                                                     |       default        |                           /var/lib/libvirt/images/CentOS-7-x86_64-GenericCloud.qcow2                           |
| CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2                                 |       default        |                 /var/lib/libvirt/images/CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2                 |
|                                                  | el8k-qjtkz-bootstrap |                /var/lib/libvirt/openshift-images/el8k-qjtkz-bootstrap/el8k-qjtkz-bootstrap.ign                 |
| focal-server-cloudimg-amd64.img                                                        |       default        |                            /var/lib/libvirt/images/focal-server-cloudimg-amd64.img                             |
| node2_0.img                                                                            |      home-jgato      |                                  /home/jgato/libvirt/pool/images/node2_0.img                                   |
| sno7_0.img                                                                             |      home-jgato      |                                   /home/jgato/libvirt/pool/images/sno7_0.img                                   |
| sno8_0.img                                                                             |      home-jgato      |                                   /home/jgato/libvirt/pool/images/sno8_0.img                                   |
+----------------------------------------------------------------------------------------+----------------------+----------------------------------------------------------------------------------------------------------------+
```

Delete the disk used by the VM

```bash
$ kcli delete disk sno7_1.img
```

and recreate it:

```bash
$ kcli create disk -s 250 sno7
```

## Moving disks

Useful if you run out of space in your disks.

Stop the VM

```bash
[jgato@provisioner ~]$ kcli stop vm sno7
Stopping vm sno7 in local...
sno7 stopped

```

Get the disk from your VM:

```bash
[jgato@provisioner ~]$ kcli info vm sno7
name: sno7
id: 64fee173-6d1b-43bc-ae9d-80b72ee57e6a
creationdate: 07-10-2022 10:07
status: down
autostart: False
plan: sno7
profile: kvirt
cpus: 16
memory: 32768
net interface: eth0 mac: 52:54:00:ff:b8:57 net: baremetal type: bridge
diskname: vda disksize: 250GB diskformat: virtio type: qcow2 path: /var/lib/libvirt/images/sno7_0.img

```

In principle we the disk, but you can move also the iso.

```bash
[jgato@provisioner ~]$ sudo mv /var/lib/libvirt/images/sno7_0.img libvirt/pool/images/
[sudo] password for jgato: 

```

I move it to my '/home' because I have more room there.

Now edit the VM to change the disk location:

```bash
[jgato@provisioner ~]$ sudo virsh edit sno7
...
...
 <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/home/jgato/libvirt/pool/images/sno7_0.img'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </disk>
...
```

Start the VM again.



## Create different configuration clients

By default kcli will access your local libvirt environment. But it could be used to access other remote environments.

Edit '~/.kcli/config'

```yaml
local:
  type: kvm

el8000-2-provisioner:
  host: 10.19.10.71
  pool: default
  protocol: ssh
  tunnel: true
  type: kvm
  user: jgato
```

now you can:

```bash
> kcli list vms
+--------------+--------+----+--------------------------------------------------------+-------+---------+
|     Name     | Status | Ip |                         Source                         |  Plan | Profile |
+--------------+--------+----+--------------------------------------------------------+-------+---------+
|    agent1    |  down  |    |                                                        | kvirt |  kvirt  |
|   bifrost    |  down  |    | CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2 | kvirt | centos8 |
|   bifrost2   |  down  |    | CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2 | kvirt | centos8 |
|   minikube   |  down  |    |                                                        |       |         |
| minikube-m02 |  down  |    |                                                        |       |         |
| minikube-m03 |  down  |    |                                                        |       |         |
|   support    |  down  |    | CentOS-8-GenericCloud-8.4.2105-20210603.0.x86_64.qcow2 | kvirt | centos8 |
+--------------+--------+----+--------------------------------------------------------+-------+---------+
```

and using a remote client:

```bash
> kcli -C el8000-2-provisioner list vms
+-------+--------+----------------------------------+--------+----------------+---------+
|  Name | Status |                Ip                | Source |      Plan      | Profile |
+-------+--------+----------------------------------+--------+----------------+---------+
| node2 |   up   | 2620:52:0:1351:67c2:adb3:cfd6:32 |        |     node2      |  kvirt  |
|  sno7 |   up   | 2620:52:0:1351:67c2:adb3:cfd6:81 |        | sno7-ipv6-numa |  kvirt  |
|  sno8 |   up   | 2620:52:0:1351:67c2:adb3:cfd6:89 |        | sno8-ipv6-numa |  kvirt  |
+-------+--------+----------------------------------+--------+----------------+---------+
```

The configuration would include much more options, like the default network, OS, etc:

```yaml
> cat ~/.kcli/config.yml 
default:
  autostart: false
  cache: false
  client: local
  cloudinit: true
  cpuhotplug: false
  cpumodel: host-model
  diskinterface: virtio
  disks:
  - default: true
    size: 10
  disksize: 10
  diskthin: true
  enableroot: true
...
...
```

## Remote access to a console

You can use a remote client to access a remote console by:

```bash
> kcli -C el8000-2-provisioner console sno7
```

## Create a Plan with IPv4/IPv6 interfaces

First we create the plan, with some params, into a file:

```yaml
parameters:
 dual: false

sno3:
 image: centos8stream
 memory: 16384
 numcpus: 8
{% if dual %}
 cmds:
 - dhclient eth0
{% endif %}
 nets:
  - name: baremetal
    ip: 2620:52:0:1351:67c2:adb3:cfd6:f39c
    mask: 64
    gateway: 2620:52:0:1351::1
    dns: 2620:52:0:1351:67c2:adb3:cfd6:f31b
```

and the we use the plan:

```bash
kcli create plan -f sno-plan.yaml sno2-ipv6
```

or with dual stack:

```bash
kcli create plan -f sno-plan.yaml -P dual=true sno2-ipv6
```

## Rendering a plan before applying

With the previous plan file:

```
kcli render -f sno2-plan.yaml
```