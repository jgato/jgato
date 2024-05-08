*ongoing work*

# how to contribute to an Openshift Cluster Operator

Or at least how it was my experience.

## Do an small change in the code and run it

I have this other document about how to code, play and do your first "hello world" into an existing operator: https://github.com/jgato/jgato/blob/main/random_docs/coding_cluster_operator.md

## Fix a bug

Create a bug ticket on our jira.

Do the fix in the code, build and test as explained above.

It is good to start from the main branch on upstream.

## Upstream first

With the fix ready, clone the Upstream version of the operator.

Rebase your branch with your fix. Build and test.

Do a PR of your branch to main branch. 

You have to sign your commit with a DCO: https://github.com/apps/dco

wait to get accepted and merged

## Cherry pick to downstream

Eventually someone will pull upstream into downstream containing your fix. But, you can do this by yourself. Specially if you want to have this into an incoming release.

Make a new brach, this time based on the main branch of downstream.

git checkout -b  PreprovisioningImage-poweroff-fix

Cherry pick the commit/commits with your fix into the branch (the commits you did in the previous PR). You have to cherry pick the commits with the fix, not the merge commit.

git cherry-pick d2d700e73

Git log shows that now you have added this last commit

```
commit 9f7b61ec799f434e4466645e521e6e4802c05d62 (HEAD -> PreprovisioningImage-poweroff-fix, origin/PreprovisioningImage-poweroff-fix)
Author: Jose Gato <jgato@redhat.com>
Date:   Mon Apr 29 13:14:59 2024 +0200

     PreprovisioningImage should not be created on poweroff

```

Now you can make a PR with the new branch into the downstream/main. That should contain downstrea/main plus your commits with the fix.

Tittle this PR with [openshift-bug-id] of our jira system.

![](assets/contribute_cluster_operator_20240430162333746.png)

Set the Jira ticket target to the release-4.y version

It is very important to have the PR correctly linked with the bug. And this properly set. If not you will receive a not-valid-bug. To have this correctly set provides automatic integration in the next steps. 

![](assets/contribute_cluster_operator_20240430163234953.png)

you will get reviewers and valid reference to the bug:

![](assets/contribute_cluster_operator_20240430163357915.png)

notice how your bug has been automatically moved to a new POST stage:

![](assets/contribute_cluster_operator_20240430163655065.png)

## Get the PR accepted on downstream

When you have the ok-to-test, a battery of test will be executed. 


When passed, and you have the ok from reviewers(/lgtm), the PR is accepted and merged.
![](assets/contribute_cluster_operator_20240502100941427.png)

![](assets/contribute_cluster_operator_20240502102836602.png)

The bug is also automatically moved to ON_QA, demonstrating the integration with Github.

![](assets/contribute_cluster_operator_20240502101228031.png)

![](assets/contribute_cluster_operator_20240502102252157.png)

When merged, the PR is included into night builds that will be tested and accepted by the QA team:

You also receive a notification about when it will be available. In Github you see:

![](assets/contribute_cluster_operator_20240502101136469.png)

And in Jira you see:

![](assets/contribute_cluster_operator_20240502102608461.png)

The QA team will now, not just test your changes, but also the whole integrated release. If the but no long happens, it will be `verified`.

### Testing your nightly build

You can check the nightly build created on Openshift release stream, with the tool:

https://amd64.ocp.releases.ci.openshift.org/

After some search you will find your build:

![](assets/contribute_cluster_operator_20240506083922016.png)

https://amd64.ocp.releases.ci.openshift.org/releasestream/4.16.0-0.nightly/release/4.16.0-0.nightly-2024-05-01-111315

There you can see all the news for that nightly build, including our fix:

![](assets/contribute_cluster_operator_20240503163750601.png)

#### Access the nightly build image on `registry.ci`

This registry requires special perms that are not included in your usual `pull-secret`.  So, first obtain an API token by visiting https://oauth-openshift.apps.ci.l2s4.p1.openshiftapps.com/oauth/token/request and login
 

```bash
> oc login --token=sha256~<TOKEN> --server=https://api.ci.l2s4.p1.openshiftapps.com:6443
```

Then, you can save the credentials to your existing pull-secret:

```
> oc registry login --registry-config ~/.config/containers/auth.json 
> > cat ~/.config/containers/auth.json | grep registry.ci -A 2
		"registry.ci.openshift.org": {
			"auth": "<TOKEN>"
		},


```

