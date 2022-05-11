# How to interact with an OCP catalogs



OCP comes with a set of catalogs where different operators are distributed.

```bash
oc get catalogsources.operators.coreos.com  -A
NAMESPACE               NAME                  DISPLAY               TYPE   PUBLISHER   AGE
openshift-marketplace   certified-operators   Certified Operators   grpc   Red Hat     98d
openshift-marketplace   community-operators   Community Operators   grpc   Red Hat     98d
openshift-marketplace   redhat-marketplace    Red Hat Marketplace   grpc   Red Hat     98d
openshift-marketplace   redhat-operators      Red Hat Operators     grpc   Red Hat     98d

```

By default you will see something like this, with 4 catalogs. Depending on the catalog, operators could not have Red Hat official Support.

For each of these catalogs, you can have the url where the repository is placed.:

```bash
> oc get -n openshift-marketplace catalogsources.operators.coreos.com certified-operators  -o jsonpath={.spec.image}
registry.redhat.io/redhat/certified-operator-index:v4.9
```

In order, to interact with the catalog you will need an authorization. Usually, it is called 'pull_secret', and you can get this from your Red Hat account [here](). Interacting with the catalog of an OCP installation would require more perms. So, it is recommended to use the 'pull_secret' used during the installation. 

In any case, you can check to which repository you 'pull_secret' has access, thanks to this [CLI pullsecret-validator](https://github.com/RHsyseng/pullsecret-validator-cli) created by [@alknopfler](https://github.com/alknopfler/alknopfler)

```bash
> ./pullsecret-validator-cli -f ~/pull-secret.jgato 
2022/05/11 09:48:21 Starting the Pull Secret file validation...It could take a time!
┌─────────────────────────────┬───────────────────────┬─────────────────────┐
│ VALID AUTHS ENTRIES         │ EXPIRED AUTHS ENTRIES │ CONNECTION ISSUES   │
├─────────────────────────────┼───────────────────────┼─────────────────────┤
│ quay.io                     │                       │ cloud.openshift.com │
│ registry.connect.redhat.com │                       │                     │
│ registry.redhat.io          │                       │                     │
│                             │                       │                     │
└─────────────────────────────┴───────────────────────┴─────────────────────┘

```

So I will have enough perms to interact with redhat official registries.

You can search for Operators:

```bash
> oc-mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.10 --package=sriov-network-operator
PACKAGE                 CHANNEL  HEAD
sriov-network-operator  4.10     sriov-network-operator.4.10.0-202204211158
sriov-network-operator  stable   sriov-network-operator.4.10.0-202204211158

```


