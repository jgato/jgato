# Enabling disks wipe out during ZTP installations

In this tutorial we will see how to clean disks using ZTP and Siteconfigs. 

The AI, that will deploy the cluster, will delete (no matter what you do) any disk susceptible of been bootable. This is to avoid errors about selecting which one is the boot disk. The AI will make this test with all the disks:

```bash
#> file -s /dev/sdc
/dev/sdc: DOS/MBR boot sector; partition 1 : ID=0xc4, start-CHS (0x0,32,33), end-CHS (0x2fe,75,13), startsector 2048, 209715200 sectors; partition 2 : ID=0x83, start-CHS (0x2fe,75,14), end-CHS (0x3fd,60,47), startsector 209717248, 727920304 sectors
```

If the MBR appears it will be deleted (just a simple 'dd' will be executed) by the AI, and you cannot avoid that. In this tutorial, we will cover how to force the disks cleaning, when the work is not done by AI, because the previous condition is not satisfied.

# Starting point

To start we have an SNO, with some disks and some partitions:

```bash
Usage: gdisk [-l] device_file
sh-4.4# sgdisk -p /dev/sdc
Found valid GPT with corrupt MBR; using GPT and will write new
protective MBR on save.
Disk /dev/sdc: 937637552 sectors, 447.1 GiB
Model: LOGICAL VOLUME  
Sector size (logical/physical): 512/4096 bytes
Disk identifier (GUID): 9F37C78F-64EB-4FDF-B014-9A55F9205630
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 937637518
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048       209717247   100.0 GiB   8300  Linux filesystem
   2       209717248       937637518   347.1 GiB   8300  Linux filesystem

```

Check that disk is not labeled as DOS/MBR, othewise:

```bash
sh-4.4# file -s /dev/sdc
/dev/sdc: data

```

After re-installing SNO, by default, the disk will not be deleted.

# Using extra-manifest

Re-installing using an extra-manifest and RHCOS [Ignition file](https://coreos.github.io/ignition/configuration-v3_2/) to configure the disks during installation. **Important: this is done with a MC that will be used during installation. This disks feature dont work as a MC on Day-1 or Day-2**

Create the MC for the Ignition configuration as an extra-manifestg:

```yaml
$> cat extra-manifests/delete-partition-mc.yaml 
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 98-format-disks-master
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      disks:
        - device: /dev/sdc
          wipeTable: true

```

Other this would be added. Also you would configure/format the partitions here as needed. In this example, we just wipe the partitions table. This should be enough to have a "clean" disk.

After the installation:

```bash
sh-4.4# sgdisk -p /dev/sdc
Creating new GPT entries.
Disk /dev/sdc: 937637552 sectors, 447.1 GiB
Model: LOGICAL VOLUME  
Sector size (logical/physical): 512/4096 bytes
Disk identifier (GUID): 796D7793-59E4-422C-8E7D-F9C9559D2EBE
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 937637518
Partitions will be aligned on 2048-sector boundaries
Total free space is 937637485 sectors (447.1 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name


```

'/dev/sdc' has no partitions.

You can check also this on the rendered MC:

```yaml
$> oc get mc rendered-master-27d33a83f8697c946eb6c876091a1af1 -o yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  annotations:
    machineconfiguration.openshift.io/generated-by-controller-version: 9f2820819fcf1bf5a4b9162adb6fd685779f7cf4
    machineconfiguration.openshift.io/release-image-version: 4.10.30
  creationTimestamp: "2022-11-08T12:22:54Z"
  generation: 1
  name: rendered-master-27d33a83f8697c946eb6c876091a1af1
  ownerReferences:
  - apiVersion: machineconfiguration.openshift.io/v1
    blockOwnerDeletion: true
    controller: true
    kind: MachineConfigPool
    name: master
    uid: d7be7106-ecbc-4aaf-a33f-4b37b018e29f
  resourceVersion: "7041"
  uid: 4f91258b-30ab-4a28-a860-887340f1d20e
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
      - name: core
        sshAuthorizedKeys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCei2UnkBB9g9DPhu4fpMFKmrlhR9UIYYPet61WF3qr6Rp2LkxEhZtbRk6tZjaiVXo/Ff6rsayyoEy86bPCE+4/Kl+3V/KueKW2fgxz/tg1uLiDerWj8+J8KAGJ8TsBAl3cWYYQtHxlwCnyPSmspWB/UegNTd+0cHhkPiTYd6wylgmbBi9MWOAISkXOLWUjsOmKUKiTLkfX2VWqvkk8BH2/blHp0xCSZ2NWifc+VmCvz+M36mj0aRF5dEmfdy+wg7m9wi2/Hq59+NLGBef3kKjBnj0A/K0wFfT0ufi03OkztDOY7Y0xxIkl8Bi/Hof4rDlfKVKA9hcMSo3TY2o0asmTTXUhGZ/FVuZcIZpULOFMXKUR3oKeqnr/dff32IHVwgYb8n8C5zUepWu7tVUKnvxZ0Gwajy1Ru+xjrlROFT+761faJHmG5Ev/EdwKHkXHq5EMHgopyiYV7swJEnFzAUzaiu8DP1FYNJyocRvp6AZpbIlyFoabyq+o2yn2Fhny6gs=
          jgato@provisioner.el8k.hpecloud.org
    storage:
      disks:
      - device: /dev/sdc
        wipeTable: true
      files:
      - contents:

```

# Using ZTP diskPartitions (ZTP 4.11)

Not tested yet.  But an example [here](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/gitops-subscriptions/argocd/example/siteconfig/example-sno.yaml#L52-L58)

```yaml
nodes:
      - hostName: "snonode.sno-worker-0.e2e.bos.redhat.com"
        role: "master"
        rootDeviceHints:
          hctl: "0:2:0:0"
          deviceName: /dev/sda
        ........
        ........
        #Disk /dev/sda: 893.3 GiB, 959119884288 bytes, 1873281024 sectors
        diskPartition:
          - device: /dev/sda
            partitions:
              - mount_point: /var/recovery
                size: 51200
                start: 800000
```

Taking a look to the code it seems you have to format partitions. So not sure if it will work to just wipe a disk, be wipe_table is not a param, as it can be seen [here](https://github.com/openshift-kni/cnf-features-deploy/blob/release-4.11/ztp/source-crs/extra-manifest/image-registry-partition-mc.yaml.tmpl) or [here](https://github.com/openshift-kni/cnf-features-deploy/blob/ff9ff558541ba9fe815e671b93359236946a18d8/ztp/siteconfig-generator/siteConfig/siteConfig.go#L250-L263)