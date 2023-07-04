# Managing RHCOS packages and repositories

OCP is available through different channels (stable, candidate, fast) and different versions. These scripts will help to find out packages and versions included in each combination

First of all, the script has been created by Robert Sandu (rsandu@redhat.com):

'cat package_by_z.sh'

```bash
  PKG=$1
  RHOCP_RELEASE=$2
  RHCOS_BUILD=$(oc adm release info quay.io/openshift-release-dev/ocp-release:${RHOCP_RELEASE}-x86_64 -o jsonpath='{.displayVersions.machine-os.Version}')
  RHOCP_MINOR=$(echo $RHOCP_RELEASE | awk -F. '{print $1 "." $2}')

  echo "Looking for the $1 package bundled with ${RHOCP_RELEASE}"
  echo "RHEL CoreOS Release Build: ${RHCOS_BUILD}"
  echo "---"
  curl -s https://releases-rhcos-art.apps.ocp-virt.prod.psi.redhat.com/storage/releases/rhcos-${RHOCP_MINOR}/${RHCOS_BUILD}/x86_64/commitmeta.json | jq -r --arg PKG "$PKG" '. ."rpmostree.rpmdb.pkglist" | .[] | select(.[] | contains('\"$PKG\"')) | .[0] +"-"+ .[2]+"-"+.[3]+"."+.[4]'
```

'package_by_channel.sh'

```bash
cat package_by_channel.sh 
  RHOCP_MINOR=${2:-"4.8"}
  RHOCP_CHANNEL=${3:-stable}
  RHOCP_RELEASE=$(curl -sH 'Accept: application/json'  https://api.openshift.com/api/upgrades_info/v1/graph?channel=${RHOCP_CHANNEL}-${RHOCP_MINOR} | jq -r '.nodes[].version' | sort -n -t . -k2,2 -k3,3 | tail -1)
  RHCOS_VERSION=$(oc adm release info quay.io/openshift-release-dev/ocp-release:${RHOCP_RELEASE}-x86_64 -o jsonpath='{.displayVersions.machine-os.Version}')
  PKG=$1

  echo "Looking for $1 in the latest ${RHOCP_RELEASE} of ${RHOCP_CHANNEL}-${RHOCP_MINOR}:"
  echo "RHEL CoreOS Release Build: ${RHCOS_VERSION}"
  echo "---"
  echo "https://releases-rhcos-art.apps.ocp-virt.prod.psi.redhat.com/storage/releases/rhcos-${RHOCP_MINOR}/${RHCOS_VERSION}/x86_64/commitmeta.json"
  curl -s https://releases-rhcos-art.apps.ocp-virt.prod.psi.redhat.com/storage/releases/rhcos-${RHOCP_MINOR}/${RHCOS_VERSION}/x86_64/commitmeta.json | jq -r --arg PKG "$PKG" '. ."rpmostree.rpmdb.pkglist" | .[] | select(.[] | contains('\"$PKG\"')) | .[0] +"-"+ .[2]+"-"+.[3]+"."+.[4]'
```

'get_RHCOS_version.sh`

```bash
#!/bin/bash
if [ -z "$CHANNEL" ]
then
    echo Provide the channel via \'export CHANNEL=stable-4.x\'
    exit 1
fi

rm -f version_maps &> /dev/null

VERSIONS=$(curl -sH 'Accept: application/json'  "https://api.openshift.com/api/upgrades_info/v1/graph?channel=${CHANNEL}" | jq '.nodes[].version' -r)

for VERSION in ${VERSIONS}
do
    RHCOSVERSION="$(oc adm release info "${VERSION}" -o 'jsonpath={.displayVersions.machine-os.Version}')"
    echo "$VERSION - $RHCOSVERSION" | tee -a ./version_maps
done

echo OpenShift to RHCOS version mapping is in 'version_maps'
```

I am just collecting some usage examples.

Mostly you usually need this:

* Find out versions of an specific package 

* Some bug have fixed on a package version, and you have to know in which distribution has been included

## Getting RPM packages versions.

Getting all the packages version from the last OCP version. In this case OCP4.10 stable

```yaml
$> ./package_by_channel.sh kernel 4.10 stable
Looking for kernel in the latest 4.10.32 of stable-4.10:
RHEL CoreOS Release Build: 410.84.202209061703-0
---
https://releases-rhcos-art.apps.ocp-virt.prod.psi.redhat.com/storage/releases/rhcos-4.10/410.84.202209061703-0/x86_64/commitmeta.json
kernel-4.18.0-305.62.1.el8_4.x86_64
kernel-core-4.18.0-305.62.1.el8_4.x86_64
kernel-modules-4.18.0-305.62.1.el8_4.x86_64
kernel-modules-extra-4.18.0-305.62.1.el8_4.x86_64
```

Getting all the packages version from an specific .Z version. In this case OCP4.10.21:

```yaml
$> ./package_by_z.sh NetworkManager 4.10.23
Looking for the NetworkManager package bundled with 4.10.23
RHEL CoreOS Release Build: 410.84.202207140725-0
---
NetworkManager-1.30.0-14.el8_4.x86_64
NetworkManager-cloud-setup-1.30.0-14.el8_4.x86_64
NetworkManager-libnm-1.30.0-14.el8_4.x86_64
NetworkManager-ovs-1.30.0-14.el8_4.x86_64
NetworkManager-team-1.30.0-14.el8_4.x86_64
NetworkManager-tui-1.30.0-14.el8_4.x86_64
```

## Getting the RHCOS version for an OCP version

OCP host operative system will be RHCOS. So, this script will help you to get all the relations between the different versions:

