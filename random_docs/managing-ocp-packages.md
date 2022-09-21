# Managing OCP packages and repositories

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

## Getting kernel packages versions.

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