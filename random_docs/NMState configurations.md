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

Both parameters, at the end, will configure the NetworkManager param for `ipv6.method`. According to the following combinations:
 * `autoconf: true` no matter the vaule of `dhcp`, then `ipv6.method: auto`.
 * `autoconf: true` and `dhcp: false`: **not supported**. 
 * `autoconf: false` and `dhcp: true`, the `ipv6.method: dhcp`.
 * `autoconf: false` and `dhcp: false`, the `ipv6.method: manual`. This includes to ignore all the other configurations about auto-dns, auto-routes, auto-*. 

So having in mind the NetworkManager will mainly work with the ìpv6.method`: 

 * `ipv6.method: manual` everything configured statically. Including routes, dns and gatway.
 * `ipv6.method: dhcp`, autoconf is disabled. Anyway, because of IPv6, the RA is received expecting to have the DHCPv6 address. SLAAC address will never be generated. 
 * `ipv6.method: auto`, autoconf is enabled. The RA is received and NetworkManager will act with regards of the following flags (that can be combined):
   * If it contains an A Flag, it will configure the SLAAC address.
   * If it contains an M Flag, it will invoque DHCPv6 is started in solicit mode to request an address.
   * If it contains an O Flag, DHCPv6 is started in info-request mode.
 
 
What if you want to configure your IP address only with SLAAC? Configure `autoconf:true` and configure your RA with flag M disabled and flag A enabled.

What if you want to configure your IP address only with DHCP? Configure `autoconf: false` and configure your RA (with IPv6 RA is always watched) and set the flag M enabled.

What if you want to configure your IP address with both SLAAC and DHCP?
Configure `autoconf:true` and configure your RA with flag M and A enabled.
