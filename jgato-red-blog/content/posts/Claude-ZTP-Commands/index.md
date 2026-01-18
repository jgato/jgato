+++
title = "Building Custom Claude Code Skills for Zero Touch Provisioning Operations"
date = "2026-01-18"
author = "Jose Gato Luis"
authorTwitter = "jgatoluis" #do not include @
cover = ""
tags = ["openshift", "ztp", "automation", "claude", "ai", "rhacm", "argocd"]
keywords = ["", ""]
description = "A new project that includes custom Claude Code commands and skills to streamline daily Zero Touch Provisioning operations. This automation enables cluster lifecycle management (create/remove/redeploy) for OpenShift clusters through seamless integration with Git, ArgoCD, and OpenShift RHACM."
showFullContent = false
readingTime = false
hideComments = false
+++


# Building Custom Claude Code Skills for Zero Touch Provisioning Operations

*** During the examples you will see a lot of oc commands. If you are not used to Openshift, you can just think that oc=kubectl *** 

  Managing multiple OpenShift clusters through Red Hat Advanced Cluster Management (RHACM) using Zero Touch Provisioning (ZTP) can repetitive. Specially repetitive when you have to deploy/destroy the same set of clusters very frequently just to test: new feature, a bug fix, new integration task, check deployment workflow, etc. Without going into details on how the ZTP workflow works, I would say a Kustomize file (sync with ArgoCD) that contains a set of manifests with information to deploy an Openshift cluster. The deployment includes ArgoCd sync, and some interactions with the Openshift cluster that acts as "cluster that deployes other clusters".
  During a week, I cannot remember how many times I do:
  * Edit kustomization file to add the yaml of a cluster to deploy.
  * Git commit and push with a clear message.
  * Refresh and sync the ArgoCD app and wait everything goes oka.
  * There are some secrets that you dont want to have on the Git repo, so, you run an script or run some `oc create secrets`.
  * Wait and check everything is oka.
  
And of course the other way around, redeploy the cluster means to do the same process the other way around. Starting to uncomment the kustomization file, push, sync, wait. And create it again.

Of course we have some scripts, and some automations about this. But in the way of improving my AI skills I decided to use Claude Code Skills and Commands to do it on a more "natural language" way, instead of so much shell scripts with many if-then, if-then, if-then, waits, etc. So, the project is intended to facilitate my life, but, by the moment specially focused on learning on how to interact with these models.

