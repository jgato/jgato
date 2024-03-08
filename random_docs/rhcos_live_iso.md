# RHCOS Live ISO

This ISO is used by RHACM Assisted Installer to mount virtual media on a baremetal server and to start the installation. Actually, it mounts a Minimal version of the ISO, with the needed information to boot the server, and then, download the rootfs and continue the installation.

Some tricks about it.

## Get the network information from the Minimal Iso

The networking information can be found on:

```yaml
> oc -n sno1 get NMStateConfig sno1.sno.hpecloud.org -o yaml | anonymize_data | bat -l yaml
───────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ STDIN
───────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ apiVersion: agent-install.openshift.io/v1beta1
   2   │ kind: NMStateConfig
   3   │ metadata:
   4   │   annotations:
   5   │     argocd.argoproj.io/sync-wave: "1"
   6   │     kubectl.kubernetes.io/last-applied-configuration: |
   7   │       {"apiVersion":"agent-install.openshift.io/v1beta1","kind":"NMStateConfig","metadata":{"annotations":{"argocd.argoproj.io/sync-wave":"1","ran.openshift.io/ztp-gitops-generated":"{}"},"labels":{"app.kubernetes.io/instance":"clusters","nmstat
       │ e-label":"sno1"},"name":"sno1.sno.hpecloud.org","namespace":"sno1"},"spec":{"config":{"dns-resolver":{"config":{"server":["192.168.XXX.YYY"]}},"interfaces":[{"ipv4":{"address":[{"ip":"192.168.XXX.YYY","prefix-length":26}],"enabled":true},"ipv6":
       │ {"enabled":false},"name":"eno3","state":"up","type":"ethernet"}],"routes":{"config":[{"destination":"192.168.XXX.YYY/0","next-hop-address":"192.168.XXX.YYY","next-hop-interface":"eno3"}]}},"interfaces":[{"macAddress":"XX:YY:ZZ:AA:BB:CC","name":"
       │ eno3"}]}}
   8   │     ran.openshift.io/ztp-gitops-generated: '{}'
   9   │   creationTimestamp: "2024-03-06T17:05:18Z"
  10   │   generation: 1
  11   │   labels:
  12   │     app.kubernetes.io/instance: clusters
  13   │     nmstate-label: sno1
  14   │   name: sno1.sno.hpecloud.org
  15   │   namespace: sno1
  16   │   resourceVersion: "130964021"
  17   │   uid: f15201eb-dfc5-4ba2-bddb-ef503e13528e
  18   │ spec:
  19   │   config:
  20   │     dns-resolver:
  21   │       config:
  22   │         server:
  23   │         - 192.168.XXX.YYY
  24   │     interfaces:
  25   │     - ipv4:
  26   │         address:
  27   │         - ip: 192.168.XXX.YYY
  28   │           prefix-length: 26
  29   │         enabled: true
  30   │       ipv6:
  31   │         enabled: false
  32   │       name: eno3
  33   │       state: up
  34   │       type: ethernet
  35   │     routes:
  36   │       config:
  37   │       - destination: 192.168.XXX.YYY/0
  38   │         next-hop-address: 192.168.XXX.YYY
  39   │         next-hop-interface: eno3
  40   │   interfaces:
  41   │   - macAddress: XX:YY:ZZ:AA:BB:CC
  42   │     name: eno3
───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```

This information is included in the isoDownloadUrl on the infraenv:

```bash
> oc -n sno1 get infraenv sno1 -o yaml  | yq '.status.isoDownloadURL'
https://assisted-image-service-open-cluster-management.apps.el8k-1.hpecloud.org/byapikey/eyJhbGciOiJFU.....bZ9ANe-g/4.14/x86_64/minimal.iso
```

We can download and mount it:

```bash
[jgato@provisioner tmp]$ curl -k "https://assisted-image-service-open-cluster-management.apps.el8k-1.hpecloud.org/byapikey/eyJhbGciOiJFU.....bZ9ANe-g/4.14/x86_64/minimal.iso" --output minimal.iso
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  105M  100  105M    0     0   492M      0 --:--:-- --:--:-- --:--:--  492M

[jgato@provisioner tmp]$ mkdir minimal
[jgato@provisioner tmp]$ sudo mount -o loop minimal.iso minimal/
mount: /tmp/minimal: WARNING: device write-protected, mounted read-only.
```

Then the NMStateConfig conte is on `minimal/images/assisted_installer_custom.img`

```bash
[root@provisioner tmp]# zcat minimal/images/assisted_installer_custom.img
07070100000001000041ED0000000000000000000000000000000000000000000000000000000000000000000000000000000500000000/etc07070100000002000041ED0000000000000000000000000000000000000000000000000000000000000000000000000000000E00000000/etc/assisted07070100000003000041ED0000000000000000000000000000000000000000000000000000000000000000000000000000001600000000/etc/assisted/network07070100000004000041ED0000000000000000000000000000000000000000000000000000000000000000000000000000001C00000000/etc/assisted/network/host007070100000005000081800000000000000000000000010000000000000154000000000000000000000000000000000000002E00000000/etc/assisted/network/host0/eno3.nmconnection[connection]
autoconnect=true
autoconnect-slaves=-1
id=eno3
interface-name=eno3
type=802-3-ethernet
uuid=e1118a10-5096-5835-8828-fa7b580d282a
autoconnect-priority=1

[ipv4]

<REDACTED>

```
