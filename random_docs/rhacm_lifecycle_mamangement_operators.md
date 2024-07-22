[draft][abstract]

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


Recently, RHACM Governance model has released a new API with this intention, called [OperatorPolicy](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.10/html/governance/governance#policy-operator). Still as tech preview, and still is not exploitable by ZTP and the PolicyGenTemplates. But, it is worthy to experiment with this new API. In order, to check if it is suitable for the Telco needs of creating blueprints, that validate solutions with an strong pinned versions of all its components.  

Recently, Red Hat Advanced Cluster Management (RHACM) Governance model has released a new API called  [OperatorPolicy](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.10/html/governance/governance#policy-operator), which aligns with this intention. Although it's still in technical preview and not yet exploitable by ZTP or PolicyGenTemplates, I believe it's worth experimenting with this new API to see if it's suitable for Telco needs, specifically creating blueprints that validate solutions with strong, pinned versions of all components.



**RHACM Goverance OperatorPolicy is still tech preview**
