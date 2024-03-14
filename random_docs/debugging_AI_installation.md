*Disclaimer: This is not an official Red Hat documentation. Just our experience working on this kind of issues. There are thousands of combinations and cases that we cannot cover on this tutorial. But, it would help you with some tips on you debugging issues creating clusters. The work done here used OCP 4.13, ACM 2.7 and ZTP 4.13*

# Debugging clusters deployment with Openshift Assisted Installer and baremetal servers

Openshift Assisted Installer is an installation method for different Openshift cluster's flavours. The Assisted Installer can be used in different ways, like the [AI SaaS](https://docs.openshift.com/container-platform/4.13/installing/installing_on_prem_assisted/installing-on-prem-assisted.html), with a cloud platform that helps you to manage your cluster's deployment. Or, it can be used on premises, running on your own Management/Hub Openshift cluster.

In this tutorial, we will focus on this last option. When you have your own, on premises, Openshift Managment cluster. The cluster's deployment are done by the [Assisted Installer from the multicluster engine for Kubernetes](https://cloud.redhat.com/blog/multicluster-engine-and-assisted-installer-integration-deploy-and-expand-clusters-easily). This way of deploying cluster is also known as ZTP (Zero Touch Provisioning). 

We will cover  some tips to do debugging during the installation. Depending on if you are using ZTP, or ZTP GitOps. Two similar installation methods that make usage of different CustomResources.

It is important to differentiate when the installation failed. The AI starts the installation booting a RHCOS Discovery ISO, that basically, collects information from each node, contacts the AI and validates the installation. This can be known as the Discovery phase. After that, it happens the first nodes reboot. This time, it will use a RHCOS installed operative system, that will proceed with the real cluster installation. We could say, this is the Installation phase.

# Why debugging is complex

In this kind of installations you usually will make use of **remote baremetal servers**. If something goes wrong, it is difficult to debug:
 * By default, RHCOS creates an unique `core` user without password. So you dont have a kind of `root`user. But can login with an ssh key.
 * If the network configuration fails, and this will be your main headache, you cannot ssh there. 
 * The Discovery ISO, by default, it is a minimal ISO to facilitate booting from it with a BMC. Just after booting, starts downloading the full iso. If the network fails, it wont download it. You will have a minimal ISO, not a fully OS loaded, neither ssh, or other similar tools.
 * You dont have physical access, so you will use a virtual console on the BMC of the baremetal server. This virtual consoles, usually, responds pretty slow.

What are we going to try to do in this tutorial?
 * To do the Discovery phase with a full ISO, instead of the default minimal one. This can be only considered for debugging. Full ISOs are not good friends on BMC and its virtual media. It is not recommended, or allowed, to download 1GB ISO per each server, on a BMC network. This is not an expected behaviour. Some BMCs would not allow that, or maybe the connection is not good enough to download this size and boot with the virtual media. Advantages of that:
   * We boot the server with a full OS. All the tools are there, it does not need to download a big `rootfs`, etc
   * If there are network problems, at least, it will not get stuck downloading the whole OS
 * To get a root console. This is our main objective. To use the BMC Virtual Console and get a root console.
 
The combination of a `root` console, a whole OS installed will help us a lot on debugging. Then, how to fix the different possible problems is not covered here. But you will have the tools to proceed. Usually, you are reading this because of some kind of networking problems. So, with the root console and the OS installed, you can check what is failing about the network.

Next sections will help you how to get both advantages: full ISO and root console. Consider, full ISO as optional. Maybe, in your scenario you cannot boot with a full ISO anyway. 

# Debugging Discovery phase


## Enable the full ISO 