```bash
$> CHANNEL=stable-4.10 ./get_RHCOS_version.sh 
4.9.31 - 49.84.202204251501-0
4.9.0 - 49.84.202110081407-0
4.10.25 - 410.84.202207262020-0
4.10.8 - 410.84.202203290245-0
4.10.32 - 410.84.202209061703-0
4.10.30 - 410.84.202208161501-0
4.9.42 - 49.84.202206302027-0
4.9.37 - 49.84.202205311501-0
...
...
```

First column is the OCP version, and second the corresponding RHCOS version that will be used as the operative system for the host.

 

# Container Images installed in each OCP version

All the OCP versions can be extracted details about modifications and Container Images containing by each version on: 

```bash
https://amd64.ocp.releases.ci.openshift.org/
```

Also with:

```bash
> oc adm release extract --tools quay.io/openshift-release-dev/ocp-release:4.12.22-x86_64 && cat release.txt 
Client tools for OpenShift
--------------------------

These archives contain the client tooling for [OpenShift](https://docs.openshift.com).

To verify the contents of this directory, use the 'gpg' and 'shasum' tools to
ensure the archives you have downloaded match those published from this location.

The openshift-install binary has been preconfigured to install the following release:

---

Name:           4.12.22
Digest:         sha256:ba7956f5c2aae61c8ff3ab1ab2ee7e625db9b1c8964a65339764db79c148e4e6
Created:        2023-06-15T11:13:37Z
OS/Arch:        linux/amd64
Manifests:      647
Metadata files: 1

Pull From: quay.io/openshift-release-dev/ocp-release@sha256:ba7956f5c2aae61c8ff3ab1ab2ee7e625db9b1c8964a65339764db79c148e4e6

Release Metadata:
  Version:  4.12.22
  Upgrades: 4.11.11, 4.11.12, 4.11.13, 4.11.14, 4.11.16, 4.11.17, 4.11.18, 4.11.19, 4.11.20, 4.11.21, 4.11.22, 4.11.23, 4.11.24, 4.11.25, 4.11.26, 4.11.27, 4.11.28, 4.11.29, 4.11.30, 4.11.31, 4.11.32, 4.11.33, 4.11.34, 4.11.35, 4.11.36, 4.11.37, 4.11.38, 4.11.39, 4.11.40, 4.11.41, 4.11.42, 4.11.43, 4.12.0, 4.12.1, 4.12.2, 4.12.3, 4.12.4, 4.12.5, 4.12.6, 4.12.7, 4.12.8, 4.12.9, 4.12.10, 4.12.11, 4.12.12, 4.12.13, 4.12.14, 4.12.15, 4.12.16, 4.12.17, 4.12.18, 4.12.19, 4.12.20, 4.12.21
  Metadata:
    url: https://access.redhat.com/errata/RHSA-2023:3615

Component Versions:
  kubernetes 1.25.10               
  machine-os 412.86.202306132230-0 Red Hat Enterprise Linux CoreOS

Images:
  NAME                                           PULL SPEC
  agent-installer-api-server                     quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:92220ada154f03e41037893bcb49041063f1fecf88143bb8fb8cf6174a7f7b79
  agent-installer-csr-approver                   quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:7167308150d1873c8f77e7a51418cd684af37ec80d244696cf7df01b72bc3f19
  agent-installer-node-agent                     quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:b51bd8e5f1c76604333394e75f4b26c39197dbe421294e9ebf307c4a5fd9d71a
  agent-installer-orchestrator                   quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:6a0597cfa1f50e3532e4d64b1398663c8d433f5dffc46617120a160cf9c28be6
  alibaba-cloud-controller-manager               quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:67d4570cd3ac924fd800ffa9a894c054fa94e407d88f80f4d1f2c34bf7f9cc8f
  alibaba-cloud-csi-driver                       quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:4874cd22a40d83da75341d3e322e76d24439dcebc14ee71775a81815f8120ec0
  alibaba-disk-csi-driver-operator               quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:c9c689a4b338d640d0af3a958c04326b51adcff2d7c7f0a8342fa0f089493a84
  alibaba-machine-controllers                    quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a859217fbc0074f786828ac839eb5b8d15f1f3f2d293c5a019f85e6f0f622346
  apiserver-network-proxy                        quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:11d64bc2998bf24c132de45f60a05e0199b8ef058fc3b0a3fae2c9eeecfbb99c
  aws-cloud-controller-manager                   quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:1aafdfea69af2aac2dcf12328a683b545280aa7f267f877a3c8d11ef0308e401
  aws-cluster-api-controllers                    quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a70ac366dcbd7d3a2d091ec2e505547b035a7a670e6659ac45c6e2708adcdacb
  aws-ebs-csi-driver                             quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:3915ed2289c5df36d8557dfb64a442874ac3229a5bf3651688090ca67d744397
  aws-ebs-csi-driver-operator                    quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:e15bcec2fd2008137bf3a6166c46d94c6e15621f2a5b99ba7677b980a3cc36e9
  aws-machine-controllers                        quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a5da07560223c050ac03d1643153e538096f74285ea5dc78b6816ba71966432a
  aws-pod-identity-webhook                       quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a463ae407f3dac940661bcccb9fa649f3592649c5499556b1d4c98834fd50a30
  azure-cloud-controller-manager                 quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:f8a035f9cad69a32454e1943c450afae77ae659d30b73b3cbc7ffda5ba75e4cf
  azure-cloud-node-manager                       quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:f6768f135c2b8147f442632cb494f5c71646e50786f914b1351c64f87302cca8
  azure-cluster-api-controllers                  quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:f4a89044c532c431c8276974f99d9e4082a54f8ffb8eecf8527e593e0ea16126

```