Now, you can access to the image:

```
>  oc adm release info  registry.ci.openshift.org/ocp/release:4.16.0-0.nightly-2024-05-01-111315
Name:           4.16.0-0.nightly-2024-05-01-111315
Digest:         sha256:eeb02c62d7ec73433e31de1aca03270fdc0ccaadefc46163e9983153f5270462
Created:        2024-05-01T11:14:56Z
OS/Arch:        linux/amd64
Manifests:      733
Metadata files: 1

Pull From: registry.ci.openshift.org/ocp/release@sha256:eeb02c62d7ec73433e31de1aca03270fdc0ccaadefc46163e9983153f5270462

Release Metadata:
  Version:  4.16.0-0.nightly-2024-05-01-111315
  Upgrades: <none>
  Metadata:

Component Versions:
  kubectl          1.29.1                
  kubernetes       1.29.4                
  kubernetes-tests 1.29.0                
  machine-os       416.94.202405010033-0 Red Hat Enterprise Linux CoreOS

Images:
  NAME                                           DIGEST
  agent-installer-api-server                     sha256:f59fb87a9f6583bb772f48cc171eb7d915c11b8d69862a1dc553d95a96321214
  agent-installer-csr-approver                   sha256:31c6c4b2640ccf2d9906f9a852cea649b6d71cdc64e32713721136b5d00413bd
  agent-installer-node-agent                     sha256:e7d77d7615da44e2e302e4ea0b9f502076d4902cb794ba32f803238bd91f03a6
  agent-installer-orchestrator                   sha256:65c3cffa769f664bc3f181fb9cf48ef29255da52fe62a42e5b7bba4540f8a758

```


> Eventually, the registry.ci token expires, so you will have to update it with a new login token


#### Try the image with the Agent Based installer

From the previous [tool](https://amd64.ocp.releases.ci.openshift.org/releasestream/4.16.0-0.nightly/release/4.16.0-0.nightly-2024-05-01-111315) you have an option to download the installer:

![](assets/contribute_cluster_operator_20240506084221789.png)

Or use `adm` command to generate the installer.

```bash
> oc adm release extract --tools registry.ci.openshift.org/ocp/release:4.16.0-0.nightly-2024-05-01-111315 --registry-config ~/.config/containers/auth.json
> ls -l
total 480920
-rwxr-xr-x. 1 jgato jgato  35909338 abr 30 02:07 ccoctl-linux-4.16.0-0.nightly-2024-05-01-111315.tar.gz
-rwxr-xr-x. 1 jgato jgato  66770163 may  1 01:14 openshift-client-linux-4.16.0-0.nightly-2024-05-01-111315.tar.gz
-rwxr-xr-x. 1 jgato jgato 389739326 may  1 11:57 openshift-install-linux-4.16.0-0.nightly-2024-05-01-111315.tar.gz
-rw-r--r--. 1 jgato jgato     32562 may  6 09:22 release.txt
-rw-r--r--. 1 jgato jgato       462 may  6 09:22 sha256sum.txt

```

#### Try out using ZTP and Assisted Installer

You can create a `clusterImagetSet`pointing to the nightly build, to be used from a Siteconfig:

> oc get clusterimagesets.hive.openshift.io img4.16.0-0.nightly-2024-05-01-111315 -o yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  creationTimestamp: "2024-05-03T15:45:12Z"
  generation: 1
  labels:
    channel: fast
    visible: "true"
  name: img4.16.0-0.nightly-2024-05-01-111315
  resourceVersion: "11197666"
  uid: 910964ab-b2c7-4b6c-a2f9-91da037f2898
spec:
  releaseImage: registry.ci.openshift.org/ocp/release:4.16.0-0.nightly-2024-05-01-111315

```yaml
---
apiVersion: ran.openshift.io/v1
kind: SiteConfig
metadata:
  name: "multinode-1"
  namespace: "multinode-1"
spec:
  baseDomain: "spoke-mno.el8k.se-lab.eng.rdu2.dc.redhat.com"
  pullSecretRef:
    name: "assisted-deployment-pull-secret"
  clusterImageSetNameRef: "img4.16.0-0.nightly-2024-05-01-111315"
<REDACTED>
```

> Notice the image is download from `registry.ci.openshift.org` which requires special perms and pull-secret.



## Make backport to other releases