*This is an experimental feature of the Assisted Installer, how to enable it it would change. More info [here](https://github.com/openshift/assisted-service/blob/master/docs/operator.md#specifying-environmental-variables-via-configmap)*

Basically, we enable a feature that allows to enable different parameters of the Assisted Installer from a ConfigMap.

This ConfigMap will enable the Full ISO provisioning. Notice, this will affect any new deployment (disable this after your debugging sessions)

```yaml
apiVersion: v1     
kind: ConfigMap                                                                
metadata:                                                                      
  name: my-assisted-service-config                                             
  namespace: multicluster-engine                                               
data:                                                                          
  ISO_IMAGE_TYPE: full-iso                                                                                            
```

Then enable AI to use this CM:

```bash
$> oc annotate --overwrite AgentServiceConfig agent unsupported.agent-install.openshift.io/assisted-service-configmap=my-assisted-service-config
```

Then, we restart the AI service:

```
> oc rollout restart deployment/assisted-service -n multicluster-engine
deployment.apps/assisted-service restarted

```

After that, you can check BMH (Host Inventory CRs) and how it is using a full iso:

```bash
> oc -n el8k-ztp-1 get bmh master-0.el8k-ztp-1.hpecloud.org -o yaml | grep url

      url: https://assisted-image-service-multicluster-engine.apps.el8k.hpecloud.org/images/ec54f476-6535-4672-b7b2-b06265bb79db?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJlYzU0ZjQ3Ni02NTM1LTQ2NzItYjdiMi1iMDYyNjViYjc5ZGIifQ.7OynkZF4LFUnEVa6ju1d7cSWRIrzqVZBp_tTZLvjFSrmCmjdscVTUixUaSGt9AEKNONFZ3iC0H8MyJsky4fncg&arch=x86_64&type=full-iso&version=4.13 

> curl  -s -k -L -I "https://assisted-image-service-multicluster-engine.apps.el8k.hpecloud.org/images/ec54f476-6535-4672-b7b2-b06265bb79db?api_key=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpbmZyYV9lbnZfaWQiOiJlYzU0ZjQ3Ni02NTM1LTQ2NzItYjdiMi1iMDYyNjViYjc5ZGIifQ.7OynkZF4LFUnEVa6ju1d7cSWRIrzqVZBp_tTZLvjFSrmCmjdscVTUixUaSGt9AEKNONFZ3iC0H8MyJsky4fncg&arch=x86_64&type=full-iso&version=4.13" | grep content-length

content-length: 1119879168
```

The new ISO is bigger than 1GB, that would cause problems on some BMCs. This is why this is optional.

## Enable debug shell using the Host Inventory resources

Advantages: 
 * You can get easily a root console, not needing to enable any extra feature.
 * It happens after all bootstrap process, the file system is mounted, ignition files applied, etc. 
 * Very good option combined with the FULL ISO to solve some networking issues, dont getting stuck on rootfs download phase.
Disadvantages: 
 * it happens after bootstrap process, if the network is wrongly configured you will get stuck on downloading rootfs stage. 
 * Change to virtual (tty9) console could be "tricky" on some barmetal consoles.

Caution: To enable the `debug shell` or `root` console is done by the Infraenv CR. This CR affects to all the hosts in the cluster we are debugging. After enabling it, it will reboot all the hosts, to make them boot again in the Discovery Phase. But this time, all of them will have `root` console on tty9. So, maybe this will not help you on mutinode clusters on production. It is not a big issue if you are in a first cluster installation stage, when the cluster is not yet created, or working with SNOs. 

Use it only for debug, enable it, troubleshooting, and when everything is fixed, re-trigger the installation without this option. Dont forget to disable it, or you will have a huge free `root`access. 

```yaml
> oc -n el8k-ztp-1 edit infraenv el8k-ztp-1

<REDACTED>                                     
spec:                                                                          
<REDACTED>                                     
  kernelArguments:                                                             
  - operation: append                                                          
    value: systemd.debug-shell=1                                               
  - operation: append                                                          
    value: console=tty1              
<REDACTED>                                     
```

`systemd.debug-shell=1` will open the `root` console on tty9. The host will reboot with this kernel param. During booting process of the baremetal server, you can use the virtual console and 'Alt+F9' to switch the console. 

![](assets/root-console.gif)

I tried this with an ILO BMC, other providers would have different behaviour about how you switch ttyX.


`console=tty1` will output all the logs to the tty1. So, you can work on tty9 as `root`not dealing with flood of logs. 


## Interrupts RHCOS boot in different stages

We can use a kernel args `rd.break` on differen stages. Mor about the different stages [here](https://github.com/dracutdevs/dracut/blob/master/modules.d/99base/init.sh) or more "readable" [here](https://access.redhat.com/solutions/2382221)

rd.break={pre-udev|pre-trigger|initqueue|pre-mount|mount|pre-pivot|cleanup}

 * cmdline:		  Stops before parsing the kernel command line
 
 * pre-udev: 		  Stops before udevd is started
 * pre-trigger: 	Stops after starting udevd, setting udev environment variables
 * initqueue: 		Stops at the dracut main loop to find the real root
 * pre-mount: 	 	Stops before /sysroot is mounted
 * mount: 	 	    Stops after /sysroot is mounted
 * pre-pivot: 		Stops before switching root
 * cleanup: 	 	  Final cleanups before switching to rootfs init

One of our biggest issues happens if the network is not correctly configured, and the system tries to download the `rootfs`. This will end-up on an infinite loop trying to download the rootfs with the network wrong configured. This happens before `pre-mount`, so, usually we cannot reach the `pre-mount` and we have to use `initqueue`.

### Interrupt boot before mount rootfs happens

Advantages: 
 * You can get easily a root console, not needing to enable any extra feature.
 * It happens very early of bootstrap process, the `/sysroot` file system is not mounted, ignition files applied, etc. Only the temporal `initramfs` is mounted
 * The network has been already configured (no matter if wrong or bad)
 * It allows you to test your own network configurations*
 * Easy to change/set a root password

Disadvantages: 
 * None really, we interrupt the process very early but with Networking configured


In this example, I have deployed with ZTP an SNO with a wrong MAC address for the eno3 interface:

```yaml
  interfaces:
    - name: "eno3"
      macAddress: "94:40:c9:1f:bf:8f" # the correc one is 94:40:c9:1f:bf:8e"
  config:
    interfaces:
      - name: eno3
        type: ethernet
        state: up
        ipv4:
          enabled: true
          address:
            - ip: 10.19.10.74
              prefix-length: 26
        ipv6:
          enabled: false
    dns-resolver:
      config:
        server:
          - 10.19.10.71
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: 10.19.10.126
          next-hop-interface: eno3

```

Therefore, imagine the server does not boot, because it is never-ending trying to download the rootfs.

![](assets/no-download-rootfs.png)

We configure the `infraEnv` object to boot with the `rd.break=initqueue`. Add `.spec.kernelArguments` lines to your `infraEnv` object, this will make the server to reboot.

```yaml
spec:
  clusterRef:
    name: sno1
    namespace: sno1
  cpuArchitecture: x86_64
  ipxeScriptType: DiscoveryImageAlways
  kernelArguments:
  - operation: append
    value: rd.break=initqueue
  nmStateConfigLabelSelector:
    matchLabels:
      nmstate-label: sno1
  pullSecretRef:
    name: assisted-deployment-pull-secret

```


The server boots and you get the root console. 

![](assets/root-console.png)

In the `journalctl` you will see that NetworkManager tried to configure the network interface.

Using nmcli we can check some created connections:

![Alt text](assets/nm-cli-conn-show.png)

Where I can see that conn eno4 (that should be called eno3) is trying to use eno4 interface, that in my server is not even connected:

![](assets/nm-cli-eno4.png)

I can guess something was messed up because of there is no interface with the specified `macAddres`. If we check the Assisted Service network configuration you will see our initial wrongly configured interface on:

![](assets/nm-eno3-config.png)
![](assets/nm-mac-config.png)

In thi case, I have very clear what is the problem to fix. Usually you will play with nmcli to try to configure your desired configuration.

Here, you can edit the connection to use the eno3 interface and reload the conn. Or fix the Assisted Service files, in this case with a proper Mac addres, and then, kill and re-run the `/sbin/NetworkManager`. 

In this example I did the second, fixed the MAC address to point the correct one:

![](assets/nm-mac-config-correct.png)

Kill and re-run the NetworkManager with --debug. Now, the connection is correctly set on eno3:

![](assets/nm-cli-eno3.png)

Curl the Openshift API of the Hub:

![](assets/curl-oka.png)

`control+d` will continue the boot. Now, `rootfs` is downloaded and the boots continue without problems:

![](assets/boot-oka.png)

Dont forget to remove the kernel arguments, to break the boot, on the infraEnv.

## Enable a root console using ZTP GitOps resources

Using ZTP GitOps resources (Siteconfigs) you cannot enable the `root`console. Or at least, in the way we did it above. 

It should be needed to implement a new param in the Siteconfig definition, that would end-up filling the proper param in the Infraenv CR from the Host Inventory.


##Debugging Installation phase


## Enable a root console using the Host Inventory resources
[ToDo]

## Enable a root console using ZTP GitOps resources

[ToDo]