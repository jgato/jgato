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

![](assets/contribute_cluster_operator_20240430163234953.png)

you will get reviewers and valid reference to the bug:

![](assets/contribute_cluster_operator_20240430163357915.png)

## Get the PR accepted on downstream

.......

## Make backport to other releases

