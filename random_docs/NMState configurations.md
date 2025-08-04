# NMState configurations

NMState is a declarative way of configuring Host Network Management: https://nmstate.io/

Dont get confused with the NMState Operator, that is: *a Kubernetes API for performing state-driven network configuration across the OpenShift Container Platform cluster’s nodes with NMState*. https://github.com/nmstate/kubernetes-nmstate

In this document I will note different configurations and findings about it. We use this API on different Openshift installtaion methods, like ABI and ZTP.

## IPv6 and autoconf and dhcp

To understand the different ways of configuring IPv6 addresses with NMState, we need to understand how to use `autoconf` and `dhcp` params. According to the documentation:

```
The autoconf property is the boolean switch for IPv6 autoconf via Router Advertisement (RA).
```

and `dhcp` that indicates if you get IP address from DHCP. Remember DHCPv6 is not like DHCPv4. A similar set of features would be that DHCPv4=(DHCPv6+RA). So, RA is a requirement for IPv6, and actually, the NetworkManager will always rely, first, on it. 

But how to combine both params (`autoconf` and `dhcp`), considering we are on an IPv6 environment? 

Both parameters will configure the NetworkManager param for `ipv6.method`. According to the following combinations:
 * `autoconf: true` no matter the value of `dhcp`, the `ipv6.method: auto`.
 * `autoconf: true` and `dhcp: false`: **not supported**. 
 * `autoconf: false` and `dhcp: true`, the `ipv6.method: dhcp`.
 * `autoconf: false` and `dhcp: false`, the `ipv6.method: manual`. This includes to ignore all the other configurations about auto-dns, auto-routes, auto-*. 

So having in mind the NetworkManager will mainly work with the ìpv6.method`: 

 * `ipv6.method: manual` everything configured statically. Including routes, dns and gatway.
 * `ipv6.method: dhcp`, autoconf is disabled. Anyway, because of IPv6, the RA is received expecting to have M flag enabled. DHCPv6 is started in solicit mode to request an address. SLAAC address will never be generated. In some way, this disable the SLAAC stack on the host. RouteAnnounce (RA), in some way, is ignored.
 * `ipv6.method: auto`, autoconf is enabled. The RA is received and NetworkManager will act with regards of the following flags (that can be combined):
   * If it contains an A Flag, it will configure the SLAAC address.
   * If it contains an M Flag, it will invoque DHCPv6 is started in solicit mode to request an address.
   * If it contains an O Flag, DHCPv6 is started in info-request mode.
 
 
What if you want to configure your IP address only with SLAAC? Configure `autoconf:true` and configure your RA with flag M disabled and flag A enabled.

What if you want to configure your IP address only with DHCP? Configure `autoconf: false` But this case is strange in an IPv6 environment if your router is sending RAs. Better do the next example.

What if you want to configure your IP address only with DHCP but receive the routes from RA? Configure `autoconf: true`  and your RA with flag M enabled and A disabled. 

What if you want to configure your IP address with both SLAAC and DHCP?
Configure `autoconf:true` and configure your RA with flag M and A enabled.

Consider that, ideally, the router rules on an IPv6 environment. So, it is recommended to have `autoconf:true` and the configure SLAAC or DHCP address using the M and A flags. Using `autoconf:false` could be confusing on an IPv6 environment. You disable the SLAAC stack, but the router could be requesting the hosts to be configured with an SLAAC address and some specific routes. You are ignoring this. `autoconf: false` would makes sense when you dont have RA, and you dont have anything to receive.

> Others params at NMState like `auto-route`, `auto-dns`, etc are not covered yet here, and these works with RA and the O flag. 