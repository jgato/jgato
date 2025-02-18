# NMState configurations

NMState is a declarative way of configuring Host Network Management: https://nmstate.io/

Dont get confused with the NMState Operator, that is: *a Kubernetes API for performing state-driven network configuration across the OpenShift Container Platform cluster’s nodes with NMState*. https://github.com/nmstate/kubernetes-nmstate

In this document I will note different configurations and findings about it. We use this API on different Openshift installtaion methods, like ABI and ZTP.

## IPv6 and autoconf and dhcp

This topic got me confused for some time. Specially, while working with ABI (Agent Based Installer). Basically, the scenario was to deploy a compact cluster, with the following configuration:
 * Dual-stack ipv4/ipv6
 * IPv6 configured by DHCP, but routes and gateway configured statically.
 * There is a DHCPv6 and RA with a M (Managed) flag, that tells you to configure IP Address with DHCP.
 
To get this we have to main params `autoconf` and `dhcp`. According to the documentation:

```
The autoconf property is the boolean switch for IPv6 autoconf via Router Advertisement (RA).
```

and `dhcp` that indicates if you get IP address from DHCP. 

But how to combine both, considering we are on an IPv6 environment? 

 * `autoconf: true` and `dhcp: true`: This combination is tricky. `autoconf: true` means that you want RA to configure you IP address (slaac protocol). And then, if the RA sends the M Flag, it will also take an IP address from dhcp. You will have two IP addresses. Therefore, this has go work with the configuration of the RA:
 

 * `autoconf: false` and `dhcp: true`: slaac protocol is not used to configure your IP Address. Anyway, the RA is received with an M Flag, forwarding to DHCP the acquisition of an IP address. You only have the IP address from DHCP.
 * `autoconf: false` and `dhcp: false`: you are in charge of configuring IPs statically.
 * `autoconf: true` and `dhcp: false`: this would appears mean: slaac address but not dhcp. But, what if the RA is configued with the flag M, and you are disabling dhcp? In this moment we cannot know it. So, this combination is not supported. If you want only slaac, you have to configure the RA with the flag A.