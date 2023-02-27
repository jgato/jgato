# Capabilities and containers

Small tutorial, tips, tricks, etc

If you really need to learn about this, go [GitHub - mvazquezc/scc-fun: Repository with files used for an SCC workshop](https://github.com/mvazquezc/scc-fun)

[Linux Kernel capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)  adds extra privileges, a part to the traditional file's perms. Basically, the old powerful root privileges were divided into different capabilities. When you execute any program, there are a set of capabilities in place that determinate what you can do.  

Containers runtimes have some of these capabilities enabled by default. But, which capabilities are required by a running container? This will depend on each container and their different needs. It will depend on how these needs to access to the Kernel API and syscalls. Even for the developer, it would be hard to know which capabilities needs. It would not now the details of which sys calls its program will do, and which capabilities are needed. But even, it would differ depending on your container runtime (runc, crun..)

## Different capabilities

There are thread and file capabilities.

Thread capabilities:

* Permitted set: the capabilities that can be used by a thread

* Inheritable set: the capabilities that can be transmitted when executing a program (execv). These are added to the program as permitted Set, only, if this program has the capabilities also as inheritable. These transmission is not allowed is the thread is non-root. In case of non-root it depends on Ambient set.

* Ambient set: the capabilities that can be transmitted when executing a program (execv) as non-root. An ambient set capability has to be also permitted and inheritable. So, it can be considered as an add-on to the inheritable, if it is going to be executed as non-root. But it is not a substitution of inheritable set.

* Bounding set: which capabilities can be gained during a program execution (execve).

* Effective set: Capabilities used by the kernel to perform permission checks for the thread.. This is what the kernel will look at, when you request a privileged action. This list is actually what you need to have enabled to perform the request.

Files capabilities, which are capabilities assigned to an executable file:

* Permitted set: capabilities acquired when executing the program and received by the main thread.

* Inheritable:  which capabilities can be permitted on an execve inside the thread.

* Effective: we would say.. the enabled capabilities from the permitted set.

When you execute a program, files capabilities go in play. It can do, what it is effective and permitted. But this program was called, maybe, from a bash (running thread). So, the bash thread capabilities affect to the new program file capabilities.

You can check any process capabilities with:

```bash
#  grep Cap /proc/$PROCESS/status
CapInh:    0000000000000000
CapPrm:    00000000a80425fb
CapEff:    00000000a80425fb
CapBnd:    00000000a80425fb
CapAmb:    0000000000000000
```

If this program, execute other programs (execve) capabilities are copied. Depending on Inheritable, Bounding and Ambient sets of the executed program. 

When you use a podman run,  the process capabilities depend on the program file capabilities.  But podam will add (if root), or drop (if non root) different sets (this will change depending on the implementation) to  default capabilities. 

If you use podman run with --add-capabilities. It will act as above, but the running thread will be modified about thread capabilities (setcap).  You cannot set capabilities that are not in the permitted ones. 

When you do an execv, the file capabilities of the program has to be compatible with the Inheritable/Bounding thread capabilities (of the thread executing the program). 

If you do a podman run to open a bash, the bash thread will have a set of capabilities. Imagine you run an service-app with a file capability (permitted, effective)  Net_Bind_service. If the bash program thread dont have a Bounding Net_Bind_service, the file capability will not be set as effective. And it will bind when tries to listen on a privileged port. 

You can get and set file capabilities with: [getcap, setcap and file capabilities](https://www.insecure.ws/linux/getcap_setcap.html)

## Capabilities and containers

How these previous capabilities work when running containers, instead of programs. 

We will use 'reversewords-test', an app listening on a given port.

Containers with UID 0 (root) the default capabilities (which depends on your runtime) are added as Effective set. For example, these are the default podman capabilities: [common/default.go at v0.33.1 · containers/common · GitHub](https://github.com/containers/common/blob/v0.33.1/pkg/config/default.go#L62-L77)

```bash
> podman run --rm -it --entrypoint /bin/bash --name reversewords-test quay.io/mavazque/reversewords:ubi8 
[root@4b2cb96f0925 reverse-words]#  grep Cap /proc/1/status
CapInh:    0000000000000000
CapPrm:    00000000a80425fb
CapEff:    00000000a80425fb
CapBnd:    00000000a80425fb
CapAmb:    0000000000000000
```

>  In this case, /proc/1 is /bin/bash

Containers with UID non 0. Default runtime capabilities are dropped, but you still have them on bounding. We can request extra capabilities that will be added to the container thread as effective set. 

```bash
> podman run --rm -it --user 1024 --entrypoint /bin/bash --name reversewords-test quay.io/mavazque/reversewords:ubi8 
bash-4.4$ grep Cap /proc/1/status
CapInh:    0000000000000000
CapPrm:    0000000000000000
CapEff:    0000000000000000
CapBnd:    00000000a80425fb
CapAmb:    0000000000000000
```

Why bounding ones are enabled? This will allow the program, to lunch other programs. The new program:

```bash
[root@4b2cb96f0925 reverse-words]# bash
[root@4b2cb96f0925 reverse-words]# grep Cap /proc/19/status
CapInh:    0000000000000000
CapPrm:    00000000a80425fb
CapEff:    00000000a80425fb
CapBnd:    00000000a80425fb
CapAmb:    0000000000000000
```

But we could request other capabilities:

```bash
> podman run --rm -it --user 1024 --cap-add=cap_net_bind_service --entrypoint /bin/bash --name reversewords-test quay.io/mavazque/reversewords:ubi8
bash-4.4$ grep Cap /proc/1/status
CapInh:    0000000000000400
CapPrm:    0000000000000400
CapEff:    0000000000000400
CapBnd:    00000000a80425fb
CapAmb:    0000000000000400
bash-4.4$ capsh --decode=0000000000000400
0x0000000000000400=cap_net_bind_service
```

The --cap-add are added as thread capabilities (capset).

## Seccomp - Security Profiles

In this case, instead of adding or giving capabilities, you fine tune which syscalls a process, can, or cannot, do. It is like a filter of syscalls. One capability would be affected by more than one syscalls. Thefore, Seccomp are more complex to create, and it requires a deep knowledge of what your code do, at syscalls level. These profiles are stored phisicaly (as files). 

To get a Seccomp profile about a running process, you can do:

```bash
> sudo podman run --rm --annotation \
io.containers.trace-syscall="of:/tmp/ls.json" fedora:32 ls / > /dev/null

{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "access",
        "arch_prctl",
        "brk",
        "capset",
        "close",
        "execve",
        "exit_group",
        "fstat",
        "getdents64",
        "ioctl",
        "mmap",
        "mprotect",
        "munmap",
        "openat",
        "prctl",
        "pread64",
        "prlimit64",
        "read",
        "rt_sigaction",
        "rt_sigprocmask",
        "select",
        "set_robust_list",
        "set_tid_address",
        "setresgid",
        "setresuid",
        "stat",
        "statfs",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [],
      "comment": "",
      "includes": {},
      "excludes": {}
    }
  ]
}
```

The Seccomp file is not on `/tmp/ls.json`

And then, you can use it to run the container:

```bash
> podman run --rm --security-opt seccomp=/tmp/ls.json fedora:32 ls / > /dev/null
```

# Openshift Security Context Constraints

SCC in openshift works in addition to RBAC resources. A set of conditions for a Pod to be accepted in the system. Similar to the Kubernetes security context resource.

Security context constraints allow an administrator to control:

- Whether a pod can run privileged containers with the `allowPrivilegedContainer` flag.

- Whether a pod is constrained with the `allowPrivilegeEscalation` flag.

- The capabilities that a container can request

- The use of host directories as volumes

- The SELinux context of the container

- The container user ID

- The use of host namespaces and networking

- The allocation of an `FSGroup` that owns the pod volumes

- The configuration of allowable supplemental groups

- Whether a container requires write access to its root file system

- The usage of volume types

- The configuration of allowable `seccomp` profiles

In this document we are focusing on using SCC to define the capabilities and seccomp. 

## How to get the capabilities of a running process

Root Running

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-root
spec:
  containers:
  - image: quay.io/mavazque/reversewords:ubi8
    name: reversewords
    securityContext:
      runAsUser: 0
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```
> oc -n test-scc exec -ti reversewords-app-captest-root -- grep Cap /proc/1/status
CapInh:    0000000000000000
CapPrm:    00000000000005fb
CapEff:    00000000000005fb
CapBnd:    00000000000005fb
CapAmb:    0000000000000000
```

Notice how default capabilities are different than running podman.

Non-root running:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-nonroot
spec:
  containers:
  - image: quay.io/mavazque/reversewords:ubi8
    name: reversewords
    securityContext:
      runAsUser: 1024
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
> oc -n test-scc exec -ti reversewords-app-captest-nonroot -- grep Cap /proc/1/status
CapInh:    0000000000000000
CapPrm:    0000000000000000
CapEff:    0000000000000000
CapBnd:    00000000000005fb
CapAmb:    0000000000000000
```

Notice that Ambient capabilities are not supported, in this moment, on Kubernetes. So this will make different behavior if you are just using Podman or Kubernetes. So, when non-root, we can only have file capabilities. 

If non-root, you cannot transmit capabilities. If you want to run a process to listen on port 80, no matter if you ask for the NET_BIND_SERVICE, it cannot be transmitted. The only way is: setting the binary to have the file capability (effective and permitted). And also, requesting it when configuring the Pod or the Deployment.

## Running a Pod/Deployment with specific capabilities

The following deployment adds NET_BIND_SERVICE

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: reversewords-app-rootuid
  name: reversewords-app-rootuid
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reversewords-app-rootuid
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: reversewords-app-rootuid
    spec:
      containers:
      - image: quay.io/mavazque/reversewords:ubi8
        name: reversewords
        resources: {}
        env:
        - name: APP_PORT
          value: "80"
        securityContext:
          runAsUser: 1024
          capabilities:
            drop:
            - CHOWN
            - DAC_OVERRIDE
            - FSETID
            - FOWNER
            - SETGID
            - SETUID
            - SETPCAP
            - KILL
            add:
            - NET_BIND_SERVICE
status: {}
```

But it is not running as root. Thefore, the 'quay.io/mavazque/reversewords:ubi8' container will need to have 'NET_BIND_SERVICE' as a file Capability. Because Openshift/Kubernete dont have Ambient capability, so, it cannot be transmitted to the, later, running thread.

## Running a pod with a Seccomp

You can create whatever Seccomp:

```yaml
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "access",
        "arch_prctl",
        "brk",
        "capget",
        "capset",
        "chdir",
        "close",
        "epoll_ctl",
        "epoll_pwait",
        "execve",
        "exit_group",
        "fchown",
        "fcntl",
        "fstat",
        "fstatfs",
        "futex",
        "getdents64",
        "getpid",
        "getppid",
        "ioctl",
        "mmap",
        "mprotect",
        "munmap",
        "nanosleep",
        "newfstatat",
        "openat",
        "prctl",
        "pread64",
        "prlimit64",
        "read",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sched_yield",
        "seccomp",
        "set_robust_list",
        "set_tid_address",
        "setgid",
        "setgroups",
        "setuid",
        "stat",
        "statfs",
        "tgkill",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": [],
      "comment": "",
      "includes": {},
      "excludes": {}
    }
  ]
}
```

Copying this to all the hosts running kubelet: `/var/lib/kubelet/seccomp/centos8-ls.json`

The following deployment will use it Seccomp:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-ls-test
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: centos8-ls.json
  containers:
  - image: registry.centos.org/centos:8
    name: seccomp-ls-test
    command: ["ls", "/"]
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

## SCCs

all the default existing SCCs:

```bash
> oc get scc 
NAME                              PRIV    CAPS                   SELINUX     RUNASUSER          FSGROUP     SUPGROUP    PRIORITY     READONLYROOTFS   VOLUMES
anyuid                            false   <no value>             MustRunAs   RunAsAny           RunAsAny    RunAsAny    10           false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
hostaccess                        false   <no value>             MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","hostPath","persistentVolumeClaim","projected","secret"]
hostmount-anyuid                  false   <no value>             MustRunAs   RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","hostPath","nfs","persistentVolumeClaim","projected","secret"]
hostnetwork                       false   <no value>             MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
hostnetwork-v2                    false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
machine-api-termination-handler   false   <no value>             MustRunAs   RunAsAny           MustRunAs   MustRunAs   <no value>   false            ["downwardAPI","hostPath"]
node-exporter                     true    <no value>             RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["*"]
nonroot                           false   <no value>             MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
nonroot-v2                        false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
privileged                        true    ["*"]                  RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["*"]
restricted                        false   <no value>             MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
restricted-v2                     false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
```

By default, in OpenShift, all pods and containers will use the Restricted SCC. 

If running a pod needs more perms/privileges the Restricted SCC, the user/serviceaccount, running the pod, needs to have assigned a proper SCC. For example, you cannot attache an app to a port 80 with a restricted SCC.

When you try to create (oc create...) a Pod, the requests goes to the API Server by your UserAccount. If you are authorized to create pod, this will run under a ServiceAccount. If not specified ServiceAccount, it will use the default one.

Even if the Pod  is run by the SA, **the validation checks runs against the SCCs of the SA and the User who created the Pod.**

```bash
> oc whoami 
system:admin
```

This is potentially dangerous, because the SCCs of the admin are, usually, more than the ones by the SA.

Which users or serviceaccounts have a concrete SCC:

```bash
> oc get scc privileged -o jsonpath={.users}
["system:admin","system:serviceaccount:openshift-infra:build-controller"]
```

Privileged SCC can be used by an Admin, or the build-controller SA on the openshift-infra NameSpace. 

A SA is composed in the way:  system:serviceaccount:namespace:name. 

### Adding SCC

To a SA. ServiceAccount are namespaced resources, so ensure you are adding the proper SA

```bash
> NAMESPACE="scc-fun"
> oc create ns ${NAMESPACE}
namespace/scc-fun created
> oc -n ${NAMESPACE} create serviceaccount testsa1
serviceaccount/testsa1 created
```

Now lets add the SCC 'anyuid' to testsa1 SA. 

```bash
> oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid system:serviceaccount:${NAMESPACE}:testsa1
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "testsa1"
```

Which is basically done by a RoleBinding:

```bash
> oc -n ${NAMESPACE} get rolebinding
NAME                          ROLE                                      AGE
system:deployers              ClusterRole/system:deployer               4m4s
system:image-builders         ClusterRole/system:image-builder          4m4s
system:image-pullers          ClusterRole/system:image-puller           4m4s
system:openshift:scc:anyuid   ClusterRole/system:openshift:scc:anyuid   41s
```

We have used the format system:serviceaccount:ns:name. But you can just use -z and the name of the SA:

```bash
> oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid -z testsa2
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "testsa2"
```

Both SA are added to the SCC:

```yaml
> oc -n ${NAMESPACE} get rolebinding system:openshift:scc:anyuid -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2023-02-18T16:59:48Z"
  name: system:openshift:scc:anyuid
  namespace: scc-fun
  resourceVersion: "1070549"
  uid: 410315bf-006f-443e-bb20-9dd8a4bae2f0
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
subjects:
- kind: ServiceAccount
  name: testsa1
  namespace: scc-fun
- kind: ServiceAccount
  name: testsa2
  namespace: scc-fun
```

you can also add the SCC to an User (not using -z, nor indicating SA Format):

```bash
> oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid jgato
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "jgato"
```

and here they are, Users and SAs:

```
> oc -n ${NAMESPACE} get rolebinding system:openshift:scc:anyuid -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2023-02-18T16:59:48Z"
  name: system:openshift:scc:anyuid
  namespace: scc-fun
  resourceVersion: "1071574"
  uid: 410315bf-006f-443e-bb20-9dd8a4bae2f0
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
subjects:
- kind: ServiceAccount
  name: testsa1
  namespace: scc-fun
- kind: ServiceAccount
  name: testsa2
  namespace: scc-fun
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: jgato
```

Or better use:

```bash
> oc -n ${NAMESPACE} adm policy who-can use scc anyuid
resourceaccessreviewresponse.authorization.openshift.io/<unknown> 

Namespace: scc-fun
Verb:      use
Resource:  securitycontextconstraints.security.openshift.io

Users:  jgato
        system:admin
        system:serviceaccount:assisted-installer:assisted-installer-controller
        system:serviceaccount:openshift-apiserver-operator:openshift-apiserver-operator
        system:serviceaccount:openshift-apiserver:openshift-apiserver-sa
<REDACTED>
        system:serviceaccount:openshift-kube-storage-version-migrator-operator:kube-storage-version-migrator-operator
        system:serviceaccount:openshift-kube-storage-version-migrator:kube-storage-version-migrator-sa
        system:serviceaccount:openshift-machine-api:cluster-baremetal-operator
        system:serviceaccount:openshift-machine-config-operator:default
        system:serviceaccount:openshift-network-operator:default
        system:serviceaccount:openshift-oauth-apiserver:oauth-apiserver-sa
        system:serviceaccount:openshift-operator-lifecycle-manager:olm-operator-serviceaccount
        system:serviceaccount:openshift-service-ca-operator:service-ca-operator
        system:serviceaccount:scc-fun:testsa1
        system:serviceaccount:scc-fun:testsa2
Groups: system:cluster-admins
        system:masters
```

### Default SCC

All the authenticated users have access to the `restricted-v2` SCC.

```bash
> oc  adm policy who-can use scc restricted-v2 -o jsonpath={.groups} |jq
[
  "system:authenticated",
  "system:cluster-admins",
  "system:masters"
]
```

So, all the ServiceAccounts have, in some way, at least, access to the `restricted-v2`. Because the user that created the Pod have it. But If you check the SCCs of a SA will not have (at least by default) that SCC. 

This happens, because the validation are done against SA and User SCCs. And all authenticated users will have this default SCC.

### SCC strategies

SCC attributes, like `seLinuxContext`, `fsGroup`, `supplementalGroups`, `runAsUser` are defined with an strategy. The supported strategies in these cases are `MustRunAs`, `RunAsAny`

```yaml
  SELinux Context Strategy: MustRunAs        
    User:                    <none>
    Role:                    <none>
    Type:                    <none>
    Level:                    <none>
  FSGroup Strategy: MustRunAs            
    Ranges:                    <none>
  Supplemental Groups Strategy: RunAsAny    
    Ranges:                    <none>
```

In the case of `MustRunAs`it can only use a value in a range. This range can be specified in the SCC, or it would be taken from the `NameSpace` related SCC annotations:

`supplemental-groups`:

```bash
> oc get ns ${NAMESPACE} -o jsonpath='{.metadata.annotations.openshift\.io\/sa\.scc\.supplemental-groups}'
1000710000/10000
```

`mcs` multi-category security:

```bash
> oc get ns ${NAMESPACE} -o jsonpath='{.metadata.annotations.openshift\.io\/sa\.scc\.mcs}'
s0:c27,c4
```

Other attributes like `RunAsUser`have other strategies: `MustRunAs`, `MustRunAsRange`, `MustRunAsNonRoot`, `RunAsAny`. But similar behavior. For example, the default range of uids for users are also part of a NameSpace annotations:

```bash
> oc get ns ${NAMESPACE} -o jsonpath='{.metadata.annotations.openshift\.io\/sa\.scc\.uid-range}'
1000710000/10000
```

The range can be changed in the SCC applied to the SA that will run a Pod:

```bash
> oc patch scc restricted-runasuser \
-p '{"runAsUser":{"uidRangeMax":2500,"uidRangeMin":2000}}' \
--type merge
```

### SCC review and priorities

SCC review allows to find which SCC(s) can be used for a Pod.

Take this pod as an example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-scc-1
spec:
  serviceAccountName: testuser
  containers:
  - image: quay.io/mavazque/reversewords:latest
    name: test-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

And lets check the SCCs considering the `testuser` SA:

```bash
> oc -n ${NAMESPACE} create sa testuser
serviceaccount/testuser created
> oc -n ${NAMESPACE} policy scc-review -z system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-1.yaml 
RESOURCE   SERVICE ACCOUNT   ALLOWED BY   
```

> -z force to test with an specific ServiceAccount, no matter what is in the field `serviceAccountName`
> 
> if no ServiceAccount is specified, it will take the `serviceAccountName`
> 
> If neither ServiceAccount nor `serviceAccountName`, it will use default SA
> 
> You cannot check with an user

Empty, but if we create the Pod:

> **NOTE**: All authenticated users have access to the `restricted-v2` SCC, since it's the default one. scc-review command doesn't take into account access to SCCs inherited via a user group, such as `system:authenticated`.
> This is ok, because it only returns you the SCCs explicitly added to the SA, not considering who hypothetically will request the Pod execution.  

```bash
> oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    openshift.io/scc: privileged
```

Why Privileged? Remember that SCCs availability depends on User and SA. Our recent just created SA has none SCC assigned. But, we are running as Admin, which has more SCCS. 

Now, you can create the Pod because of running as Admin. But this will not always be true. And it depends on how you make your deployments. Also, privileged is too much for this Pod.

Now, lets create the Pod, but instead of as an Admin, as the SA:

```bash
> oc -n ${NAMESPACE} adm policy add-role-to-user edit system:serviceaccount:${NAMESPACE}:testuser
clusterrole.rbac.authorization.k8s.io/edit added: "system:serviceaccount:scc-fun:testuser"
> oc -n ${NAMESPACE} create -f /tmp/pod-scc-1.yaml --as=system:serviceaccount:${NAMESPACE}:testuser
pod/pod-scc-1 created
> oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    openshift.io/scc: restricted-v2
```

What is happening? Now we are no Admin, ww are the SA testuser.  As authenticated, it will receive the SCC `restricted-v2` by default. 

Now, we will add to SA the anyuid SCC:

```bash
> oc -n ${NAMESPACE} adm policy add-scc-to-user anyuid system:serviceaccount:${NAMESPACE}:testuser
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "testuser"
```

If we do a scc-review we will see only this last one. Remember that it does not show inherited ones, like the ones by the authenticated group (default):

```bash
> oc -n ${NAMESPACE} policy scc-review -z system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-1.yaml 
RESOURCE        SERVICE ACCOUNT   ALLOWED BY   
Pod/pod-scc-1   testuser          anyuid 
```

But we know, that when we make the request as user testuser, it will also have the `restricted-v2`

```bash
> oc -n ${NAMESPACE} create -f /tmp/pod-scc-1.yaml --as=system:serviceaccount:${NAMESPACE}:testuser
pod/pod-scc-1 created
> oc -n ${NAMESPACE} get pod pod-scc-1 -o yaml | grep "openshift.io/scc"
    openshift.io/scc: anyuid
```

Why is using `anyuid` instead of `restricted-2`. It just a matter of priorities:

```bash
> oc get scc anyuid restricted-v2
NAME            PRIV    CAPS                   SELINUX     RUNASUSER        FSGROUP     SUPGROUP   PRIORITY     READONLYROOTFS   VOLUMES
anyuid          false   <no value>             MustRunAs   RunAsAny         RunAsAny    RunAsAny   10           false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
restricted-v2   false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsRange   MustRunAs   RunAsAny   <no value>   false            ["configMap","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
```

### SCC and capabilities

As we have learn, one of the most important security concern is about capabilities. With Openshift, your allowed capabilities depend on who runs the Pod (User+SA) and their SCCs. 

By default, the `restricted-v2`have the following capabilities:

```yaml
defaultAddCapabilities: null
allowedCapabilities:
- NET_BIND_SERVICE
requiredDropCapabilities:
- ALL
```

Only NET_BIND_SERVICE, that allows you to bind a process to a port.

About the runAsUser for the Container, this SCC tell us:

```yaml
runAsUser:
  type: MustRunAsRange
```

It has to be in a range of [1000700000, 1000709999]

So we would run this kind of pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-80-2
spec:
  serviceAccountName: testuser
  containers:
  - image: quay.io/mavazque/reversewords:latest
    name: reversewords
    env:
    - name: APP_PORT
      value: "8080"
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

We are not specifying an uncorrect runAsUser (lets Openshift assign a proper one). We are not even asking for Capabilities.

```bash
> oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-capabilities.yaml 
pod/reversewords-app-captest-80-2 created
> oc -n ${NAMESPACE} logs pod/reversewords-app-captest-80-2
2023/02/18 18:03:52 Starting Reverse Api v0.0.25 Release: NotSet
2023/02/18 18:03:52 Listening on port 8080
```

We could bind to the port 8080, because the Pod is running with the default SCC `restricted-v2` which allows that bind:

```bash
> oc -n ${NAMESPACE} get pod reversewords-app-captest-80-2 -o yaml | grep "openshift.io/scc"
    openshift.io/scc: restricted-v2
```

Now, we will try a privileged Port:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-80-2
spec:
  serviceAccountName: testuser
  containers:
  - image: quay.io/mavazque/reversewords:latest
    name: reversewords
    env:
    - name: APP_PORT
      value: "80"
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

And:

```bash
> oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-capabilities.yaml 
pod/reversewords-app-captest-80-2 created

> oc -n ${NAMESPACE} logs pod/reversewords-app-captest-80-2
2023/02/18 18:06:34 Starting Reverse Api v0.0.25 Release: NotSet
2023/02/18 18:06:34 Listening on port 80
2023/02/18 18:06:34 listen tcp :80: bind: permission denied
```

We cannot bind on Port 80 running with the user:

```bash
> oc -n ${NAMESPACE} get pod reversewords-app-captest-80-2 -o yaml | grep runAsUser
      runAsUser: 1000700000
```

So, lets try with runAsUser=0

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-80-2
spec:
  serviceAccountName: testuser
  containers:
  - image: quay.io/mavazque/reversewords:latest
    name: reversewords
    env:
    - name: APP_PORT
      value: "80"
    securityContext:
      runAsUser: 0
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

```bash
> oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-capabilities.yaml 
Error from server (Forbidden): error when creating "/tmp/pod-scc-capabilities.yaml": pods "reversewords-app-captest-80-2" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, spec.containers[0].securityContext.runAsUser: Invalid value: 0: must be in the ranges: [1000700000, 1000709999], provider "restricted": Forbidden: not usable by user or serviceaccount, provider "nonroot-v2": Forbidden: not usable by user or serviceaccount, provider "nonroot": Forbidden: not usable by user or serviceaccount, provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, provider "machine-api-termination-handler": Forbidden: not usable by user or serviceaccount, provider "hostnetwork-v2": Forbidden: not usable by user or serviceaccount, provider "hostnetwork": Forbidden: not usable by user or serviceaccount, provider "node-exporter": Forbidden: not usable by user or serviceaccount, provider "privileged": Forbidden: not usable by user or serviceaccount]
```

Our current SCC dont allow us to run as whatever user we want. Only the users on the range above.

So, we have to create a new SCC. Similar to the restricted one, but allowing to run as root:

```yaml
kind: SecurityContextConstraints
metadata:
  name: restricted-netbind
priority: 1
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities:
- NET_BIND_SERVICE
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: MustRunAs
groups: []
```

```bash
> oc -n ${NAMESPACE} adm policy add-scc-to-user restricted-netbind system:serviceaccount:${NAMESPACE}:testuser
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:restricted-netbind added: "testuser"
```

Now we can create the previous Pod as user 0:

```bash
> oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-capabilities.yaml 
pod/reversewords-app-captest-80-2 created
> oc -n ${NAMESPACE} logs pod/reversewords-app-captest-80-2                                                  
2023/02/18 18:12:29 Starting Reverse Api v0.0.25 Release: NotSet
2023/02/18 18:12:29 Listening on port 80
```

and it can bind to port 80.

So, we have (as testuser) this capability. But the Pod would require to drop it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-80-2
spec:
  serviceAccountName: testuser
  containers:
  - image: quay.io/mavazque/reversewords:latest
    name: reversewords
    env:
    - name: APP_PORT
      value: "80"
    securityContext:
      runAsUser: 0
    securityContext:
      capabilities:
        drop:
        - NET_BIND_SERVICE
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

You can do it, you have these privileges:

```bash
> oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-capabilities.yaml 
pod/reversewords-app-captest-80-2 created
> oc -n ${NAMESPACE} logs pod/reversewords-app-captest-80-2
2023/02/18 18:17:14 Starting Reverse Api v0.0.25 Release: NotSet
2023/02/18 18:17:14 Listening on port 80
2023/02/18 18:17:14 listen tcp :80: bind: permission denied
```

Maybe, this example makes no sense. But more complex examples, a good Pod definition would require to Drop not needed Capabilities. For security reasons. Even if you have many capabilities available, you dont want to use it, and you dont want your process to have them effective. It is a power that you dont want to have.

The other way around, your Pod would require a list of whatever number of capabilities:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: reversewords-app-captest-80-2
spec:
  serviceAccountName: testuser
  containers:
  - image: quay.io/mavazque/reversewords:latest
    name: reversewords
    env:
    - name: APP_PORT
      value: "80"
    securityContext:
      runAsUser: 0
    securityContext:
      capabilities:
        add:
        - NET_BIND_SERVICE
        - KILL
  dnsPolicy: ClusterFirst
  restartPolicy: Never
```

Here, again, is maybe not a good example. We know we dont need `KILL`capabilitie. But it is anyway requiered:

```bash
> oc -n ${NAMESPACE} create --as=system:serviceaccount:${NAMESPACE}:testuser -f /tmp/pod-scc-capabilities.yaml 
Error from server (Forbidden): error when creating "/tmp/pod-scc-capabilities.yaml": pods "reversewords-app-captest-80-2" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, spec.containers[0].securityContext.capabilities.add: Invalid value: "KILL": capability may not be added, provider "restricted": Forbidden: not usable by user or serviceaccount, provider "nonroot-v2": Forbidden: not usable by user or serviceaccount, provider "nonroot": Forbidden: not usable by user or serviceaccount, provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, provider "machine-api-termination-handler": Forbidden: not usable by user or serviceaccount, provider "hostnetwork-v2": Forbidden: not usable by user or serviceaccount, provider "hostnetwork": Forbidden: not usable by user or serviceaccount, spec.containers[0].securityContext.capabilities.add: Invalid value: "NET_BIND_SERVICE": capability may not be added, provider "node-exporter": Forbidden: not usable by user or serviceaccount, provider "privileged": Forbidden: not usable by user or serviceaccount]
```

And it will fail, because the required `KILL` capability is not included in any of our available SCCs. 

Running as Admin, of course, solves the problem (running with SCC `privileged`=

```bash
> oc -n ${NAMESPACE} create  -f /tmp/pod-scc-capabilities.yaml 
pod/reversewords-app-captest-80-2 created
> oc -n ${NAMESPACE} get pod/reversewords-app-captest-80-2 -o yaml | grep "openshift.io/scc"
    openshift.io/scc: privileged
```

### Other SCC attributes

We have only seen how the Linux Capabilities can be managed on a SCC, but there are many other security aspects you can manage. Here, a very quick overview:

* fsGroup ID: it is used for block storage, to CHWON the content of that storage to that GroupID. As explained before, this attribute strategy defines which values can be used: `MustRunAs` in a range or `RunAsAny`
  
  ```yaml
      spec:
        securityContext:
          fsGroup: 6005
  ```
  
  Inside the pod, depending on which `securityContext.runAsUser`value and strategy, we will see our user belonging to its default group, plus, the `fsGroup`:
  
  ```bash
  > oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- id
  uid=1000710000(1000710000) gid=0(root) groups=0(root),6005
  ```
  
  This pod was also mounting a `Volume`:
  
  ```yaml
          volumeMounts:
            - name: test-volume
              mountPath: "/mnt"
  ```
  
  Inside the Pod, `/mnt` belongs to the group:
  
  ```bash
  > oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- touch /mnt/testfile
  > oc -n ${NAMESPACE} exec -ti deployment/reversewords-app-storage -- ls -lrt /mnt/
  total 0
  drwxrws---. 3 root       6005 17 Feb 20 12:52 systemd-private-8711221dc49143cc9d3c7cca577db462-chronyd.service-z5pxzt
  -rw-rw-rw-. 1 1000710000 6005  0 Feb 24 18:13 testfile
  ```

* seLinuxContext:  it controls which SELinux context is used by the containers. In general, it is set with an strategy `MustRunAs`, which make all the Containers in a Pod, to use the MCS ([Multi-Category Strategy](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/assembly_using-multi-category-security-mcs-for-data-confidentiality_using-selinux)) labels:
  
  ```bash
  > oc get ns ${NAMESPACE} -o yaml | grep "sa.scc.mcs"
      openshift.io/sa.scc.mcs: s0:c27,c4
  ```

* seccompProfiles: explained above how these are configured in Openshift. Here, you can list which ones can be used. If not set (empty, nill), none can be used. It can be used * to allow all available ones. 
  
  Remember these are stored, physically, as files in all the servers that would execute the Pod.

* AllowPrivilegeEscalation: [TODO]

* Others

# Pod Security Admission

*Feature stable on Kubernetes 1.25 and Openshift 4.11*

It is an admission controller to enforce the **Pod Security Standards**. Pod security restrictions are applied at Namespace level. 

There are three levels of **Pod Security Standards**: `privileged`, `baseline`, or `restricted`. Each one, with different security restrictions.

Then you have modes that specify how to act when this security policies are not satisfied: 

| **enforce** | Policy violations will cause the pod to be rejected.                                                                                                                                                 |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **audit**   | Policy violations will trigger the addition of an audit annotation to the event recorded in the [audit log](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/), but are otherwise allowed. |
| **warn**    | Policy violations will trigger a user-facing warning, but are otherwise allowed.                                                                                                                     |

You can configure different modes depending on the security policies:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.26

    # We are setting these to our _desired_ `enforce` level.
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.26
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.26
```

If the `baseline` policy is not satisfied (which is a intermediate policy) the Pod will not be created. 

When no satisfied some of the restrictions on for the `restricted`policy, there will be a warning, and an audition annotation will be added to the Pod. 

In this way, we require to satisfy at least the `baseline`level. The `restricted`one is recommended, so, if not satisfied, you will have some annotations and warnings.

The `*-version` is important. Because the Pod security standards will change and evolve in different versions. The conditions for `restricted`will be different depending on the Kubernetes version. 

This would be the default configuration of the admission controller: 

```yaml
apiVersion: apiserver.config.k8s.io/v1 # see compatibility note
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    # Defaults applied when a mode label is not set.
    #
    # Level label values must be one of:
    # - "privileged" (default)
    # - "baseline"
    # - "restricted"
    #
    # Version label values must be one of:
    # - "latest" (default) 
    # - specific version like "v1.26"
    defaults:
      enforce: "privileged"
      enforce-version: "latest"
      audit: "privileged"
      audit-version: "latest"
      warn: "privileged"
      warn-version: "latest"
    exemptions:
      # Array of authenticated usernames to exempt.
      usernames: []
      # Array of runtime class names to exempt.
      runtimeClasses: []
      # Array of namespaces to exempt.
      namespaces: []
```

Notice, you can configure some exemptions for Users, Namespaces and runtimeClasses.

This is a Namespace example configuration:

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: test-namespace
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: v1.25
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.25
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.25
```

It will only warn (and audit) if something is wrong about `restricted`security standard. And it will make mandatory to satisfy the `privileged`security standard, or the Pod will not be admitted.

## PSA and Openshift

Now we will run some examples, but instead of just a Kubernetes cluster, we will use an OCP 4.12

But in OCP we already have something similar, the SCCs, that configures what you can or cannot do. In OCP 4.11 (and above) a new controller configures and synchs the PSA standards according to the SCCs. *In OCP 4.11 this controller modifies the `warn` and `audit` modes* 

*The way this controller works is by introspecting the service account 
permissions to use SCCs and maps these SCCs to Pod Security Standards. 
Based on the SCC configuration the controller will set the value of the 
different PSA modes that allow the workloads to run in the namespace 
without triggering warnings or audit logs.*

So, this OCP controller synchs the default PSA Mode values, on each Namespace, depending on the SCC of the acting User/SA.

Here, a default Namespace on OCP4.12:

```bash
> oc get ns test-psa -o yaml | grep pod-security
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
```

This simple Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: go-app
  name: go-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-app
  strategy: {}
  template:
    metadata:
      labels:
        app: go-app
    spec:
      containers:
      - image: quay.io/mavazque/reversewords:latest
        name: reversewords
        resources: {}
```

The Deployment is created and it will print out some warnings.

```bash
Warning: would violate PodSecurity "restricted:v1.24": 
allowPrivilegeEscalation != false 
(container "reversewords" must set securityContext.allowPrivilegeEscalation=false), 
unrestricted capabilities 
(container "reversewords" must set securityContext.capabilities.drop=["ALL"]), 
runAsNonRoot != true 
(pod or container "reversewords" must set securityContext.runAsNonRoot=true), 
seccompProfile 
(pod or container "reversewords" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/go-app created
```

The `restricted` policy of warns you about:

* `allowPrivilegeEscalation != false` 

* All the capabilities have to be dropped

* `runAsNonRoot != true`

* seccompProfile has to be "RuntimeDefault" or "Localhost"

It is really interesting how it wans you about security issues that you are not considering, and it tells you how to solve it.

How to consider this complains? Well, you can see this as good practices. 

For example, it encourages you (because we are audit/wan, but it could require) to drop all, even if you will not use any. Actually, the controller dont know if you will use any. But, it is a good practice to drop all (and enable only needed ones)

Other example, it tells you to specifically configure to run as non root. Even, if later it would run as any other. Like it is the case:

```bash
> oc -n test-psa get pod go-app-5b954b7b74-z99qx -o yaml | grep -i user
      runAsUser: 1000880000
```

Remember we are using here Openshift, so SCCs will determinate that. This would change in other runtimes, and the Admission controlller acts at Kubernetes level.

```bash
> oc -n test-psa get pod go-app-5b954b7b74-z99qx -o yaml | grep scc
    openshift.io/scc: restricted-v2
> oc get scc restricted-v2 -o yaml | grep runAsUser -A 1
runAsUser:
  type: MustRunAsRange
```

But the Admission controller acts accepting (or not) the Pod creation. The user will be assigned after this controller. So, the PSA does a god job warning (or requiring) to specifically configure your Pods to not use Root user.

There are other warnings in our example that can be easily considered.

## Openshift and the PSA Controller

Back to the Namespace and the controller that synchs the PSA values:

```bash
> oc get ns test-psa -o yaml | grep pod-security
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
```

We see it is selected the Policy Security Standard as `restricted`. This is because we have one SCC which is actually `restricted-v2`, as default.  This is done to have a direct (or similar) match. 

Now we will change that, creating a SA with the `privileged` SCC:

```bash
> oc -n psa-auto-modes create serviceaccount test-user
serviceaccount/test-user created

> oc -n psa-auto-modes adm policy add-scc-to-user privileged -z test-user
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "test-user"
```

The `privileted` SCC has a direct synch with the `privileged`PSS (Policy Security Standard)

```bash
> oc get namespace psa-auto-modes -o yaml | grep pod-security
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: privileged
    pod-security.kubernetes.io/warn-version: v1.24
```

Other SCC will also have its match. We remove `privileged`SCC and assign other different one:

```bash
> oc -n psa-auto-modes adm policy remove-scc-from-user privileged -z test-user
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged removed: "test-user"
> oc -n psa-auto-modes adm policy add-scc-to-user anyuid -z test-user
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "anyuid"


> oc get namespace psa-auto-modes -o yaml | grep pod-security
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: v1.24
```

So, the `anyuid`SCC is synched as `baseline` PSA. 

OCP have SCC which directly match to `restricted`and `privileged` PSAs. Other SCC will find the maximum similarity. 

Now, I will create a new SCC, which is basically the same as `privileged` with only one filed modified. 

```bash
> oc get scc | grep invented
invented-one                      true    ["*"]                  RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   true             ["*"]
```

Lets give the user, the new SCC (which is 99% the `privileged`)

```bash
> oc -n psa-auto-modes adm policy remove-scc-from-user anyuid -z test-user
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid removed: "test-user"
> oc -n psa-auto-modes adm policy add-scc-to-user invented-one -z test-user
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:invented-one added: "test-user"
```

Even if it is not exactly the same as the `privileged` SCC, the controller still detects that the PSS that better fits is `privileged`. So, it is synched to `privileged` PSS:

```bash
> oc get namespace psa-auto-modes -o yaml | grep pod-security
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/audit-version: v1.24
    pod-security.kubernetes.io/warn: privileged
    pod-security.kubernetes.io/warn-version: v1.24
```

Just one SA with an SCC applied will make the synch to happen. 

This synch can be disabled, having only the default values:

```bash
> oc label namespace psa-manual-modes security.openshift.io/scc.podSecurityLabelSync=false
```

## PSA and SCC

PSA is an admission controller, like a filter that will act at Pod specification. Actually, at `spec.securityContext`. This is independent on the privileges that the User/SA, running the Pod, has.
The Openshift SCC defines User/SA privileges, capabilities, etc.  PSA are security standards independently of your privileges and perms. And this can be considered as recommendations, or obligations. 

An SCC would give to you user the power or running Pods as Root. But this would be avoided by one PSA.