# Locally generation of ZTP resources

ZTP Gitops is based on two kustomize plugins. Above ZTP4.12, you can use the ztp-site-generate container image, the execute locally this plugins:

```bash
> podman run  --rm registry.redhat.io/openshift4/ztp-site-generate-rhel8  generator --help
ztp-site-generator single site resources generator

Usage: generator <subcommand> [options]

Available commands:
  install          generate installation CRs of given SiteConfig(s)
  config           generate configuration CRs of given PolicyGenTemplate(s)

Options:
  --help Help for generator

```

This tool is a good way of simulating, what it is going to be done inside ArgoCD by the plugins. 

## Generating SiteConfig Resources

You need a directory with SiteConfigs:

```bash
> ls
el8k-ztp-1-3node-dualstack-4-10.yaml            el8k-ztp-1-compact-4-10.yaml            el8k-ztp-1-standard-4-8.yaml             el8k-ztp-1-standard-ipv6.yaml  sno3-e-4-10.yaml          sno4.yaml                  sno-b7-e-4-10-ipv4.yaml
el8k-ztp-1-3node-ipv6-4-10-extra-manifest.yaml  el8k-ztp-1-compact-dualstack-4-12.yaml  el8k-ztp-1-standard-dualstack-4-12.yaml  el8k-ztp-1-standard.yaml       sno3-ipv6-4-12.yaml       sno5-e-4-9-only-sctp.yaml  sno-b8-e-4-10-ipv4.yaml
el8k-ztp-1-3node-ipv6-4-10.yaml                 el8k-ztp-1-compact-ipv4-4-12.yaml       el8k-ztp-1-standard-ipv4-4-12.yaml       extra-manifests                sno4-dual-4-12.yaml       sno5-e-4-9.yaml
el8k-ztp-1-3node-ipv6.yaml                      el8k-ztp-1-compact-ipv6-4-12.yaml       el8k-ztp-1-standard-ipv6-4-12.yaml       kustomization.yaml             sno4-e-4-10-no-sctp.yaml  sno5-ipv4-4-12.yaml
el8k-ztp-1-3node.yaml                           el8k-ztp-1-standard-4-10.yaml           el8k-ztp-1-standard-ipv6-4-8.yaml        out                            sno4-e-4-10.yaml          sno5.yaml

```

In this case, I will generate all the CRs from the Siteconfig 'l8k-ztp-1-compact-ipv4-4-12.yaml'

```bash
> podman run  --rm -v ${PWD}:/resources:Z,U \
  registry.redhat.io/openshift4/ztp-site-generate-rhel8  \
  generator install ./el8k-ztp-1-compact-ipv4-4-12.yaml 
Processing SiteConfigs: /resources/el8k-ztp-1-compact-ipv4-4-12.yaml 
Generating installation CRs into /resources/out/generated_installCRs ...

```


## Generating PolicyGenTemplates Resources

In a directory with different PGTS:

```bash
> ls -l mb-du-sno/
total 16
-rw-r--r--. 1 jgato jgato 1774 jul 17 17:40 common.yaml
-rw-r--r--. 1 jgato jgato 2209 jul 17 16:43 group-du-mb-operator-config.yaml
-rw-r--r--. 1 jgato jgato  552 jul 17 09:59 group-du-mb-operator-fec.yaml
-rw-r--r--. 1 jgato jgato 1317 jul 17 17:17 group-du-mb-performance-config.yaml
```

Then, you can generate Policies and Bindings:
```bash
> podman run  --rm -v ${PWD}:/resources:Z,U \ 
  registry.redhat.io/openshift4/ztp-site-generate-rhel8 \
  generator config ./mb-du-sno/common.yaml 
Processing PolicyGenTemplates: /resources/mb-du-sno/common.yaml 

```

### Using your extra source-crs

If your PGTs are using your own source-crs, you have to add these to the Container Image. So, you will have the default [source-crs](https://github.com/openshift-kni/cnf-features-deploy/tree/master/ztp/source-crs) that were used to build the container. Plus, your own source-crs

We will have to modify you local image of the ztp-site-generate. So, create a Container:
```bash
$> podman run  -it -v ${PWD}:/resources:Z,U  registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.12c /bin/bash

$> ls -l /kustomize/plugin/ran.openshift.io/v1/policygentemplate/source-crs       
total 332
-rw-r--r--. 1 root root  134 Jan  5  2023 AcceleratorsNS.yaml
-rw-r--r--. 1 root root  246 Jan  5  2023 AcceleratorsOperGroup.yaml
-rw-r--r--. 1 root root  727 Jan  5  2023 AcceleratorsOperatorStatus.yaml
-rw-r--r--. 1 root root  380 Jan  5  2023 AcceleratorsSubscription.yaml
-rw-r--r--. 1 root root  260 Jan  5  2023 AmqInstance.yaml
-rw-r--r--. 1 root root  670 Jan  5  2023 AmqOperatorStatus.yaml
<REDACTED>
```

Just keep it running, we will copy extra source-crs to the default folder. In other terminal:

```bash
# Take the container ID
> podman ps
CONTAINER ID  IMAGE                                                         COMMAND     CREATED         STATUS         PORTS       NAMES
e885d1a2637e  registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.12c  /bin/bash   22 seconds ago  Up 22 seconds              priceless_carson

# Copy there our own source-crs
> podman cp source-crs/. e885d1a2637e:/kustomize/plugin/ran.openshift.io/v1/policygentemplate/source-crs/

# commit the changes
> podman commit e885d1a2637e
Getting image source signatures
Copying blob 52cbfc36b72b skipped: already exists  
Copying blob f1732c6974a1 skipped: already exists  
Copying blob bd2f5c65dade skipped: already exists  
Copying blob cd719646bf14 done  
Copying config 694b95eb8d done  
Writing manifest to image destination
Storing signatures
694b95eb8d819bc298c0de48ca4008eef389582f4becf981bd5516edf0a73acf

# tag the generated image
> podman tag 694b95eb8d819bc298c0de48ca4008eef389582f4becf981bd5516edf0a73acf  registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.12c
```

Notice we have now a new image with tag: v4.12c.

Now, you can exit the previous created container. And you can execute it again (with the new image) as explained above:

```bash
podman run  --rm -v ${PWD}:/resources:Z,U \ 
  registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.12c \
  generator config ./mb-du-sno/common.yaml 
```