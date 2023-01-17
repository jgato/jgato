# Git tips and tricks

## Follow a PR to get the branches and tags containing it

This depends on how it is the Git flow. In this example, we start from a PR from a forked project into upstream main. As an example we use: [MGMT-12794: allow to edit ProvisionRequirement post install by slaviered · Pull Request #4843 · openshift/assisted-service · GitHub](https://github.com/openshift/assisted-service/pull/4843)

From this PR is easy to see that the PR was done on Master. Se can look the commit's IDs for that PR using the title:

```bash
> git log origin/master | grep "MGMT-12794: allow to edit ProvisionRequirement post install " -B 4
commit 8dace6302f29790c4f676264e047e2848d21a271
Author: slavie <slavie@redhat.com>
Date:   Sun Jan 1 17:29:43 2023 +0200

    MGMT-12794: allow to edit ProvisionRequirement post install (#4843)
```

Now we have the commid ID we can look to branches including it:

```bash
> git branch -a --contains 8dace6302f29790c4f676264e047e2848d21a271
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dependabot/docker/openshift/release-golang-1.19
  remotes/origin/dependabot/go_modules/github.com/thoas/go-funk-0.9.3
  remotes/origin/dependabot/go_modules/k8s.io/api-0.27.0-alpha.0
  remotes/origin/dependabot/go_modules/k8s.io/client-go-0.27.0-alpha.0
  remotes/origin/dependabot/go_modules/k8s.io/kube-aggregator-0.26.0
  remotes/origin/master
  remotes/origin/release-4.13
  remotes/origin/release-4.14
  remotes/origin/release-ocm-2.8
```

It is not available in the release-2.5?  When doing a cherry pick command, using the bot, the commit id changes. Because the author is now, also, the bot.

So, we will have to dig into the logs of other branches:

```bash
> git log origin/release-ocm-2.5  | grep "MGMT-12794" -B 4
commit 7b7faf4c7e310b5c0a31e391901223d892c89062
Author: slavie <slavie@redhat.com>
Date:   Wed Jan 4 15:48:58 2023 +0200

    MGMT-12794: allow to edit ProvisionRequirement post install (#4860)

> git log origin/release-ocm-2.6  | grep "MGMT-12794" -B 4
commit 6c9f6fc232d2e77aae6fc9a6e0ee269a75b019dc
Author: OpenShift Cherrypick Robot <openshift-cherrypick-robot@redhat.com>
Date:   Wed Jan 4 10:37:17 2023 +0000

    [release-ocm-2.6] MGMT-12794: allow to edit ProvisionRequirement post install (#4854)

    * MGMT-12794: allow to edit ProvisionRequirement post install

> git log origin/release-ocm-2.7  | grep "MGMT-12794" -B 4
commit 5c3586f243e8512081cf1078942c9737ffd06035
Author: OpenShift Cherrypick Robot <openshift-cherrypick-robot@redhat.com>
Date:   Tue Jan 3 13:30:57 2023 +0000

    [release-ocm-2.7] MGMT-12794: allow to edit ProvisionRequirement post install (#4853)

    * MGMT-12794: allow to edit ProvisionRequirement post install
--
commit edc56e7beb49748fc18089039789fae8ece59c03
Author: slavie <slavie@redhat.com>
Date:   Tue Dec 27 13:56:23 2022 +0200

    Revert "MGMT-12794: allow to edit ACI post install (#4836)" (#4840)
--
commit fd129c6b3e8751bb884fb3ca5b67f6aa50959c1d
Author: OpenShift Cherrypick Robot <openshift-cherrypick-robot@redhat.com>
Date:   Mon Dec 26 21:58:30 2022 +0000

    MGMT-12794: allow to edit ACI post install (#4836)


```

The original commit was the '8dace6302f29790c4f676264e047e2848d21a271' done by slavie. The cherry picks for 2.6, 2.7 and 2.8 where done invoking the bot, that now appears as author.  And these why they have different commit ids. 

In the 2.5 the cherry pick was done by slavie, not by the bot. But it also has a different commit ID. But something else would have changed to make the commit different.

```bash
> git diff 8dace6302f29790c4f676264e047e2848d21a271 7b7faf4c7e310b5c0a31e391901223d892c89062
```

shows many differences. 



# Extra tricks



## Container images related



### Get the Tag from the hash



```bash
> skopeo inspect --format "{{.Labels.version}}" "docker://registry.redhat.io/multicluster-engine/agent-service-rhel8@sha256:3cf067da681b2c523d207c4847a23da61b16ed96f7f815d5555381d23fc9028d"

```


