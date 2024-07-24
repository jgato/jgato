[draft][abstract]

This document is just a first experiment on the new [RHACM Governance OperatorPolicy API](https://docs.openshift.com/container-platform/4.16/edge_computing/policygentemplate_for_ztp/ztp-configuring-managed-clusters-policies.html). This API would allow us to implement the [Ability to declare Operator versions through GitOps/ZTP workflow](https://issues.redhat.com/browse/RFE-4890).

The objective is to test this new feature, and to detect the main missing integration points between ZTP GitOps and the new API. Due to, this API is not exploitable yet from ZTP.

This document is not a real tutorial on how to use OperatorPolicies from ZTP (but it would help you on some manual work to do it), because this is not implemented as today. 


# Lifecycle management of Openshift operators with RHACM and ZTP

[Red Hat Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management) (ACM) allows you to deploy, upgrade, and configure different spoke clusters from a hub cluster. It's an OpenShift cluster that manages other clusters. For all management, one can imagine many APIs exist to complete all the needed features. Together with RHACM, we use [Zero Touch Provisioning](https://docs.openshift.com/container-platform/4.16/edge_computing/ztp-preparing-the-hub-cluster.html) (ZTP), which enables us to facilitate interaction with these resources. Additionally, using a GitOps methodology allows for life cycle management in a more DevOps way. ZTP includes a set of helpers to define your infrastructure and configurations with a scalable approach, and is very focused on Telco scenarios and needs.

One of the helpers for life cycle management between ZTP and RHACM APIs are the resources needed to install and upgrade different Telco operators, such as those for [PTP](https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs/PtpSubscription.yaml) or [SRIOV(https://github.com/openshift-kni/cnf-features-deploy/blob/master/ztp/source-crs/SriovSubscription.yaml. ZTP provides an API called [PolicyGenTemplate](https://docs.openshift.com/container-platform/4.16/edge_computing/policygentemplate_for_ztp/ztp-configuring-managed-clusters-policies.html) that allows you to 
include these resources, which can then be committed to a Git repository.


```yaml
apiVersion: ran.openshift.io/v1
kind: PolicyGenTemplate
metadata:
  name: "common"
  namespace: "ztp-common"
spec:
  bindingRules:
    # These policies will correspond to all clusters with this label:
    common: "true"
  sourceFiles:
    # Create operators policies that will be installed in all clusters
    - fileName: SriovSubscription.yaml
      policyName: "subscriptions-policy"
      spec:
        source: redhat-operators
	- fileName: PtpSubscription.yaml
	  policyName: "subscriptions-policy"
````

There is still another component in place. [The Topology Aware Lifecycle Manager](https://www.redhat.com/en/blog/how-to-use-the-topology-aware-lifecycle-manager) (TALM) will be in charge, once the cluster is deployed, to make compliant the PolicyGenTemplates and RHACM Policies ([RHACM Governance model](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html-single/governance). When having to install a new operator:

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-ptp
  namespace: openshift-ptp
spec:
  channel: "stable"
  name: ptp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: ptp-operator.v4.14.0-202404250639
status:
  state: AtLatestKnown
```

TALM and Openshift [OLM](https://docs.openshift.com/container-platform/4.16/operators/understanding/olm/olm-understanding-olm.html) will try to get the latest available version, even if you specific a concrete one with `startingCSV`.
This is a know [limitation](https://issues.redhat.com/browse/OCPBUGS-22838) and there is a [Request Feature Enhancement](https://issues.redhat.com/browse/RFE-4890) about it. 

This approach of always having the latest available version works well in most scenarios, based on the idea that "latest is always best". However, if something goes wrong, it will likely be fixed in a subsequent version. In Telco scenarios, where we need to validate hardware, platform, and workloads, it's crucial to release a validated solution with specific versions of each component, as opposed to relying solely on the latest available version.


Recently, Red Hat Advanced Cluster Management (RHACM) Governance model has released a new API called  [OperatorPolicy](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html/governance/governance#policy-operator), which aligns with this intention. Although it's still not yet exploitable by ZTP or PolicyGenTemplates, I believe it's worth experimenting with this new API to see if it's suitable for Telco needs. In order to create blueprints that validate solutions with strong pinned versions of all components.

## Using the new API

With ZTP and PolicyGenTemplate (PGT), we are used to create whatever needed Openshift/Kubernetes object as a manifest inside a `source-crs` folder. So, we could just create a new manifest source, that could be linked from a PGT. PGTs are managed by a ZTP kustomize generator plugin that transforms the PGTs and creates a RHACM `Policy`and the desired objects inside a `ConfigurationPolicies`. Simplified: `Policy.object-definition.ConfigurationPolicy.object-templates[].object`. PGTs allows you easily to fill the list of objects to be managed by the RHACM Governance. 

But, an [`OperatorPolicy` cannot be encapsulated under a ConfigurationPolicy](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html/governance/governance#create-operator-policy), that is the only way of working with the PGT generator. And therefore, we cannot create `OpertorPolicies` from a PGT.  interacting with RHACM Governance. And therefore, 


> Here we find the first lack of feature, PGT generator plugin encapsulate all the CRs as a `ConfigurationPolicy`. This is oka for encapsulating the creation/configuration of any Openshift/Kubernetes object. But, `OpertorPolicy` works differently:  [ToDo] Link to an RFE 


Anyway, ZTP is not only a set of generators, it is also a methodology. If we cannot manage the `OperatorPolicy` by a PTG, we still can just directly use the RHACM Governance API. Placing the needed manifests inside our ZTP Git repository. 

It is not so easy as using a PGT and it makes us to directly play with the RHACM Governance API. In this case: `OperatorPolicies` and the `PlacementBindings` and  `PlacementRules`. 

```
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: common-ran-operators
  namespace: ztp-common
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1beta1
        kind: OperatorPolicy
        metadata:
          name: install-operators
        spec:
			...
			...
			...
```

Using the placements, we will select the clusters affected by the `OperatorPolicies`, that we will have to also create manually:
 
```
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: common-ran-operators-placement
  namespace: ztp-common
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common
              operator: In
              values:
                - 'true'
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: common-ran-operators-placement-binding
  namespace: ztp-common
placementRef:
  name: common-ran-operators-placement
  apiGroup: cluster.open-cluster-management.io
  kind: Placement
subjects:
  - name: common-ran-operators
    apiGroup: policy.open-cluster-management.io
    kind: Policy

```


There is also another extra configuration derived of using `OperatorPolicies`. `ConfigurationPolicies` match affected clusters with `PlacementBindings` and the `PlacementRules`. `OperatorPolicies` need also to mach a RHACM `ClusterSet`:

```yaml
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: common-ran-operators-placement
  namespace: ztp-common
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common
              operator: In
              values:
                - 'true'
  clusterSets:
    - default
```

If you are used to use ZTP, maybe you are not very aware of the `ClusterSet`API, because it is not needed. But anyway, there is a default one that includes all the created clusters. We can use this default one, setting it also to be able to access the Namespace where the Policies will be places. In our case, `ztp-common` Namespace.

![](assets/rhacm_lifecycle_mamangement_operators_20240722101443206.png)

> Second issue, a `ClusterSet` needs to be created/modified with a `ClusterSetBinding` to the usual Namespaces we use with ZTP: ztp-common, ztp-group, etc. 

With the `ClusterSet` accessing the Namespaces of the `Policies` that we will create, we can start creating our own `OperatorPolicies`.

## Create `OperatorPolicies` to manage the Operators lifecycle

Following an example, of an `OperatorPolicy` that install the PTP operator on an specific version:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: common-ran-operators
  namespace: ztp-common
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1beta1
        kind: OperatorPolicy
        metadata:
          name: install-operators
        spec:
          complianceType: musthave
          remediationAction: enforce
          severity: critical
          upgradeApproval: None
          operatorGroup:
            name: openshift-ptp
            namespace: openshift-ptp
            targetNamespaces:
            - openshift-ptp
          subscription:
            channel: stable
            name: ptp-operator
            source: redhat-operators
            sourceNamespace: openshift-marketplace
            startingCSV: ptp-operator.v4.14.0-202404250639
          versions:
          - 4.14.0-202402211209
          - 4.14.0-202406051038
          - 4.14.0-202407021509
          - 4.14.0-202312062209
          - 4.14.0-202404161544
          - 4.14.0-202404250639
          - 4.14.0-202406180839
```

 

Some comments on the Manifest for the `OperatorPolicy`:
 * `remediationAction: inform` following the ZTP way of doing. Policies are always inform, and the user can use TALM to decide when (And to which clusters) we want to remediate/apply the Policy. 
 * `upgradeApproval: none` , when TALM takes a Policy from inform to enforce to start the remediation, different InstallPlans for the Operator will be created. None means these InstallPlans are not automatically approved. Again, this is compatible with the ZTP approach. Where TALM is in charge of approving the InstallPlans. **Note: how is anyway the way of indicating you want to upgrade? if many, which IP will approve TALM? maybe TALM is still not ready for this?**
 * `subscription` is the usual specification for an operator `subcription`. Here `startingCSV` will be the version that will be used to configure the cluster right after the installation. 
 * `versions`: later, during day-2, the Operator would be upgraded. If the running operator's version belongs to any of the list, the `Policy` will remain complaint. This is a good way of ensuring the operator always runs in a validated version. 

In summary, this `OperatorPolicy` will install the operator PTP with the version `v4.14.0-202404250639`. But we also allow, as compliant, other validated versions. 


Also, we create another Policy->ConfigurationPolicy to manage the creation of the Namespace needed to install the Operator. 

```yaml

    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: install-operators-ns
        spec:
          remediationAction: inform
          evaluationInterval:
            compliant: 10m
            noncompliant: 10s
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: Namespace
                metadata:
                  name: openshift-ptp

```

We keep Policies to inform, so, for demoing we can manually force them when needed. We also get rid of the `ztp-deploy-wave` annotation. Therefore, TALM will ignore these `Policies` and we can just manage them with the RHACM Governance API.

Which clusters are affected by these two `Policie`?. We use the `PlacementBindings` and the `PlacementRules`. Basically, any of our clusters with the label `common`:

```yaml
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: common-ran-operators-placement
  namespace: ztp-common
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common
              operator: In
              values:
                - 'true'
  clusterSets:
    - default                
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: common-ran-operators-placement-binding
  namespace: ztp-common
placementRef:
  name: common-ran-operators-placement
  apiGroup: cluster.open-cluster-management.io
  kind: Placement
subjects:
  - name: common-ran-operators
    apiGroup: policy.open-cluster-management.io
    kind: Policy
  - name: common-ran-operators-ns
    apiGroup: policy.open-cluster-management.io
    kind: Policy
```


Placing all these CRs in our ZTP Git Repo will configure the RHACM Governance accordingly. 

## Enforcing and remediating the Policy

As explained above, we cannot use TALM to manage the remediation of our Policies. But the Policies are created anyway, and we can just enforce them to do the remediation. As TALM would ideally do in a more controlled way.

The Policy created has two different templates: the Namespace (`Policy->ConfigurationPolicy`) and the OperatorPolicy (`Policy->OperatorPolicy`):


We can enforce both in order to install the ptp-operator:


Notice how the OperatorPolicy will create all the different resources, including `Subscriptions`, `ClusterServiceVersion`, `OperatorGroup`, etc. But more important, it creates the needed `InstallPlan` but it will approve only, the one set with the `startingCSV`.

```yaml
> oc -n openshift-ptp get subscriptions,csv,ip 
NAME                                             PACKAGE        SOURCE             CHANNEL
subscription.operators.coreos.com/ptp-operator   ptp-operator   redhat-operators   stable

NAME                                                                           DISPLAY        VERSION               REPLACES   PHASE
clusterserviceversion.operators.coreos.com/ptp-operator.v4.14.0-202404250639   PTP Operator   4.14.0-202404250639              Succeeded

NAME                                             CSV                                 APPROVAL   APPROVED
installplan.operators.coreos.com/install-nkcnt   ptp-operator.v4.14.0-202404250639   Manual     true
installplan.operators.coreos.com/install-p8fwp   ptp-operator.v4.14.0-202407021509   Manual     false

```

After a while, the operator is installed and available.


![](assets/rhacm_lifecycle_mamangement_operators_20240724154654166.png)

## Upgrading the operator

To install an operator on an specific version is part of the functionality that we need. But, what about upgrading the operator, and again. to a new specific version and not the last available one?


## Improvements in ZTP to easiness all the process:

 * PGT generator to support encapsulate OperatorPolicies.
 * Configure `ClusterSet` and `ClusterSetBindings`
 * TALM to manage Policies with two different kinds of PolicyTemplates (OperatorPolicies and ConfigurationPolicies) inside the same Policy
 
 
 Bugs found:
  * [TALM policy objects inspect wrong mesage](https://issues.redhat.com/browse/OCPBUGS-37466)