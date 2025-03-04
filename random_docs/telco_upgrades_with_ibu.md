# Upgrading clusters with Image Base Upgrade (IBU)

In this blog, I will focus on how to proceed on upgrading the clusters of your infrastructure, managed by Red Hat Advanced Cluster Management (RHACM). More in concrete, using the new upgrade mechanism called Image Base Upgrade.

Previously, you could upgrade your infrastructure, just using the Openshift usual upgrade way:
![](assets/telco_upgrades_with_ibu_20250130155334523.png)

Or, just using the `oc cli`, [more in the official documentation](https://docs.openshift.com/container-platform/4.16/updating/updating_a_cluster/updating-cluster-cli.html).

Both methods trigger the same procedure, that it will update the operative system, Openshift, operators, etc. For a Single Node Openshift (SNO) this time would vary depending on the configuration. But it would be around 60/70 minutes.

When we move to telecommunications scenarios, SNOs are designed to run the Telco Radio Access Network. You can think on the software managing every antenna, and your infrastructure would be composed by thousands of antennas that you need upgrade. The process is done during a maintenance window, and the requirements on time are super strict. 

Therefore, IBU comes to solve this scenario, providing a mechanisms that provides upgrades on about 25 minutes. IBU is based on creating an image from a called "seed" clusters. All the clusters in your infrastructure, that can be considered as clones of this seed cluster, could be upgraded by this image. So, the mechanism is very good on this telco RAN scenarios, very homogeneous and composed by SNO. But IBU is not suitable for multi node clusters, or heterogeneous infrastructures. Actually, IBU contains some pre-checks related to configurations for telco RAN.

During this blog I will briefly cover how this new upgrade process works. But, I will not go into detail on how configure, install, deploy your infrastructure. The starting scenario is composed by three SNOs, already installed, configured and managed by ACM.
![](assets/telco_upgrades_with_ibu_20250130165329318.png)

Notice how all these are based on Openshift 4.14, and we want to upgrade to 4.16. Other advantage of IBU, is that we can move directly to 4.16. Not needing to go through 4.15 first (1 hour) and then 4.16 (1 hour.).

There is a fourth SNO4 cluster that will be used as seed clusters. All the clusters are based on the same hardware, software, and network configuration.

## Using the seed cluster to create the upgrade image

*The whole process I am explaining is more detailed [here](https://docs.openshift.com/container-platform/4.16/edge_computing/image_based_upgrade/cnf-understanding-image-based-upgrade.html)*

The seed cluster is a kind of clone cluster that contains the software and desired version. In this case, SNO4 has been deployed with Openshift 4.16, that is the desired version to upgrade. But it contains the same hardware and network than the others.

The seed cluster is something that should be considered as en ephemeral cluster. It should be installed, configured, created the seed and disappear. It does not contain any extra workload, or, these will be received by the upgrade clusters later. The seed cluster is not a cluster that has been running for a time. Otherwise, we would create an image not so clean as expected. 

If the seed cluster is part of ACM (or ZTP), it should be first detached. To create an image not containing workloads related to ACM.

Apart from the usual Openshift installation, and the RAN configurations (that we don cover in this blog), it needs two extra operators:
 * [Operator life-cycle agent](https://docs.openshift.com/container-platform/4.16/edge_computing/image_based_upgrade/preparing_for_image_based_upgrade/cnf-image-based-upgrade-install-operators.html#cnf-image-based-upgrade-installing-lifecycle-agent-using-cli_install-operators): that trigger the image creation.
 * [OADP](https://docs.openshift.com/container-platform/4.16/backup_and_restore/application_backup_and_restore/installing/oadp-installing-operator.html#oadp-installing-operator-doc) (OpenShift APIs for Data Protection) for backups management. The seed cluster does not do any backup, but upgraded clusters will do a restore of their own workloads. So, the operator is packaged with the image. When the clusters are upgraded, they will contain this operator, that will be used to trigger a restore of the previous running workloads. 
 
Check the official documentation of each operator about how to do the installation. But, it is just an usual Openshift operator installation.

After that, we have to trigger the seed creation. First, we create a secret with the authentication details for the container registry to store the image:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: seedgen
  namespace: openshift-lifecycle-agent
type: Opaque
data:
  seedAuth: <jgato_quay_authentication>
```

In my case I use quay.io and the `seedAuth` is base64 encoded of a json similar to:

```json
{
	"auths": {
		"quay.io/jgato": {
			"auth": "amdhdG9......FuX0c2bmE="
		}
	}
}
```

Now, we trigger the seed creation with:

```yaml
apiVersion: lca.openshift.io/v1
kind: SeedGenerator
metadata:
  name: seedimage
spec:
  seedImage: quay.io/jgato/sno4:4.16.9
```

We can observe the image creation:
```bash
$ oc create -f seedgenerator.yaml  && oc get seedgenerators.lca.openshift.io  -w
seedgenerator.lca.openshift.io/seedimage created
NAME        AGE   STATE   DETAILS
seedimage   0s            
seedimage   0s    SeedGenInProgress   Waiting for system to stabilize
seedimage   2s    SeedGenInProgress   Starting seed generation
seedimage   2s    SeedGenInProgress   Pulling recert image
seedimage   7s    SeedGenInProgress   Preparing for seed generation
seedimage   8s    SeedGenInProgress   Cleaning cluster resources
seedimage   80s   SeedGenInProgress   Launching imager container
seedimage   80s   SeedGenInProgress   Launching imager container

```

In this moment, kubelet is stoped, and a container is created (outside Openshift) to create the image. After a while, kukbelet will be restarted and:

```bash
$ oc get seedgenerators.lca.openshift.io  -w
NAME        AGE   STATE              DETAILS
seedimage   21s   SeedGenCompleted   Seed Generation completed

```

The image has been generated and upload it to my quay.io.

![](assets/telco_upgrades_with_ibu_20250130172132409.png)

## Upgrading clusters
### Preparing the backup

As a difference of a seed cluster, the cluster to be upgrade is an operational cluster and it would be running whatever workload. These extra workloads will be included on a backup (by OADP), and recovered after the upgrade. A part from that, all the clusters are practically the same.

As an example, I have deployed a simple workload on the namespace `example-workload`. That also is making usage of a PersistentVolume provided by the LocalStorageOperator. This works as an example of backup process, and restore. Remember that the seed image tries to be as clean as possible. It is in your side to backup your workloads, pv, roles, and whatever needed CRD.

```bash
> oc -n example-workload get deployment,pod,pvc
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/exception-app-deployment   1/1     1            1           56s

NAME                                            READY   STATUS    RESTARTS   AGE
pod/exception-app-deployment-7c9ff94dd9-c52x2   1/1     Running   0          57s

NAME                           STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/my-pvc   Bound    local-pv-4ad70ba3   1Gi        RWO            general        13m

```

The Pod is just simulating some exceptions (just an example):

```bash
> oc -n example-workload logs exception-app-deployment-7c9ff94dd9-c52x2 
{"timestamp": "2025-01-31T09:16:18.080413Z", "level": "INFO", "message": "Running... Exception will be raised in 30 seconds.", "app": "exception-app"}
Traceback (most recent call last):
  File "/app/exception_app.py", line 36, in cause_complex_exception
    level_one()
  File "/app/exception_app.py", line 28, in level_one
    level_two()
  File "/app/exception_app.py", line 31, in level_two
    level_three()
  File "/app/exception_app.py", line 34, in level_three
    raise Exception("Custom exception at level three")
Exception: Custom exception at level three

```

Lets see how continue working after upgrade, and restore the workload.

Preparing the backup, it depends on the SNO, and different options on operators and storage. I am not covering this on this blog to not make it too dense. But you can find the details [here](https://docs.openshift.com/container-platform/4.16/edge_computing/image_based_upgrade/preparing_for_image_based_upgrade/cnf-image-based-upgrade-prep-resources.html). And I will focus on how to backup and specific workload.

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  labels:
    velero.io/storage-location: default
  name: backup-app
  namespace: openshift-adp
spec:
  includedNamespaces:
  - example-workload
  includedNamespaceScopedResources:
  - persistentvolumeclaims
  - deployments
  excludedClusterScopedResources:
  - persistentVolumes
---
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: test-app
  namespace: openshift-adp
  labels:
    velero.io/storage-location: default
  annotations:
    lca.openshift.io/apply-wave: "4"
spec:
  backupName:
    backup-app
```

This, and the not covered backups, dont need to be directly created on the cluster. These need to be included into a ConfigMap:

```bash
> oc create -n openshift-adp configmap oadp-cm-example \
 --from-file=backup-acm-klusterlet.yaml=backup-acm-klusterlet.yaml \
 --from-file=backup-workload.yaml=backup-workload.yaml
```

And we patch the ImageBaseUpgrade resource with the backups.

```bash
> oc patch imagebasedupgrade upgrade \
-p='{"spec": {"oadpContent": [{"name": "oadp-cm-example", "namespace": "openshift-adp"}]}}'   --type=merge 
```

### Triggering the backup
*The whole process I am explaining is more detailed [here](https://docs.openshift.com/container-platform/4.16/edge_computing/image_based_upgrade/cnf-image-based-upgrade-base.html)*

On all the backups receiving the upgrade, it has been installed the  Lifecycle Agent operator. This, will automatically create the ImageBaseUpgraded (that I already patched above with backups info) in charge of managing the upgrade. But also, informing about the current stage.

Initially we are in the `idle` stage:

```bash
$ oc get ibu upgrade
NAME      AGE   DESIRED STAGE   STATE   DETAILS
upgrade   18h   Idle            Idle    Idle

```

There are [other stages](https://docs.openshift.com/container-platform/4.16/edge_computing/image_based_upgrade/cnf-understanding-image-based-upgrade.html#cnf-image-based-upgrade_understanding-image-based-upgrade) that supports the logic of the whole Lifecycle Agent. 

Before moving to `pre` stage, we have to configure the `seedImageRef`.

```
apiVersion: lca.openshift.io/v1
kind: ImageBasedUpgrade
metadata:
  creationTimestamp: "2025-02-05T16:26:36Z"
  generation: 5
  name: upgrade
  resourceVersion: "225303"
  uid: 7b9ca970-b418-453e-8673-ba5be07c9622
spec:
  oadpContent:
  - name: oadp-cm-example
    namespace: openshift-adp
  seedImageRef:
    image: quay.io/jgato/sno4:4.16.9
    pullSecretRef:
      name: secret-pull-seed
    version: 4.16.9
  stage: Idle


```

The secret has been created cluster wide and contains a pullSecret to download the seed image:

```
apiVersion: v1
data:
  .dockerconfigjson: ewoJYXV0aHM6IHsKCQlxd....Qp9Cg==
kind: Secret
metadata:
  name:  secret-pull-seed
  namespace: openshift-lifecycle-agent
type: Opaque

```

Lets move to the `pre` stage:

```bash
$ oc patch imagebasedupgrades.lca.openshift.io upgrade -p='{"spec": {"stage": "Prep"}}' --type=merge -n openshift-lifecycle-agent
imagebasedupgrade.lca.openshift.io/upgrade patched
```

```bash
$ oc get ibu upgrade  -w
NAME      AGE   DESIRED STAGE   STATE        DETAILS
upgrade   17h   Prep            InProgress   Stateroot setup job in progress. job-name: lca-prep-stateroot-setup, job-namespace: openshift-lifecycle-agent
upgrade   17h   Prep            InProgress   Successfully launched a new job precache. job-name: , job-namespace: 
upgrade   17h   Prep            InProgress   Precache job in progress. job-name: lca-prep-precache, job-namespace: openshift-lifecycle-agent. No precache status file to read yet.
upgrade   17h   Prep            InProgress   Precache job in progress. job-name: lca-prep-precache, job-namespace: openshift-lifecycle-agent. total: 125 (pulled: 20, failed: 0)
upgrade   17h   Prep            InProgress   Precache job in progress. job-name: lca-prep-precache, job-namespace: openshift-lifecycle-agent. total: 125 (pulled: 40, failed: 0)
...
...
upgrade   17h   Prep            Completed    Prep stage completed successfully

```

Now, we are ready to do the upgrade:

```bash
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.14.40   True        False         17h     Cluster version is 4.14.40

$ date
Thu Feb  6 05:04:23 EST 2025
$ oc get ibu upgrade  -w
NAME      AGE   DESIRED STAGE   STATE        DETAILS
upgrade   17h   Upgrade         InProgress   Backup of Application Data is in progress
upgrade   17h   Upgrade         InProgress   Backing up Application Data
upgrade   17h   Upgrade         InProgress   Exporting Application Configuration
upgrade   17h   Upgrade         InProgress   Exporting Policy and Config Manifests
upgrade   17h   Upgrade         InProgress   Exporting Cluster and LVM configuration
upgrade   17h   Upgrade         InProgress   In progress

```

The SNO is rebooting. After that, in about 5 minute, you can see the node with the upgraded OCP version:

```bash
$ date
Thu Feb  6 05:10:56 EST 2025

$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.16.9    True        False         False      6d20h   
config-operator                            4.16.9    True        False         False      6d21h   
dns                                        4.16.9    True        False         False      6d20h   
etcd                                       4.16.9    True        False         False      6d21h   
ingress                                    4.16.9    True        False         False      6d21h   
kube-apiserver                             4.16.9    True        False         False      6d21h   
kube-controller-manager                    4.16.9    True        False         False      6d21h   
kube-scheduler                             4.16.9    True        False         False      6d21h   
kube-storage-version-migrator              4.16.9    True        False         False      6d21h   
machine-approver                           4.16.9    True        False         False      6d21h   
machine-config                             4.16.9    True        False         False      6d21h   
marketplace                                4.16.9    True        False         False      6d21h   
monitoring                                 4.16.9    True        False         False      6d20h   
network                                    4.16.9    True        True          False      6d21h   DaemonSet "/openshift-multus/network-metrics-daemon" is waiting for other operators to become ready...
node-tuning                                4.16.9    True        False         False      6d20h   
openshift-apiserver                        4.16.9    True        False         False      6d21h   
openshift-controller-manager               4.16.9    True        False         False      6d21h   
operator-lifecycle-manager                 4.16.9    True        False         False      6d21h   
operator-lifecycle-manager-catalog         4.16.9    True        False         False      6d21h   
operator-lifecycle-manager-packageserver   4.16.9    True        False         False      6d20h   
service-ca                                 4.16.9    True        False         False      6d21h  

```

But still some work to do.

```bash
$ oc get ibu upgrade  -w
NAME      AGE    DESIRED STAGE   STATE        DETAILS
upgrade   7m2s   Upgrade         InProgress   Waiting for system to stabilize: one or more health checks failed...
upgrade   7m28s   Upgrade         InProgress   Applying Policy Manifests
upgrade   7m28s   Upgrade         InProgress   Applying Config Manifests
upgrade   7m28s   Upgrade         InProgress   Restoring Application Data
upgrade   7m28s   Upgrade         InProgress   Restore of Application Data is in progress
upgrade   7m58s   Upgrade         InProgress   Applying Policy Manifests
upgrade   7m58s   Upgrade         InProgress   Applying Config Manifests
upgrade   7m58s   Upgrade         InProgress   Restoring Application Data
upgrade   7m58s   Upgrade         InProgress   Restore of Application Data is in progress
upgrade   8m28s   Upgrade         InProgress   Applying Policy Manifests
upgrade   8m28s   Upgrade         InProgress   Applying Config Manifests
upgrade   8m28s   Upgrade         InProgress   Restoring Application Data
upgrade   8m30s   Upgrade         InProgress   Restoring Application Data
upgrade   8m30s   Upgrade         InProgress   Restoring Application Data
upgrade   8m30s   Upgrade         Completed    Upgrade completed

[jgato@provisioner ~]$ date
Thu Feb  6 05:19:47 EST 2025
```

Everything done in about 15 minutes. Considering this is baremetal, only the reboot consumed about 5 of these minutes.

Lets check the restore of our workload:

```bash
$ oc -n example-workload get pod
NAME                                        READY   STATUS    RESTARTS   AGE
exception-app-deployment-575c65d8cf-szjsf   1/1     Running   0          94s

```

There are other features not tested in the blog, like rollback if fail. But I did not want to do it too complex and give only a first approach.

## Upgrading a cluster with traditional upgrade

*Note: This is just a comparative on the amount of time, as reference. But it is not intended to compare (or to conclude) which one is better. As explained in the introduction IBU only covers very specific scenario and only SNO clusters. "Traditional" upgrade has to cover absolutely all the possible scenarios.*

We take a similar SNO and (simplified installation steps for clarity of the blog):

To intermediate version 4.15.38

```bash
$ date
Thu Feb  6 05:34:15 EST 2025

$ oc adm upgrade --to=4.15.38

$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.15.38   True        False         50s     Cluster version is 4.15.38
$ date
Thu Feb  6 06:33:14 EST 2025

```

Then to 4.16.23 (there is no update path to .9, but it is oka). I also need some time to update some OLM Operators:

```bash
$ date
Thu Feb  6 06:46:10 EST 2025

$ oc adm upgrade --to=4.16.23

$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.15.38   True        True          37s     Working towards 4.16.23: 110 of 903 done (12% complete), waiting on etcd, kube-apiserver


```

So, it took like minutes to do the two upgrades up to reach to 4.16.

# Conclusion

Image Base Upgrade of Single Node Openshift cluster is a new an efficient way of upgrading clusters when there are very strict maintenance windows. But at the same time, it is limited to very concrete scenarios, where your infrastructure is composed by homogenous SNOs. If it is the case, ugprades (with backup/restore of workloads) can take just 15/20 minutes. Which is a very meaningful improvement, compared to mechanisms that has to cover all the possible scenarios. 