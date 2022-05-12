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

So I will have enough perms to interact with redhat official registries. And your auth file on path '${XDG_RUNTIME_DIR}/containers/auth.json'

*There is known bug with oc-mirror 4.10 that would make you copy the auth.json file on ~/.docker/config.json*

You can search for Operators:

```bash
> oc-mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.10 --package=sriov-network-operator
PACKAGE                 CHANNEL  HEAD
sriov-network-operator  4.10     sriov-network-operator.4.10.0-202204211158
sriov-network-operator  stable   sriov-network-operator.4.10.0-202204211158
```

You can extend the search of an specific operator and a concrete channel to get all the available versions

```bash
> oc-mirror list operators --catalog=registry.redhat.io/redhat/community-operator-index:v4.9 --package=hive-operator --channel=alpha
VERSIONS
1.2.3254-512b7f4
1.2.3299-f0cc1ba
1.1.11
1.1.14
1.2.3606-2544e8a
1.1.10
1.2.3481-ad1892a
1.2.3427-e50103a
1.2.3552-133480f
1.2.3598-e150a7f
1.2.3633-595563c
1.1.6
1.2.3222-1411793
1.2.3337-d668960
1.2.3359-8f0773c
1.2.3588-5ccea30
1.2.3594-20fd254
1.1.16
1.1.8
1.1.9
1.2.3329-16c6e0f
1.2.3490-7e0b248
1.2.3530-0be0b66
1.1.13
1.1.7
1.1.5
1.2.3375-1f0f3eb
1.2.3461-a607db0
1.2.3506-f54f0ec
1.2.3542-688681d
1.2.3557-bfb69ae
1.1.15
1.2.3274-a27a4f7
1.2.3310-bc1417a
1.2.3391-55f6363
1.2.3431-6ba92bb
1.2.3457-b212e8a
1.2.3577-1549a35
1.2.3581-88b1c28
1.1.12
1.2.3179-0756b70e6


```