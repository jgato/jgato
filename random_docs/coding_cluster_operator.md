# My first steps playing/coding with an Openshift Cluster Operator

I just want to take one Openshift Cluster Operator, make some changes, build it and run it into a cluster.

The choosen one is the Baremetal Operator.

## Show me the code

This step is very trivial, just fork the [operator](https://github.com/openshift/baremetal-operator). Because I am using a testing cluster with OCP4.14, I will work from the branch `release-4.14`. 

Then do the usual `make build` and `make docker` to generate the image and push it to any container registry. 

```bash
> make docker
cd apis; /home/jgato/Projects-src/github/baremetal-operator/tools/bin/controller-gen object:headerFile="../hack/boilerplate.go.txt" paths="./..."
tools/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
cd apis; /home/jgato/Projects-src/github/baremetal-operator/tools/bin/controller-gen "crd:allowDangerousTypes=true,crdVersions=v1" rbac:roleName=manager-role webhook paths="./..." output:webhook:dir=../config/webhook/ output:crd:artifacts:config=../config/crd/bases
tools/bin/controller-gen rbac:roleName=manager-role paths="./..." output:rbac:artifacts:config=config/rbac
tools/bin/kustomize build config/default > config/render/capm3.yaml
docker build . -t baremetal-operator:latest --build-arg http_proxy= --build-arg https_proxy=
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
make: *** [Makefile:191: docker] Error 1
```

Ok, we prefer to build it with Podman:

```bash
> podman build . -t baremetal-operator:latest --build-arg http_proxy= --build-arg https_proxy=
[1/2] STEP 1/13: FROM docker.io/golang:1.19.9@sha256:86af5649fa1d9265d3fe7caf633231340b93e4164b96e14bc4e1131a191c1ddd AS builder
Trying to pull docker.io/library/golang@sha256:86af5649fa1d9265d3fe7caf633231340b93e4164b96e14bc4e1131a191c1ddd...
Getting image source signatures
Copying blob 6a2aa356f854 done   | 
<REDACTED>
[2/2] COMMIT baremetal-operator:latest
--> 84757ef10542
Successfully tagged localhost/baremetal-operator:latest
84757ef10542a4e5e51cbafbde6c9d8a5fe981b9d8d1fd186096803077500ef9
```

And push it:

```bash
> podman push localhost/baremetal-operator:latest quay.io/jgato/baremetal-operator:4-14
Getting image source signatures
Copying blob 6fbdf253bbc2 done   | 
Copying blob ff5700ec5418 done   | 
Copying blob e624a5370eca done   | 
Copying blob d52f02c6501c done   | 
Copying blob 7bea6b893187 done   | 
Copying blob e023e0e48e6e done   | 
Copying blob 1a73b54f556b done   | 
Copying blob 0e0bde1d0f54 done   | 
Copying blob d2d7ec0f6756 done   | 
Copying blob 4cb10dd2545b done   | 
Copying config 84757ef105 done   | 
Writing manifest to image destination


```

## Make Baremetal Operator unmanaged

Because it is a Cluster Operator we have to tell Clusteversion Operator to not manage it. I still have no clear how to make this operator unmanaged (ToDo). But, we can just disable the Clusterversion Operator to not overwrite the changes that we will do. Anyway, we are just playing.

```bash
> oc scale deployment cluster-version-operator -n openshift-cluster-version --replicas=0

```

## Modify the Baremetal Operator to use our new image


We have to modify the Baremetal Operator images to use the new one generated. The Baremetal Operator is pretty complex and composed by many images. Here, we are just modifying the Baremetal Operator image:

```yaml
> oc -n openshift-machine-api get cm cluster-baremetal-operator-images -o yaml
apiVersion: v1
data:
  images.json: |
    {
      "clusterBaremetalOperator": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:ca330a1ea546e5577c21e7d222bb2749a04c1ae870aba07eb719af8eb190a5ca",
      "kubeRBACProxy": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:20ed327b89dd0c1713419c451ff6f1bb00fd4ea25c137baf828fae824b10c72b",
      "baremetalOperator": "quay.io/jgato/baremetal-operator:4-14",
      "baremetalIronic": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:05572d39d1abd52f83ce2db716ba018584edce83d5d62d35b7b91d05dc7c17dd",
      "baremetalMachineOsDownloader": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:cad1314cfcdd7097ccfb64d46e243113e2e48d8f0bd3dc825d798b3bafcc53f0",
      "baremetalStaticIpManager": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:047f22503ec75bfed9177ca3f1d1e10947beebdbf93dd15b3319e9d41a9d7249",
      "baremetalIronicAgent": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:0134af9c1b3179482be7b9d12d135eeae306daeca330c34f543e2ac6d57d9162",
      "imageCustomizationController": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:02deda8fd8f9fecbe0e4d1766078b4bc557e06a08330b7d92ef77143dbc902cf",
      "machineOSImages": "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:bb90f3729b1e82d4e2d7d78030d667d0682708990b56fee3a90202a06951c2fb"
    }


```

This one `"baremetalOperator": "quay.io/jgato/baremetal-operator:4-14"`.

Now, kill the pod to regenerate it with the new image:

```bash
> oc -n openshift-machine-api delete pods cluster-baremetal-operator-xxxxxxxx
```

# Lets code a little

I will start with the branch `release-4.14` to create my own hello branch:

```bash
> git checkout release-4.14
Already on 'release-4.14'
Your branch is up to date with 'upstream/release-4.14'.
> git branch just-hello
> git checkout just-hello 
Switched to branch 'just-hello'
Your branch is up to date with 'origin/just-hello'.

```

Lets do some code:

https://github.com/jgato/baremetal-operator/commit/e5aa7ca0f7f6b018bd6225fe7fa3ee99541d5a7c

Just saying hello.

And we build again the Container Image:

```bash
> make build
cd apis; /home/jgato/Projects-src/github/baremetal-operator/tools/bin/controller-gen object:headerFile="../hack/boilerplate.go.txt" paths="./..."
tools/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
cd apis; /home/jgato/Projects-src/github/baremetal-operator/tools/bin/controller-gen "crd:allowDangerousTypes=true,crdVersions=v1" rbac:roleName=manager-role webhook paths="./..." output:webhook:dir=../config/webhook/ output:crd:artifacts:config=../config/crd/bases
tools/bin/controller-gen rbac:roleName=manager-role paths="./..." output:rbac:artifacts:config=config/rbac
tools/bin/kustomize build config/default > config/render/capm3.yaml
go build -ldflags "-X "github.com/metal3-io/baremetal-operator/pkg/version".Raw=e5aa7ca0f7f6b018bd6225fe7fa3ee99541d5a7c -X "github.com/metal3-io/baremetal-operator/pkg/version".Commit=e5aa7ca0f -X "github.com/metal3-io/baremetal-operator/pkg/version".BuildTime=2024-04-26T13:37:59+0200" -o bin/baremetal-operator main.go
go build -o bin/get-hardware-details cmd/get-hardware-details/main.go
go build -o bin/make-bm-worker cmd/make-bm-worker/main.go
go build -o bin/make-virt-host cmd/make-virt-host/main.go

> podman build . -t baremetal-operator:latest --build-arg http_proxy= --build-arg https_proxy=
[1/2] STEP 1/13: FROM docker.io/golang:1.19.9@sha256:86af5649fa1d9265d3fe7caf633231340b93e4164b96e14bc4e1131a191c1ddd AS builder
<REDACTED>
[2/2] COMMIT baremetal-operator:latest
--> eb47c0d1f557
Successfully tagged localhost/baremetal-operator:latest
eb47c0d1f557c031519c03375f3b7d13a0580056dc6d00b0aa70f56d83f72644

> podman push localhost/baremetal-operator:latest quay.io/jgato/baremetal-operator:4-14
Getting image source signatures
Copying blob eb6702f4c235 skipped: already exists  
Copying blob 927ef23185e7 skipped: already exists  
Copying blob c4629cdd872f skipped: already exists  
Copying blob c23224575d87 skipped: already exists  
Copying blob ef5c7eb8a157 skipped: already exists  
Copying blob e9706f3b6b9a done   | 
Copying blob b4aa04aa577f skipped: already exists  
Copying blob 4e700e4525b7 skipped: already exists  
Copying blob b49267e86090 skipped: already exists  
Copying blob 80f00805faa8 skipped: already exists  
Copying config eb47c0d1f5 done   | 
Writing manifest to image destination
```

And just kill again the operator:

```bash
> oc -n openshift-machine-api delete pod cluster-baremetal-operator-85f5ccc89c-kl8dj
pod "metal3-baremetal-operator-7fcbf77b97-5rwh7" deleted

```

We added a line in the reconciler function, so we will see our message very frequently: 

```bash
> oc -n openshift-machine-api logs metal3-baremetal-operator-7fcbf77b97-4ppsv | grep "just me"
{"level":"info","ts":1714131677.920332,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno1","namespace":"sno1"}}
{"level":"info","ts":1714131677.9203718,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno2","namespace":"sno2"}}
{"level":"info","ts":1714131677.9203756,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno3","namespace":"sno3"}}
{"level":"info","ts":1714131677.9204252,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno4","namespace":"sno4"}}
{"level":"info","ts":1714131677.9204528,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"master-2.hub-2.el8k.se-lab.eng.rdu2.dc.redhat.com","namespace":"openshift-machine-api"}}
{"level":"info","ts":1714131677.9204834,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"master-0.hub-2.el8k.se-lab.eng.rdu2.dc.redhat.com","namespace":"openshift-machine-api"}}
{"level":"info","ts":1714131677.9205441,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"master-1.hub-2.el8k.se-lab.eng.rdu2.dc.redhat.com","namespace":"openshift-machine-api"}}
{"level":"info","ts":1714131678.2192159,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno3","namespace":"sno3"}}
{"level":"info","ts":1714131678.219884,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno2","namespace":"sno2"}}
{"level":"info","ts":1714131678.2205665,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno1","namespace":"sno1"}}
{"level":"info","ts":1714131678.2271695,"logger":"controllers.BareMetalHost","msg":"eyyy it is just me saying hello","baremetalhost":{"name":"sno4","namespace":"sno4"}}

```

> The cluster-baremetal-operator manages the deployments of the metal3-baremetal-operator and other deployments. And the ImagePullPolicy is set to IfNotPresent. So, every time you make a change, create a new tag, edit the CM to point the new image, and kill again the cluster-baremetal-operator. Or, scale to 0 the cluster-baremetal-operator, and you can directly manage the deployment of the metal3-baremetal-operator and set the ImagePull to Always. Yes it is tricky.
