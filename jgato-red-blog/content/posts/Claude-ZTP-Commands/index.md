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

**Note:** If you're not familiar with OpenShift, just think `oc` = `kubectl` in the examples below.

**Note:** This is not an official project, and for sure, nothing I would use in production environments. By the moment, just a personal project to learn about using "Claude Code" for these tasks.

Managing multiple OpenShift clusters through RHACM using Zero Touch Provisioning (ZTP) is highly repetitive, especially when deploying and destroying the same clusters frequently for testing new features, bug fixes, or workflow validation. In a nutshell, ZTP uses a Kustomize file (synced with ArgoCD) containing manifests to deploy OpenShift clusters through a hub cluster.

During a typical week, I perform these steps countless times:
* Edit the kustomization file to add a cluster YAML
* Git commit and push with a descriptive message
* Refresh and sync the ArgoCD app, then wait for completion
* Create secrets that shouldn't live in Git repos
* Monitor until everything is ready

Redeploying means reversing the process: comment out the kustomization entry, push, sync, wait, then repeat the deployment steps.

While I had some scripts for this, I wanted to improve my AI skills by using Claude Code Skills and Commands instead of complex shell scripts full of conditionals and wait loops. The project aims to make my life easier while focusing on learning how to interact with these models effectively.

The project is available [here](https://github.com/jgato/ztp-cluster-automation/tree/main), and I welcome feedback.

I won't dive into every command detail since the Git repo has documentation and things are actively changing. Instead, I'll walk through some typical workflows.

## Deploying a Cluster

While my favorite command is "redeploy a cluster" (the one that inspired this project), I'll demonstrate the simpler "deploy a cluster" workflow.

First, configure your environment by setting your KUBECONFIG and ArgoCD endpoint:

**Note:** Command outputs are simplified for readability. The actual execution is longer and requires some initial user interaction.

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

During the process these will be the tasks to do.
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

The only user intervention will happen during "prepare cluster pre-requierements". To introduce some credentials. Here we go:
```console
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

The visualization command runs every 5 minutes until the cluster is fully deployed. 
```console
● # 🎯 vsno6 | Status: ✅ INSTALLED | Duration: ~41m (hub-2)

  ## ClusterInstance Status (Primary CR)
  **Created:** 11:31:58Z | **Generation:** 1 (observed: 1) | **Phase:** Deployed
  - ✅ ClusterInstanceValidated - Validation succeeded
  - ✅ RenderedTemplates - Rendered templates successfully (7 manifests)
  - ✅ RenderedTemplatesValidated - Rendered templates validation succeeded
  - ✅ RenderedTemplatesApplied - Applied site config manifests
  - ✅ Provisioned - Provisioning completed

  ## Core Resources
  +----------------------+--------+--------------------+----------------------+
  | Resource             | Status | State/Info         | Details              |
  +----------------------+--------+--------------------+----------------------+
  | 📦 BareMetalHost     | ✅     | provisioned/det.   | Power: Off           |
  | 💿 InfraEnv          | ✅     | Image ready        | Created: 11:31:59Z   |
  | 🚀 AgentClusterInst  | ✅     | adding-hosts       | Cluster is installed |
  | 🎮 ManagedCluster    | ✅     | Available & Joined | Ready                |
  +----------------------+--------+--------------------+----------------------+

  ## Agent Details (1 total, 1 approved)
  +---------+--------+-----------+
  | ID      | Role   | State     |
  +---------+--------+-----------+
  | ...0006 | master | installed |
  +---------+--------+-----------+

  **Conditions:** ✅ Validated ✅ Requirements ✅ Completed ✅ Not Failed

  Installation Complete! 🎉

  The cluster vsno6 has successfully completed installation in approximately 41 minutes:
  - ✅ ManagedCluster is Available and Joined
  - ✅ Agent installed and operational
  - ✅ All provisioning steps completed

```

Finally, it extracts credentials and shows the final results:
```console
● Deployment Complete! 🎉

  The cluster vsno6 has been successfully deployed on hub-2.

  Deployment Summary

  Cluster Information:
  - Name: vsno6
  - Type: Single Node OpenShift (SNO)
  - OpenShift Version: 4.18.5
  - Installation Time: ~41 minutes
  - Status: ✅ Available and Joined

  Cluster Access:
  - Kubeadmin Password: ******************
  - Kubeconfig Location: /tmp/kubeconfig-vsno6

  Deployment Steps Completed:
  1. ✅ Verified cluster manifest (vsno6.yaml)
  2. ✅ Created namespace and secrets (pull-secret, BMC credentials)
  3. ✅ Updated kustomization.yaml (uncommmented vsno6.yaml entry)
  4. ✅ Committed and pushed changes to git (commit: ea48355)
  5. ✅ Synced ArgoCD clusters application on hub-2
  6. ✅ Monitored installation to completion
  7. ✅ Extracted kubeadmin password
  8. ✅ Extracted kubeconfig file

  Final Cluster Status:
  - ManagedCluster: Available & Joined
  - Agent: Installed (master role)
  - BareMetalHost: Provisioned and detached
  - All conditions: Validated ✅

```

The cluster is now ready for use.

The redeploy command is much more impressive, but it requires time and gathering lots of logs. Not very readable for the blog, but I invite you to try it and give me some feedback.

## Key Learnings

The most valuable part of this project has been the lessons learned. LLM examples always show the final successful result, but let me be honest about the main challenges I faced:

### Upfront User Prompting

I needed the model to handle the entire redeployment with error control and retries, but without asking for secrets 90 minutes into execution. I wanted to trigger it and walk away confidently.

Getting the model to gather all necessary context upfront and maintain it throughout the process proved challenging. The non-deterministic nature of LLMs made this harder than it sounds.

### Preventing Unwanted Decision-Making

ZTP operations are highly repeatable. In a controlled environment where I've deployed clusters hundreds of times, failures are rare and expected to succeed. I needed the model to follow the task list patiently without making decisions that could break the process.

During long redeployments, the model would sometimes try to debug, investigate problems, or even attempt fixes. This wasn't the goal. I don't want debugging or problem-solving during these workflows. If something takes too long, just break and let me investigate when I return.

### Over-Engineering by the Model

The cluster status visualization command was particularly challenging. Even with a clear structure, the model would sometimes improvise its own inspection and show different information. This broke the consistent "application experience" I wanted.

Constraining the model to show only what I specified was painful. After displaying status, it would often investigate potential failures or provide unnecessary details. Sometimes, less is more.

### Non-Deterministic Command Execution

I wanted to use natural language as much as possible. If I had to script everything, I might as well write a complete shell script—that wasn't my goal. However, for some steps, I needed scripts to constrain exactly how actions execute.

Otherwise, the model would execute the same command in different ways. For example, getting a secret could be:
* `oc --kubeconfig get secret <params>`
* `oc get secret --kubeconfig <params> && oc get secret <params>`
* Various combinations with `echo`, `cat`, pipes, etc.

While generally smart, this variability becomes a security concern. Claude Code uses permission mechanisms to allow approved commands without user intervention. I spent hours controlling my allow list:

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

I wanted to allow operations based on your kubeconfig permissions. This config looked good, but after an hour of deployment, the model might use:

```bash
oc get secret -n vsno6
```

Instead of the allowed patterns:
```bash
oc --kubeconfig=.... get secret -n vsno6
# or
KUBECONFIG=.... oc get secret -n vsno6
```

This doesn't match the allow list, triggering permission prompts. I don't want user intervention during automation, but I also don't want an allow list covering every command variation.

The non-deterministic nature of models made controlling this extremely challenging. I didn't want to rely entirely on predefined scripts, but finding the right balance was difficult. 

### The Addictive Nature of AI-Assisted Development

Adding new features becomes surprisingly addictive. The initial implementation is easy and straightforward—getting a working first version takes minutes. Refining it to reliable quality is harder, but you can leverage the model itself to add new steps or make the workflow more robust.

This creates an "addiction loop": "Could I also make it do...?" Why not? A couple of minutes later, you see results. The rapid feedback cycle is compelling.

Example: "Can I make ArgoCD login use KUBECONFIG instead of opening an interactive web browser?" (Currently working on this!) 

### Final Thoughts

This has been fascinating work with impressive results. However, I experienced the classic "AI effect": the first successful run felt like magic, but moving from demo to active use revealed challenges—mainly non-deterministic command execution and security control.

I still have work to do. Time will tell whether this saves me more time than I spend refining prompts... or whether some prompts ultimately become scripts again. :) 