# How to build your own container registry and an Openshift catalog

Specially useful creating disconnected environments:
 * First you create a container registry with quay.io
 * Then, you use `oc-mirror` to get your needed images and populate them into the registry.
 * Finally, in your Openshift cluster, create a new CatalogSource pointing to the new registry


## Build the registry


## Populate the registry

Download the oc-mirror, and generate the imageset according to your needs:

```
---
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
#  platform:
#    channels:
#    - name: stable-4.18
#      minVersion: 4.18.5
#      maxVersion: 4.18.5
#    graph: false
  operators:
    - catalog: "registry.redhat.io/redhat/redhat-operator-index:v4.18"
      packages:
      - name: advanced-cluster-management
        channels:
        - name: release-2.13
      - name: multicluster-engine
        channels:
        - name: stable-2.8
      - name: topology-aware-lifecycle-manager
        channels:
        - name: stable
      - name: openshift-gitops-operator
        channels:
        - name: gitops-1.15
      - name: odf-operator
        channels:
        - name: stable-4.18
      - name: ocs-operator
        channels:
        - name: stable-4.18
      - name: rook-ceph-operator
        channels:
        - name: stable-4.18
      - name: mcg-operator
        channels:
        - name: stable-4.18
      - name: odf-dependencies
        channels:
        - name: stable-4.18
      - name: ocs-client-operator
        channels:
        - name: stable-4.18
      - name: odf-csi-addons-operator
        channels:
        - name: stable-4.18
      - name: cephcsi-operator
        channels:
        - name: stable-4.18
      - name: odf-prometheus-operator
        channels:
        - name: stable-4.18
      - name: local-storage-operator
        channels:
        - name: stable
  additionalImages:
  - name: registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.18.0
  - name: registry.redhat.io/rhel8/support-tools:8.9-7
```

For this example, I am not mirroring any platform because of storage constrains. And I am basically pulling some operators.

Then, use oc-mirror to pull the images and push them to the previously created registry:

```bash
$ oc mirror --v2 --config=./imageset-config-4.19.yaml --workspace file://oc-mirror-workspace docker://registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443
```

Notice: `oc mirror` or `oc-mirror` will use the pull secret from your `/run/user/1001/containers/auth.json`.

## Create the CatalogSource

Before creating the CatalogSource, we have to create the `ImageContentSourcePolicy` that makes the redirections. You cannot change every workload definition to point to the new registry, so, you use a re-direction. In our case, of some operators and container images:

```yaml
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: generic-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/rhel8
    source: registry.redhat.io/rhel8
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: operator-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/multicluster-engine
    source: registry.redhat.io/multicluster-engine
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/odf4
    source: registry.redhat.io/odf4
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/rhel8
    source: registry.redhat.io/rhel8
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/rh-sso-7
    source: registry.redhat.io/rh-sso-7
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/openshift-gitops-1
    source: registry.redhat.io/openshift-gitops-1
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/rhacm2
    source: registry.redhat.io/rhacm2
```

Because the registry uses a selfsigned certificate (at least, in my case), we have to add the CA to the trusted list:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-cas
  namespace: openshift-config
data:
  registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com..8443: |
    -----BEGIN CERTIFICATE-----
    MIIGBzCCA <REDACTED>    rtuRldHSSr9
    -----END CERTIFICATE-----

```

And we add it to cluster image configuration:
```bash
> oc patch image.config.openshift.io/cluster --patch '{"spec":{"additionalTrustedCA":{"name":"registry-cas"}}}' --type=merge
```

Usually, you should have deployed the cluster with a pull-secret containing the credentials of the new registry. If not your case, update and add it:

```
> oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/home/jgato/.config/containers/auth.json
```

In my auth.json I have the credentials for the new registry in the way of:

```yaml
    "registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443": {
      "auth": "YWRtaW....SE="
    },

```

With the auth value in the way of "user:password" base64 encoded (be careful dont generate it with return line).

Now, we create a CatalogSource in our marketplace but pointing to our registry index. This index only contain the operators we have mirrored before:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: redhat-operators-disconnected
  namespace: openshift-marketplace
spec:
  image: registry.infra.el8k.se-lab.eng.rdu2.dc.redhat.com:8443/redhat/redhat-operator-index:v4.18
  sourceType: grpc

```
