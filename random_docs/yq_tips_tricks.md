# yq tips and tricks

Using yq from [mikefarah](https://github.com/mikefarah/yq)

## Modify yamls

 * Select elements that you pipe to xargs
 
```bash
> oc get managedcluster -o yaml | yq '.items[].metadata | select( .labels.vendor == "OpenShift" ) | .name' | xargs -n 1  -r  oc get managedcluster

```
 * Add an element to an array:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name:
  annotations:
    ran.openshift.io/ztp-deploy-wave: "2"
    my-key: myvalue
```

```bash
> yq e '.metadata.annotations.my-key = "myvalue"' customNamespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name:
  annotations:
    ran.openshift.io/ztp-deploy-wave: "2"
    my-key: myvalue
```

*use -i param to modify the file instead of output the result*

 * Append a value to the value of a key:
 
```bash
> yq e  '.metadata.annotations.my-key = .metadata.annotations.my-key + "+other-value"' customNamespace.yaml 
---
apiVersion: v1
kind: Namespace
metadata:
  name:
  annotations:
    ran.openshift.io/ztp-deploy-wave: "2"
    my-key: myvalue+other-value
```

## OpenShift/Kubernetes tricks

Get the pods that have restarted in the last 24 hours, and show, how many restarts they have accumulated. 

```bash
> oc get pods --all-namespaces --sort-by=.status.containerStatuses[0].restartCount -o json | \
jq '.items[] 
    | select(.status.containerStatuses[0].restartCount > 0) 
    | {
        namespace: .metadata.namespace,
        pod: .metadata.name,
        restarts: .status.containerStatuses[0].restartCount,
        lastRestart: .status.containerStatuses[0].lastState.terminated.finishedAt
      } 
    | select(.lastRestart != null and (now - (.lastRestart | fromdateiso8601) < 86400)) 
    | (.restarts | tostring) + " " 
      + .namespace + " " 
      + .pod + " last restart " 
      + ((now - (.lastRestart | fromdateiso8601)) / 3600 | tostring) + " hours"'

```