# Finding last commit of ACM builds

Similar to this [gist](https://gist.github.com/jgato/06bd4cf7ab1946acdc011763e0dbf9d5) to get Assisted Service running version and included commits, we want to find out ACM last commits.

ACM is composed by many operators, so, having an installed one, you can get the Manifests for the installation:

```bash
> oc -n open-cluster-management get cm | grep manifest
mch-image-manifest-2.10.4           48     89d
mch-image-manifest-2.11.0           47     88d
mch-image-manifest-2.11.1           47     75d
mch-image-manifest-2.9.3            46     137d
mch-image-manifest-2.9.4            46     131d

```

in this case I did several upgrades, so I could find out info for many versions, which is cool.

For example 2.10.4 and the Observability operator:

```bash
> oc -n open-cluster-management get cm mch-image-manifest-2.10.4 -o yaml | grep -i observability
  multicluster_observability_operator: registry.redhat.io/rhacm2/multicluster-observability-rhel9-operator@sha256:496c017a09ce7f00c5fea254e423e95ec35741ff2e4f49d77fadb3a67e072fb5

```

once we have the container image, we can use skope to get more information:

```bash
> skopeo inspect docker://registry.redhat.io/rhacm2/multicluster-observability-rhel9-operator@sha256:496c017a09ce7f00c5fea254e423e95ec35741ff2e4f49d77fadb3a67e072fb5 | grep -E 'version|upstream-ref'
        "io.buildah.version": "1.29.0",
        "upstream-ref": "27e6b21f7170151f3226c0290d55c25087007757",
        "version": "v2.10.4"

```

So, we are sure that is `v2.10.4` and we have the reference commit: [27e6b21f7170151f3226c0290d55c25087007757](https://github.com/stolostron/multicluster-observability-operator/commit/27e6b21f7170151f3226c0290d55c25087007757). 

If we want to ensure that a commit, that for example fixes a bug, was also included, clone the operator repo and:

```bash
> git status
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean

> git fetch 
remote: Enumerating objects: 399, done.
remote: Counting objects: 100% (394/394), done.
remote: Compressing objects: 100% (215/215), done.
remote: Total 399 (delta 252), reused 299 (delta 174), pack-reused 5 (from 1)
Receiving objects: 100% (399/399), 289.99 KiB | 886.00 KiB/s, done.
Resolving deltas: 100% (252/252), completed with 25 local objects.
From https://github.com/open-cluster-management/multicluster-observability-operator
 * [new branch]        am-test                                               -> origin/am-test
 + b510d59b...6ee4d368 appstudio-multicluster-observability-operator-acm-211 -> origin/appstudio-multicluster-observability-operator-acm-211  (forced update)
 * [new branch]        fix_concurrent_map                                    -> origin/fix_concurrent_map
 * [new branch]        philipgough-patch-1                                   -> origin/philipgough-patch-1
   42b63874..ca69deeb  release-2.10                                          -> origin/release-2.10
   83411fe1..3de380de  release-2.11                                          -> origin/release-2.11
 * [new branch]        release-2.12                                          -> origin/release-2.12
 * [new branch]        release-2.12-bak                                      -> origin/release-2.12-bak
   442f7148..542db57c  release-2.9                                           -> origin/release-2.9
 * [new branch]        update201                                             -> origin/update201

```

then:

```bash
> git branch -a --contains 27e6b21f7170151f3226c0290d55c25087007757 
  remotes/origin/fix_concurrent_map
  remotes/origin/release-2.10
  remotes/origin/update201
```

Now, we know in which branch the last commit in our build was included. In this case, maybe, it was easy to guess it. But it is good to check. So, we take this brunch and print all the commits that were done from our build commit:

```bash
> git rev-list 27e6b21f7170151f3226c0290d55c25087007757..remotes/origin/release-2.10 --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(cyan)<%an>%Creset' --abbrev-commit --date=relative 
commit ca69deeb
ca69deeb - (origin/release-2.10) [release-2.10] Make sure local cluster is always in the managedcluster list (#1645) (2 weeks ago) <Coleen Iona Quadros>
commit 77b689af
77b689af - fix: [release-2.10] [ACM-14189]identify healthy managed clusters (#1636) (3 weeks ago) <Subbarao Meduri>
commit 5e25f0de
5e25f0de - [release-2.10] ACM-13706: Stop CMO update if not changed (#1632) (4 weeks ago) <OpenShift Cherrypick Robot>
commit 447840f9
447840f9 - sort metrics args (#1619) (5 weeks ago) <Thibault Mange>
commit 307c7ef7
307c7ef7 - [ACM-14235]: Ensure CMO ConfigMap is reconciled on any event (#1616) (5 weeks ago) <Philip Gough>
commit 39452478
39452478 - fix(KONFLUX-3663): format PipelineRun files and upload SAST results (#1577) (8 weeks ago) <Camilo Cota>
commit 850687a7
850687a7 - Update Konflux references (#1571) (8 weeks ago) <red-hat-konflux[bot]>
commit a9524dd9
a9524dd9 - add `-y` to `microdnf update` to unstuck builds (#1583) (8 weeks ago) <OpenShift Cherrypick Robot>
commit 0e376aec
0e376aec - Fix dnf install to prevent interactive confirmation (#1582) (8 weeks ago) <Jacob Baungård Hansen>
commit 779ba008
779ba008 - Update Konflux references to 20cdb35 (#1570) (9 weeks ago) <red-hat-konflux[bot]>
commit 0c4adffe
0c4adffe - Update Konflux references (release-2.10) (#1459) (2 months ago) <red-hat-konflux[bot]>
commit 1476a52f
1476a52f - (origin/update201) Swap out cmo dependency for local structs (#1548) (2 months ago) <Philip Gough>
commit 70d0d866
70d0d866 - Watch CMO cluster monitoring ConfigMap for changes (#1560) (3 months ago) <Philip Gough>
commit 768817f2
768817f2 - chore(deps): update registry.access.redhat.com/ubi8/go-toolset docker digest to 00a6449 (#1514) (3 months ago) <red-hat-konflux[bot]>
commit aa8775a7
aa8775a7 - Adding rw mutexes for RBAC query proxy enforcement (#1556) (3 months ago) <Philip Gough>
commit dd4116c3
dd4116c3 - Update Konflux references (#1536) (3 months ago) <red-hat-konflux[bot]>

```

In my case, I am looking for a fix for [Backport MCO reconcile issues to 2.10](https://github.com/stolostron/multicluster-observability-operator/pull/1496). That does not appear there, so, most likely happened before and it is included:

```bash
> git log ddef4d94de6365cd3da2f42dcbc9ced94c4c8254 --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(cyan)<%an>%Creset' --abbrev-commit -1
ddef4d94 - Backport MCO reconcile issues to 2.10 (#1496) (4 months ago) <Coleen Iona Quadros>
> git log 27e6b21f7170151f3226c0290d55c25087007757 --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(cyan)<%an>%Creset' --abbrev-commit -1
27e6b21f - *ks: correctly define cluster:node_cpu:ratio rule (#1521) (3 months ago) <OpenShift Cherrypick Robot>

```

yes, the commit with the fix happened 1 month before. 