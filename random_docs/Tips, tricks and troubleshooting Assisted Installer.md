# Tips, tricks and troubleshooting Assisted Installer

Some tips, tricks and troubleshooting that would help deployment OCP with AI.

## AI Kubernetes API based

Here, our main way of interacting is using Openshift/Kubernetes resources. As the way of deploying the cluster.

## Agent Based AI

With  Agent Based AI, the bootstrap node exposes the Assisted Service REST API. This AI is easily accesible. It provides information about the clusters, infraenvs and hosts. The same than Kuberentes API, but using a REST API, instead of Openshift/Kubernetes resources.

Of course, also logs, from the different services, helps with the troubleshootings

### Interacting with the Assisted Service REST API

You have to SSH into the host which is the boostrap, and it contains the assisted-service. 

#### How to know the installation status:

You can get the validation info of each host:

```bash
#> curl -s http://localhost:8090/api/assisted-install/v2/infra-envs/{INFRAENV-ID}/hosts | jq -r .[].validations_info | jq
```

you can make it more selectively: 

```bash
#> curl -s http://localhost:8090/api/assisted-install/v2/infra-envs/{INFRAENV_ID}/hosts/{HOST_ID} | jq -r .validations_info | jq
```

![](assets/2023-02-15-12-00-42-image.png)

Get just the status and status info of each host:

```bash
#> curl -s http://localhost:8090/api/assisted-install/v2/infra-envs/{INFRAENV-ID}/hosts | jq  '.[] | .requested_hostname + ": " + .status + " " + .status_info'
```

![](assets/2023-02-15-12-06-17-image.png)

#### How to get the Connectivity Groups

During installation, there are some tests about the connectivity between all the nodes to be installed. In case of dual stack, it includes also connectivity in both interfaces.

```bash
#> curl -s http://localhost:8090/api/assisted-install/v2/clusters | jq -r .[].connectivity_majority_groups | jq
```

In this example, we have 3 hosts, so all have connectivity in all groups.

![](assets/2023-02-15-12-19-41-image.png)
