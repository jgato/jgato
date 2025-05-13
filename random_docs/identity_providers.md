# Openshift identity providers and users

Small tutorial on how to create an identity provider (htpasswd) and users.

First we create an htpasswd file:

```bash
>  htpasswd -c -B -b /tmp/htpasswd jgato jgato
Adding password for user jgato
>  htpasswd  -B -b /tmp/htpasswd kubeadmin kubeadmin
Adding password for user kubeadmin
> cat /tmp/htpasswd 
jgato:$2y$05$gBNxv0MT2Oe5iKxk019rp.Xv4xeT0qIDOg5uQ4hJ/fHNYZLSMGDnK
kubeadmin2:$2y$05$J1oci9OfOIEI.r1wgSIF.OA/5p4OlK8RJjNmvNza32uwogBOzX9jC

```

This needs to be added as a secret on `openshift-config`NS:

```bash
> oc create secret generic htpass-secret --from-file=htpasswd=/tmp/htpasswd -n openshift-config
secret/htpass-secret created
```

Then, we have to edit the Oauth CR to add an `identityProvider` for HTPasswd pointing to that Secret:

```yaml
> oc get oauth cluster -o yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
<REDACTED>
  name: cluster
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: htpass-secret
    mappingMethod: claim
    name: my_identity_provider
    type: HTPasswd

```