The project is [here](https://github.com/jgato/ztp-cluster-automation/tree/main) and I am very welcomed to receive feedback.

I've built a comprehensive automation framework that transforms how I handle day-to-day cluster operations, combining GitOps principles with AI-assisted tooling to streamline what used to be manual, error-prone workflows. The automation framework enables comprehensive OpenShift cluster lifecycle management including creation, removal, and redeployment through seamless integration with Git, ArgoCD, and Red Hat Advanced Cluster Management (RHACM).

I dont do in the details of every command. You have the Git repo documented and ... I am changing many things on this moment. So, whatever I describe on the workflow would be changed. But I can go with some of the usual workflows

## Deploying a cluster workflow

Even if my favourite command, that started my idea to build the project, is "redeploy a cluster", for demoing it is better to just "deploy a cluster".

The usual workflow should start configuring your environment. That is pretty straight forward about setting your KUBECONFIG and the endpoint to interact with ArgoCD:

(The execution of every command is pretty long. And in the first phase it requires some user intervention. So, these are prettified simplified outputs for the blog):

 ```console
 ❯ /configure_environment 

● I'll help you configure the environment for ZTP operations.

  Let me start by checking the current KUBECONFIG status.

● The KUBECONFIG environment variable is not currently set. I need you to provide the path to your kubeconfig file.

● User declined to answer questions

❯ KUBECONFIG=~/Servers/EL8000-2/kubeconfig-el8k-hub-2 

  ⎿  🔍 Checking connectivity to OpenShift cluster...
     ✅ Connected to OpenShift cluster

● Excellent! The cluster is reachable with the KUBECONFIG: /home/jgato/Servers/EL8000-2/kubeconfig-el8k-hub-2

  Now let me check the available hubs to help you select one.

● Perfect! I found the available hubs. Based on your KUBECONFIG path (which contains "hub-2"), I'll present the hub options:
                                                                                      
● User answered Claude's questions:
  ⎿  · Which RHACM hub would you like to use for ZTP operations? → hub-2 (Recommended)

● Perfect! Environment configuration is complete.

  Configuration Summary:
  - KUBECONFIG: /home/jgato/Servers/EL8000-2/kubeconfig-el8k-hub-2
  - Selected Hub: hub-2 (Default hub)
	  - ArgoCD Endpoint: openshift-gitops-server-openshift-gitops.example.com
  - Cluster Status: ✅ Connected and reachable

✻ Worked for 33s

  ```

Now we are ready to deploy with `/deploy_cluster vso6`:

During the process these will be the tasks to do:
```console
  Todos · ctrl+t to hide todos
  ☐ Verify cluster manifest exists and check kustomization.yaml
  ☐ Prepare cluster pre-requirements (namespace and secrets)
  ☐ Add/uncomment cluster entry in kustomization.yaml
  ☐ Create git commit for cluster addition
  ☐ Push changes to origin/main
  ☐ Sync ArgoCD clusters application on hub-2
  ☐ Monitor cluster installation until ManagedCluster is available and joined (max 3 hours)
  ☐ Extract kubeadmin password
  ☐ Extract and save kubeconfig file

```

Here we go:
```
❯ /deploy_clusters vsno6 

● Let me read the cluster manifest to show you a summary.

● Read(vsno6.yaml)
  ⎿  Read 84 lines

● Cluster Summary for vsno6:
  - Type: Single Node OpenShift (SNO)
  - Namespace: vsno6
  - OpenShift Version: 4.18.5
...
...
  - Status in kustomization.yaml: Currently commented out (line 12: #- vsno6.yaml)

● Now I'll prepare the cluster pre-requirements (namespace and secrets).

● Skill(/prepare_clusters)
  ⎿  Successfully loaded skill

● I'll prepare the cluster pre-requirements for vsno6. Since we're in the context of cluster deployment, I'll proceed with creating the namespace and secrets.

● The namespace vsno6 already exists. Now I'll run the preparation script to create the required secrets.

● Bash(export KUBECONFIG=/home/jgato/Servers/EL8000-2/kubeconfig-el8k-hub-2 && .claude/commands/scripts/prepare_ztp_cluster_pre_reqs.sh vsno6)
  ⎿  vsno6                                         
     secret/assisted-deployment-pull-secret created
     Now introduce the credentials for the BMC
     … +4 lines (ctrl+o to expand)

● The secrets were created successfully. Let me verify both secrets exist in the vsno6 namespace.

● Bash(oc --kubeconfig /home/jgato/Servers/EL8000-2/kubeconfig-el8k-hub-2 get secret -n vsno6 assisted-deployment-pull-secret vsno6-bmc-secret)
  ⎿  NAME                              TYPE     DATA   AGE
     assisted-deployment-pull-secret   Opaque   1      15s
     vsno6-bmc-secret                  Opaque   2      6s

● Cluster preparation completed successfully!

  The namespace vsno6 now has the required secrets:
  - ✅ assisted-deployment-pull-secret (created 15s ago)
  - ✅ vsno6-bmc-secret (created 6s ago)

● Now I'll uncomment the vsno6.yaml entry in kustomization.yaml.

● Changes applied to kustomization.yaml:
  - vsno5.yaml
  -#- vsno6.yaml
  +- vsno6.yaml
  #- vsno7.yaml

● Now I'll create a git commit and push the changes.

● Commit created successfully. Now I'll push to origin/main.

● Push completed successfully. Now I'll sync the ArgoCD clusters application on hub-2.

● Skill(/synch_clusters)
  ⎿  Successfully loaded skill

● I'll sync the ArgoCD clusters application on hub-2.

  Sync Details:
  - Hub: hub-2 (openshift-gitops-server-openshift-gitops.apps.hub-2.example.com)
  - Application: clusters
  - Prune: No (standard sync without prune)

  Let me check if we're already logged in to the ArgoCD instance.

● We're already logged in to the ArgoCD instance. Now I'll refresh and sync the clusters application.

● Let me sync the application directly, which will also refresh it.

● Bash(argocd app sync clusters --server openshift-gitops-server-openshift-gitops.apps.hub-2.example.com --insecure --grpc-web) 

● ArgoCD Sync Completed Successfully!

  Key results:
  - ✅ ClusterInstance vsno6 created in namespace vsno6
  - ✅ Sync Status: Synced to main (ea48355)
  - ✅ Phase: Succeeded
  - ✅ Duration: 2s

● Now I'll begin monitoring the cluster installation. I'll check the status every 5 minutes until the ManagedCluster is Available and Joined, with a maximum wait of 3 hours.

  Starting installation monitoring at 12:32...

● Skill(visualize-cluster-status)
  ⎿  Successfully loaded skill

● I'll check the status of cluster vsno6 using the optimized data collection script.

```

From now on, it will use the visualization command every 5 minutes until the cluster is fully created. I only show you the visualization final status:

```

```


## Conclusion and learnings

I think the most important part here are, as usual, the learnings. All these examples interacting with LLMs are always shown as the final successful result. But we have to be honest, and humble, and these were my main struggles:

### prompting the user at the beginning 

I want to make a whole redeployment, with errors control and re-tries. But I dont want the model to ask me to configure the secrets 1 hour and half after invoking the command. I want to trigger it and leave it running, with the confidence that is finished when I am back. 
The context configuration was pretty tricky about doing everything at starting phase, and instruct the model to keep it using all over the process. Looks easy, but not deterministic nature of the model made it complex.

### Model dont take decisions 

This was something important for me. The model dont take decisions. It has a list of to-do. Some waits, and waits, and be patient. These ZTP operations are usually super repeatable. In a controlled environment, when you have deployed clusters hundreds of times, it will not fail. So, it is expected the process will not fail. Keep to model waiting but not taking decisions, or it would brake the process. 
Happened, specially during redeployments, that it takes too long for the process. The model tries to debug, and to find out a possible problem, and even worst: try to fix it. It is not the intention of this workflow. Here we dont debug, we dont try to find problems, and of course, we dont want the model try to fix things. Maybe instruct the model, that if is takes too long to break. I will see what happened when I am back.

### Models try to do more work than expected

One of the commands that took me more time is the cluster visualization status. It was crazy. Even if I had an structure of what I wanted to visualize, some times the model tries to do their own inspection and show different things. This breaks a kind of "application experience" that you always have the same result and the same visualization of the status. To constrain the model to only visualize what I wanted to show was painful. 

This one is also very related to "dont take decisions". After visualize the status, it tried many times to find out more about what could be failing. Or trying to give more details than need it. "Such a smarty-pants model". Not always more is better..

### No deterministic commands

During the build I have tried to use as much "natural language" as possible. If I have to script too much... I would just do the model to do a super complete, long, and hard to maintain shell script. This was not my objective. But for some steps, I had to use an script for constrain at maximum how to execute some actions. 

Otherwise, the model always try to solve the command on a different way. For example, to a get a secret it would be done with:
 * `oc --kubeconfig get secret <params>` or,
 * `oc get secret --kubeconfig <params> && oc get secret <params>`
 * or whatever other combination like combining `echo` with a command. Other times, using `cat` and pipes, etc. I guess you understand the point.
 
It is not only that makes you lost the control, in general is pretty smart. My point here is security. Claude Code, and other tools, use different mechanisms to allow some approved commands without user intervention. For hours, I tried to have my allow list very controlled. So, I would have things like:

```json
  "permissions": {
    "allow": [
      "Skill(redeploy_clusters)",
...
...
      "Bash(.claude/commands/scripts/*)",
      "Bash(.claude/skills/visualize-cluster-status/scripts/*)",
...
...
      "Bash(mkdir -p tmp-*)",
      "Bash(oc --kubeconfig *)",
...
    ],

```

I wanted to allow whatever is done with your kubeconfig. So, it is up to you have much operations are allowed by your kubeconfig. This config looks good, but after an hour of deploying, maybe the model tries to do:

```
 oc get secret -n vsno6
```

Instead of:

```
oc --kubeconfig=.... get secret -n vsno6
```

or:
```
KUBECONFIG=.... oc get secret -n vsno6
```


This does not match your allow list configuration, and it will ask user for perms. As explained before, I dont want to need user intervention for an automated task. But at the same time, you dont want to have your allow list full of different combinations of commands, or worst, to have to have allow every oc command.

And I would show you so many different combinations, and not only about oc commands. Specially tricky when trying to work with some shell commands. 

To have all this under control was hard and painful as hell. We know models are not deterministic, but here it is really painful. And I did not want to use predefined scripts for everything. 

### Final thoughts

As learning it has been super interesting work. And the results are awesome. But it happened me the same awesome AI effect. The first time I had it running was like magic. Then, when you try to use it more actively and not playing with a demo, the problems start to raise. In general, the not deterministic results of the commands and having security under control. 

I still have a lot of work to do. Lets see how much time this save me, and how much time I have to spend refining the different prompts. Or maybe changing prompts for scripts ;) 