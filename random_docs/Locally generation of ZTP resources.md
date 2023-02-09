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


