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

In this case, instead of adding or giving capabilities, you fine tune which syscalls a process, can, or cannot, do. It is like a filter of syscalls. One capability would be affected by more than one syscalls. Thefore, Seccomp are more complex to create, and it requires a deep knowledge of what your code do, at syscalls level.

To get a Seccomp profile about a running process, you can do:

```bash
> sudo podman run --rm --annotation io.containers.trace-syscall="of:/tmp/ls.json" fedora:32 ls / > /dev/null

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

Privileged SCC can be used by an Admin, or the build-controller SA on the openshift-infra NameSpace. A SA is composed by:  system:serviceaccount:namespace:name. 

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



### Custom SCC and capabilities

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
