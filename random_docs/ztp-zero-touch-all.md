# Going further about Zero Touch with ZTP Gitops tools

[Zero Touch Provisioning Gitops way](https://docs.openshift.com/container-platform/4.11/scalability_and_performance/ztp_far_edge/ztp-deploying-far-edge-clusters-at-scale.html) mainly exposes the creation of clusters with **Siteconfigs**, and the configuration/upgrade with **PolicyGenTemplates**.

In this document, we will go further about how to configure your cluster, during deployment/installation. And also, how to configure/upgrade your cluster in a more advanced way, than just using the usual PolicyGenTemplates. 

## Siteconfigs: allowed configurations during deployments

Siteconfig is the template for making your cluster's deployment. It is related to day-0 and it includes some pre-configuration or RAN profile. This profile includes some extra configurations related to telco RAN infrastructures. 

Apart from these included configurations, the profile can be extended with the usage of extra-manifests: [mainly Machineconfigs](https://github.com/openshift/machine-config-operator), but other Openshift/Kubernetes Resources would be included.

It is important to remark these manifests are applied during installation. The advantage of applying here is, that you have your cluster created and configured at the same time. Not having to apply policies on day-2 operations. 

So, Siteconfigs are mainly intended to create/deploy your clusters. But,  there is room for some extra configurations. These configurations are applied once, later, these are out of the control of the GitOps flow. 

### Using extra-manifests

To extend any Siteconfig with extra configuration, you can create any directoy with yamls including Openshift/Kubernetes Resources to be added to the cluster during installation. Then, you can point to this directory from the 'Siteconfig.spec.extra-manifests' attribute.

#### Example: extra-manifest to wipe disks during installation

So first, we create a YAML with the resource you want to include during your cluster installation. In principle MachineConfigs, but others resources would also work.

In this example, we create a MC that uses RHCOS Ignition to wipe some disks during the installation:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 98-format-disks-master
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      disks:
        - device: /dev/sdc
          wipeTable: true
```

Just add the path to the directory containing your extra-manifests:

```yaml
apiVersion: ran.openshift.io/v1
kind: SiteConfig
metadata:
  name: "intel-1-sno-1"
  namespace: "intel-1-sno-1"
spec:
  baseDomain: "hubcluster-1.lab.eng.cert.redhat.com"
  pullSecretRef:
    name: "assisted-deployment-pull-secret"
  clusterImageSetNameRef: "img4.10.30-x86-64-appsub"
  sshPublicKey: "ssh-rsa AAAAB3NzaC1y.....hny6gs= jgato@provisioner.el8k.hpecloud.org"
  clusters:
  - clusterName: "intel-1-sno-1"
    clusterLabels:
      common: "true"
      du-profile-4.10: ""
      sites : "intel-1-sno-1"
    networkType: "OVNKubernetes"
...
...
    extraManifestPath: "extra-manifests/"

....
....
```

In this case, '/dev/sdc' will be wiped-out during installation. But other MCs would make other different tasks.

### Disabling default configurations from a RAN profile

[ToDo] But this is very simple

## PolicyGenTemplates

PolicyGenTemplate (PGT) is a CRD exposed by ZTP GitOps. It allows you to use some [pre-existing templates](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs) that will generate ACM Policies related to Telco RAN usual activities. It can be seen as a set of pre-created helpers, that can make same upgrade/configuration of Telco RAN activities easier. For example: deploying RAN operators, configuring SRIOV interfaces, configuring PTP, etc

What happens when you need to make configurations out of the scope of these [pre-existing templates](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs)? 

* You can create and include your own templates. That can be referenced from the PGTs. [See Injecting and creating your own PolicyGenTemplates](# Injecting-and-creating-your-own-PolicyGenTemplates)

* Instead of using PGT, as helpers, you can directly add any ACM Policy to your GitOps repo. [See section Configuring with ACM Policies](# Configuring-with-ACM-Policies)

### Configuring/Upgrading with exiting PolicyGenTemplates

The list of templates that can be references from a PGT can be found [here](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs)

From there we will find different templates that can be easily used from a PGT for making new configurations

#### Example: Disabling CatalogesSources from an OperatorHub

There exists a [template for managing the OperatorHub](https://github.com/openshift-kni/cnf-features-deploy/blob/release-4.10/ztp/source-crs/OperatorHub.yaml). It is designed to disable the pre-configured CatalogesSources that came, by default, with Openshift:

```bash
NAMESPACE               NAME                  DISPLAY               TYPE   PUBLISHER   AGE
openshift-marketplace   certified-operators   Certified Operators   grpc   Red Hat     16d
openshift-marketplace   community-operators   Community Operators   grpc   Red Hat     16d
openshift-marketplace   redhat-marketplace    Red Hat Marketplace   grpc   Red Hat     16d
openshift-marketplace   redhat-operators      Red Hat Operators     grpc   Red Hat     16d
```

So it will disable all of them if you add this to your PGT:

```yaml
kind: PolicyGenTemplate                                                           
metadata:                                                                         
  name: "common-4-9"                                                              
  namespace: "ztp-common"                                                         
  annotations:                                                                    
    force: "force-again"                                                          
spec:                                                                             
  bindingRules:                                                                   
    common-4-9: "true"                                                            
  sourceFiles:       
    - fileName: OperatorHub.yaml                                               
      policyName: "registries-policy"               
```

This will disable all the 'Catalogesources'.

You can configure your PGT to be more specific. When you add the 'OperatorHub.yaml', this is including the disableall. But, you can override that, and be more specific. According to the specification of the CR OperatorHub, you could do something like:

```yaml
    - fileName: OperatorHub.yaml                                               
      policyName: "registries-policy"                                          
      spec:                                                                    
        disableAllDefaultSources: false                                        
        sources:                                                               
          - disabled: true                                                     
            name: redhat-marketplace                                           
          - disabled: true                                                     
            name: certified-operators                                          
          - disabled: true                                                     
            name: community-operators 
```

Only these 3 sources will be disabled. remaining the redhat-operators only.

```bash
$ oc get catalogsources -A
NAMESPACE               NAME               DISPLAY             TYPE   PUBLISHER   AGE
openshift-marketplace   redhat-operators   Red Hat Operators   grpc   Red Hat     17d
```

### Injecting and creating your own PolicyGenTemplates

In the previous section we have used PGTs with existing templates to facilitate configurations and upgrades. When you are using a PGT to create a Policy you will have some of these references to other existing yamls.

```yaml
 sourceFiles:
    - fileName: SriovSubscription.yaml
      policyName: "subscriptions-policy"
    - fileName: SriovSubscriptionNS.yaml
      policyName: "subscriptions-policy"
    - fileName: SriovSubscriptionOperGroup.yaml
      policyName: "subscriptions-policy"
```

These yamls files lives inside the ZTP installation you did into ArgoCD. You can check all these pre-created yamls [here](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs/). But mainly, inside these files, you will find just Openshift/Kubernetes pre-configured resources:

'SriovSubscription.yaml'

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: sriov-network-operator-subscription
  namespace: openshift-sriov-network-operator
  annotations:
    ran.openshift.io/ztp-deploy-wave: "2"
spec:
  channel: "stable"
  name: sriov-network-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
status:
  state: AtLatestKnown
```

Internally, ZTP tools will create the proper Policies (based on these resources) and will make the needed bindings according to your PGT rules:

```yaml
apiVersion: ran.openshift.io/v1
kind: PolicyGenTemplate
metadata:
  name: "common-rangen-4.9"
  namespace: "ztp-common"
spec:
  bindingRules:
    common: "true"
    du-profile: "v4.9"
```

If you need to manage the desired status of any Openshift/kubernetes Resource, that is not managed by any of these pre-existing files, you can inject your own ones templates.

How we inject our own files to be used by a PGT?

First you create the yaml with the Resource you want to manage:

```yaml
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: foo
status:
  phase: Active
```

Then we have to inject the yaml into the currently running Pod with ArgoCD.

Lets copy that file inside:

```bash
$>  oc -n openshift-gitops rsync source-crs/ openshift-gitops-repo-server-684d5bd56-t4lts:/.config/kustomize/plugin/ran.openshift.io/v1/policygentemplate/source-crs/ -c argocd-repo-server
WARNING: cannot use rsync: rsync not available in container
CreateProject.yaml
```

Now it can be referenced from a PGT:

```yaml
 sourceFiles:
    - fileName: CreateProject.yaml
      policyName: "projects-policy"
```

Finally, you can use it to create projects any Project.  You have to reference the template, and override the needed attributes. In this case, the Project name.

```yaml
 sourceFiles:
    - fileName: CreateProject.yaml
      policyName: "projects-policy"
      metadata:
        name: bar
```

[ToDo] When injecting the templates into the Pods, these will be lost after Pod restarts. But, you can create your own ztp-site-generate Container Image including them.

### Hub cluster templates

[Hub cluster template](https://open-cluster-management.io/concepts/policy/#policy-templates) is an advanced feature of ACM to customize your Policies. But, this can be also used at PGT level.

[ToDo] Include documentation from @aidan

## Configuring with ACM Policies

When there are no PGT templates, raw ACM Policies can be created. Actually, the PGT Templates are transformed into ACM Policies by one of the ZTP GitOps tools.

When you need to make more generic configurations, like RBAC, Users, Projects, etc, you can still combine your GitOps pipelines but directly using ACM Policies. ACM Policies would be a little bit more complex, than just using a PGT, but are much more flexible. Actually, these Policies can act on the desired status of any Openshift/Kubernetes Resource.

An ACM Policy is composed by a Policy with the desired status, and a set of roles and bindings to select which cluster will be affected. More in concrete, these are the needed objects:

* [The Policy](https://open-cluster-management.io/concepts/policy/). The main object, it contains a set of 'must have' and must not have' Kuberentes/Openshift objects.

* PlacementRule. This object select different clusters according to different labels.

* PlacementBinding. This object selects a set of Policies with the PlacementRules. 

These objects have to be placed on a Git repository/directory controlled by an ArgoCD App. It could be the same you are using for your PGTs. These objects will be synchronized by ArgoCD as usual Kubernetes/Openshift objects. ACM controllers will make the reconciliation accordingly. 

#### Creating a Policy

A Policy is an object with a set of 'must have' and/or 'must not have' of any Kuberentes/Openshift object.

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
    ran.openshift.io/ztp-deploy-wave: "2"
  name: extra-project-create
  namespace: policies-site
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: testing-policy-project
      spec:
        namespaceselector:
          exclude:
          - kube-*
          include:
          - '*'
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: project.openshift.io/v1
            kind: Project
            metadata:
              name: foo
            status:
              phase: Active
        remediationAction: inform
        severity: low
  remediationAction: inform
```

In this example:

* Name: I have put a prefix 'extra-' to the name. Just to differentiate this Policy from the ones created by ZTP Tooling using PGTs. But it is just a suggestion or a personal convention. 

* complianceType: the object musthave or mustnothave the objectDefinition.

* objectDefinition: it will create a Project called 'foo'. 
  
  ```yaml
  apiVersion: project.openshift.io/v1
  kind: Project
  metadata:
    name: foo
  status:
    phase: Active
  ```

* remediationAction: from ZTP4.10, all the Policies are created with remediation Inform. This is hidden inside the existing PGT templates. When you create raw ACM Policies, you have to set it to Inform. Later, TALM will manage when to Enforce the Policies

* severity is just informational

#### Making the match between policies and clusters

ZTP Gitops Tooling proposes 3 kind of PGT about which clusters are affected. And these are managed by a convencion about labeling clusters on Siteconfigs:

* SiteSpecific Policies. Labeling clusters on the way of:  'name: "CLUSTER_NAME"'

* Group Policies. Labeling clusters on the way of: '<GROUP_POLICY_NAME>: ""'

* Common Policies. Labeling clusters on the way of: '<COMMON_POLICY_NAME>: "true"'

Then ZTP Tooling will configure the different PlacementRule/PlacementBinding. When using raw ACM Policies, you have to create PlacementRule/PlacementBinding manually together with the Policy.

##### Applying Policies to Site Specific

With the above Policy created, we have to match the PlacementRule/PlacementBinding to be applied to an specific site. Consider we have one cluster which is called 'intel-1-sno-1'. This match will fire the configuration only on this cluster.

`PlacementRule:' to an specific site

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: extra-intel-1-sno-1-placementrules
  namespace: policies-site
spec:
  clusterSelector:
    matchExpressions:
      - key: name
        operator: In
        values:
          - intel-1-sno-1
```

You are just creating a link, called 'extra-intel-1-sno-1-placementrules', with the cluster 'intel-1-sno-1'. 

`PlacementBinding:' it will relate a policy (or a a list of policies) with a 'PlacementRule'

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: extra-intel-1-sno-1-placementbinding
  namespace: policies-site
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: extra-intel-1-sno-1-placementrules
subjects:
  - apiGroup: policy.open-cluster-management.io
    kind: Policy
    name: extra-project-create
```

Here we are linking the previous selected cluster and the Policy called site-project.

With all these objects created, you will see how the Policy is created on ACM but affecting only one cluster.

##### Applying Policies to Groups

We can use the same previous created Policy, but this time we configure the PlacementRule/PlacementBinding to match a group of clusters. 

A group of clusters, according to ZTP Tooling convention, means clusters labeled in the way of: '<GROUP_POLICY_NAME>: ""'

`PlacementRule:' to match clusters with the label group-du=""'

```yaml
apiVersion: apps.open-cluster-management.io/v1                                   
kind: PlacementRule                                                               
metadata:                                                                         
  name: extra-intel-1-sno-1-placementrules                                        
  namespace: policies-group                                                       
spec:                                                                             
  clusterSelector:                                                                
    matchExpressions:                                                             
      - key: group-du                                                           
        operator: Exists                                                              
```

`PlacementBinding:' here there are no changes, you are just linking the Policy with the PlacementRule

```yaml
apiVersion: policy.open-cluster-management.io/v1                                  
kind: PlacementBinding                                                            
metadata:                                                                         
  name: extra-intel-1-sno-1-placementbinding                                      
  namespace: policies-group                                                       
placementRef:                                                                     
  apiGroup: apps.open-cluster-management.io                                       
  kind: PlacementRule                                                             
  name: extra-intel-1-sno-1-placementrules                                        
subjects:                                                                         
  - apiGroup: policy.open-cluster-management.io                                   
    kind: Policy                                                                  
    name: extra-project-create  
```

##### Applying Policies to Common

We can use the same previous created Policy, but this time we configure the PlacementRule/PlacementBinding to match a Common Policy. 

A Common Policy, according to ZTP Tooling convention, means clusters labeled in the way of:  '<COMMON_POLICY_NAME>: "true"'

`PlacementRule:' to match clusters with the label 'common="true"'

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: extra-intel-1-sno-1-placementrules
  namespace: policies-common
spec:
  clusterSelector:
    matchExpressions:
      - key: common
        operator: In
        values:
          - "true"
```

`PlacementBinding:' here there are no changes, you are just linking the Policy with the PlacementRule

```yaml
apiVersion: policy.open-cluster-management.io/v1                                  
kind: PlacementBinding                                                            
metadata:                                                                         
  name: extra-intel-1-sno-1-placementbinding                                      
  namespace: policies-group                                                       
placementRef:                                                                     
  apiGroup: apps.open-cluster-management.io                                       
  kind: PlacementRule                                                             
  name: extra-intel-1-sno-1-placementrules                                        
subjects:                                                                         
  - apiGroup: policy.open-cluster-management.io                                   
    kind: Policy                                                                  
    name: extra-project-create  
```

### Hub cluster templates

[Hub cluster templates](https://open-cluster-management.io/concepts/policy/#policy-templates) for Policies are good option to better customize your desired status. This is a more advanced option, that provides a great level of flexibility. A Policy about configuring networks, would template the configuration parameters depending, for example, on each node. 

It is mainly based on the usage of the delimiter {{hub … hub}} and Golang text template specifications.

# Some more examples

Following, a list of advanced configuration examples, based on the previous explained options.

I am not including the PlacementBinding/PlacementRule to facilitate the readiness. But it is just a matter of making each policy to match clusters or groups. Explained above.

## Node labeling (ACM Policies with hub cluster templates)

## Deleting a Node from a cluster (ACM Policies)

This is a pretty straight forward example, just based on ACM Policy for Site Specific.

```yaml
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
    ran.openshift.io/ztp-deploy-wave: "20"
  name: node-drain-remove
  namespace: ztp-site
spec:
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: node-drain-remove-configurationpolicy
      spec:
        namespaceselector:
          exclude:
          - kube-*
          include:
          - '*'
        object-templates:
        - complianceType: mustnothave
          objectDefinition:
            apiVersion: machine.openshift.io/v1beta1
            kind: Machine
            metadata:
              name: el8k-ztp-1-worker-1.el8k-ztp-1.hpecloud.org
              namespace: openshift-machine-api
        remediationAction: inform
        severity: low
  remediationAction: inform
```