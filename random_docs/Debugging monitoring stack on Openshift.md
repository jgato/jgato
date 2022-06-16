# Debugging monitoring stack on Openshift

The following tutorial will cover a possible way of debugging Openshift monitoring stack (aka Prometheus/NodeExporter). Actually, our main objective is the node_exporter pod.

We are trying to simulate a Telco/RAN environment, where optimizing the platform is really important. 

The tutorial will cover 3 different steps

1. Lets burn our cluster

2. Get a cpu profile

3. Some possible tuning

Consider a previous step 0) for creating an Openshift Cluster. In this case I will use a cluster with these characteristics: 

* OCP Standard Cluster with 3 Master and 2 Workers



![](assets/2022-06-16-09-41-59-image.png)

* The cluster is not running any specific workload and it is pretty clean. A part from the usual [Telco/RAN operators distributed](https://www.redhat.com/en/resources/5G-open-ran-deployment-overview)

* The worker nodes will have a workload cpu partitioning. You can read more about this [here](https://docs.openshift.com/container-platform/4.6/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html#cnf-restricting-cpu-infra-container_cnf-master) and [here](https://docs.openshift.com/container-platform/4.10/scalability_and_performance/sno-du-enabling-workload-partitioning-on-single-node-openshift.html). This is managed by one of the operators in the previous bullet point (Performance Addon Operator)

This last point is key for this debugging. Some systems (like Telco RAN) need to optimize platform at maximum. So only an small set of CPUs are assigned for platform (OS, Openshift, etc) and the others are used by the user to run their workloads on Openshift. In these scenarios, optimizing each component of Openshif is need it. 

## Lets burn the cluster

Lets start creating some loads into the cluster. In order to make this, I will use [kube-burner](https://github.com/cloud-bulldozer/kube-burner): *Kube-burner is a tool aimed at stressing kubernetes clusters*

The installation is pretty easy, you can just download pre-compiled binaries from [here](https://github.com/cloud-bulldozer/kube-burner/releases)

```bash
$ ./kube-burner  version
Version: 0.16
Git Commit: deaa2793742d9c89b8c277801a31912cddc2ab0d
Build Date: 2022-06-08T10:10:31Z
Go Version: go1.16.15
OS/Arch: linux amd64
```

kube-burner allows you to define a set of jobs that will create different Kubernetes objects with a set of conditions. The framework provides some pre-created examples that will help you to start working. 

As starting point we used [kubelet-density-heavy](https://github.com/cloud-bulldozer/kube-burner/tree/master/examples/workloads/kubelet-density-heavy) which basically creates a client/server app: PostreSQL server and client deployments and a Kubernetes service. 

Some basic configuration params for this example:

```yaml
jobs:
  - name: kubelet-density-heavy
    jobIterations: 280
    qps: 25
    burst: 25
    namespacedIterations: false
    namespace: kubelet-density-heavy
    waitWhenFinished: true
    podWait: false
```

* jobIterations: how many times the job will be executed. Each job will execute n times its 'objects' section:

```yaml
    objects:
      - objectTemplate: templates/postgres-deployment.yml
        replicas: 1
      - objectTemplate: templates/app-deployment.yml
        replicas: 1
      - objectTemplate: templates/postgres-service.yml
        replicas: 1
```

> **Note**
> 
> It will create 280 times a replica of the postgres-deployment, app-deployment and the service. 

> If you check these templates, you will just see regular yaml templates with Kubernetes objects. 

* qps/burst: to manage/limit the number of requests

* namespacedIterations if 'true' it will create a new namespace by each job repetition. 

* waitWhenFinished: it just waits after all the jobs are done

* podWait if 'true' it waits for all the pods of each iteration to be running, before going with the next one. Here it is false, so we will create more pods at the same time ;)

I have done some modifications to this example to allow you to select which node will run all the jobs. Why all in the same node? 

* I want to optimize node_exporter, which runs individually in each node.

* Each node would have different characteristics, hardware, performance profiles, loads, etc.

* So, lets debug node_exporter according to each node

The kube-burner jobs that I will create can be found [here]()

What is actually this example doing: it is creating a pair client/server pods. Because it is configured with 280 iteractions, it will create 280 clients and 280 serves. Also, I am placing all the pods in the same host. Therefore, the host will have to have capacity for at least 560 pods.

About the limit of pods, by default OCP has a limit of 252 per node. We will change that following this interesting [article](https://cloud.redhat.com/blog/500_pods_per_node). I have increased pods limit to 400, which does not require to provide more IP Addresses. 

The experiment will push the cluster limits. It is not only about the number of pods, also, it depends on the communication capacities between 'kubelet' and 'kube-apiserver' and the number of requests. This can be tuned with the qps/burst. There is a great article about the [limits of running pods](https://cloud.redhat.com/blog/500_pods_per_node). 

### Burning worker-0

The starting point of this node is pretty clean, a part from some Operators used on Telco/RAN environments.

```bash
$ oc debug node/worker-0.el8k-ztp-1.hpecloud.org                                                                                                
Starting pod/worker-0el8k-ztp-1hpecloudorg-debug ...                                                                                                                           
To use host binaries, run `chroot /host`                                                                                                                                                                                                                      
chPod IP: 2620:52:0:1351:67c2:adb3:cfd6:81                                                                                                                                                                                                                    
If you don't see a command prompt, try pressing enter.                                                                                                                                                                                                       
sh-4.4# chroot /host                                                                                                
sh-4.4# crictl ps | wc -l                                                                                           
62                                                                                                                  
sh-4.4# ip link show 2>/dev/null | wc -l                                                                            
50                              
```

* prometheus cpu consumption about 5%-8%

* node_exporter has peaks of 5%-7% each time is requested to provide metrics

Using my custom kube-burner job, I have configured it in that way:

```yaml
  - name: kubelet-density-heavy
    jobIterations: 170
    qps: 50
    burst: 100
    namespacedIterations: false
    namespace: kubelet-density-heavy
    waitWhenFinished: true
    podWait: false
    objects:

      - objectTemplate: templates/postgres-deployment.yml
        inputVars:
          nodeName: worker-0.el8k-ztp-1.hpecloud.org
        replicas: 1

      - objectTemplate: templates/app-deployment.yml
        replicas: 1
        inputVars:
          readinessPeriod: 10
          nodeName: worker-0.el8k-ztp-1.hpecloud.org
```

We will check how kube-burner will be consuming resources:

![](assets/2022-06-15-17-13-31-image.png)

* 'top-left' pan contains: number of pods, containers and network interfaces in the objetive node

* 'top-right' it shows the number of pods created by kube-burner, which are already running. The final objective will be '340'

* 'bottom-left' it just runs kube-burner

* 'bottom-right' we can see top processes on the node

We lunch the experiment:

```bash
$ ~/kube-burner/kube-burner init -c ./kubelet-density-heavy.yml
```

and we observe the how the resources are consumed as long as pods are created.

![](assets/2022-06-15-17-51-01-output.gif)

When all the pods have been created the system new status is:

```bash
Pods:
377
Containers:
422
Network Interfaces:
740
```

This amount of Network Interfaces would be problematic, with node_exporter willing to inspect all of them. But we will see this later with the conclusions. Also, more tan 400 containers is an enough big load for such kind of server. Not something too high about cpu consuming, because these pods are not very demanding.

# CPU Profiling node_exporter process

## Delete the node-burner jobs

# Some useful scripts I use to monitor

Get some node statistics:

```bash
sh-4.4# watch -n 5 "
  echo 'Pods:'; \
  crictl pods | wc -l; \
  echo 'Containers:'; \
  crictl ps | wc -l; \
  echo 'Network Interfaces:'; \
  ip link show 2>/dev/null | wc -l;
"
```