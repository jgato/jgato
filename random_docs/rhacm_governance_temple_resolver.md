# Template resolver for RHACM

In order to make this tutorial quick an simple, it is expected the following knowledge:
 * You already know about RHACM (Red Hat Advanced Cluster Management), or the upstream Open Cluster Management.
 * You are familiar with the the RHACM Governance model (Policies).
 * You are familiar with the [Configuration Policy Templates](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.14/html-single/governance/index#template-processing)
 * (Optional) You are familiar with Policies generators as PolicyGentTemplates or PolicyGenerator.
 
If so, you are used to build extra logic to your policies like:

```
apiVersion: policy.open-cluster-management.io/v1                               
kind: Policy                                                                   
metadata:                                                                      
...
  name: common-config-policy                                                   
  namespace: vsno5                                                                                                                                                                                                                                            
...
spec:                                                                          
  disabled: false                                                              
  policy-templates:                                                            
  - objectDefinition:                                                          
      apiVersion: policy.open-cluster-management.io/v1                         
      kind: ConfigurationPolicy                                                
      metadata:                                                                
        name: common-config-policy-config                                      
      spec:                                                                    
...
        object-templates:                                                      
        - complianceType: musthave                                             
          objectDefinition:                                                    
            apiVersion: v1                                                     
            data:                                                              
              url: {{hub (index (lookup "cluster.open-cluster-management.io/v1" "ManagedCluster" "" .ManagedClusterName).spec.managedClusterClientConfigs 0).url hub}} 
                  
...

```

Here, I am adding a programmatically way of getting a basedomain for configuring correctly a destination url.

I dont go with the details, the syntax, or the understanding of all the different functions that you can use for templating `{{..}}` or `{{hub}` templating. 

If you are used to build this kind of templates, you know there is a lot of try and error. Create the Policy, upload it to the hub, let ACM to interpret it, and see the resolved result. If you are using generators (like my case) PGT or PolicyGenerator, this adds more extras. You write the generator, that creates the policy, ACM has to render it, etc, etc.

There exists a template resolver tool that lets work with this locally first.

Install the tool ([more about template resolver in the official doc](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.12/html-single/governance/index#policy-cli-commands):

```
go install github.com/stolostron/policy-cli/cmd/policytools@latest
```

Write your Policy without generators. Or extract it using your generator.

My previous Policy:

```
...
  url: {{hub (index (lookup "cluster.open-cluster-management.io/v1" "ManagedCluster" "" .ManagedClusterName).spec.managedClusterClientConfigs 3).url hub}} 
...
```

How can I try if this will work (locally):

```
> template-resolver --cluster-name vsno5 --hub-kubeconfig ~/kubeconfig-hub-2 --hub-namespace vsno5 /tmp/policy.yaml 
```

That returns:
> ``` error: template: tmpl:51:38: executing "tmpl" at <index (lookup "cluster.open-cluster-management.io/v1" "ManagedCluster" "" .ManagedClusterName).spec.managedClusterClientConfigs 3>: error calling index: index out of range: 3```

Upss, I am out of index, lets try:

```
...
  url: {{hub (index (lookup "cluster.open-cluster-management.io/v1" "ManagedCluster" "" .ManagedClusterName).spec.managedClusterClientConfigs 0).url hub}} 
...
```

and the template resolver returns the full Policy rendered. The part we are interested in:

```
  url: https://api.vsno5.spoke.el8k.se-lab.eng.rdu2.dc.redhat.com:6443

```

Now, it is easy to build and debug this templates locally.

## What if you are using a generator

In my case, I am used to use PTG to build the Policies. So, I can extract the generator (there are many ways), and execute it locally to return the raw ACM Policy:

```
> podman run --log-driver=none --rm registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.18 extract /home/ztp --tar | tar x -C /tmp/out
> /tmp/ztp-kustomize-plugin/ran.openshift.io/v1/policygentemplate/PolicyGenTemplate ./common.yaml |\
 yq 'select(.kind == "Policy" and .metadata.name == "common-config-policy")' |\ 
 template-resolver --cluster-name vsno5 --hub-kubeconfig ~/Servers/EL8000-2/kubeconfig-el8k-hub-2 --hub-namespace ztp-common -
```