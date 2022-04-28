# Creating IPv6 Local Link Addresses

We will make a quick tutorial to create a couple of Namespaces with virtual interfaces that can ping each other through the IPv6 Local Link. All the hacking work/credits to  @kkarampo (thanks)

First we create the two Namespaces: radio and ocp. The 'ocp' one emulates an OCP server (Single Node Openshift) which pings to a Radio Server on a RAN environment.

Create the two NS:

```bash
$> ip netns add radio
$> ip netns add ocp
```

Then, we use a [VETH](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking?source=sso#veth) (Virtual Interface) to communicate both NS. [IPv6 Local Link](https://labs.ripe.net/author/philip_homburg/whats-the-deal-with-ipv6-link-local-addresses/) is accessible only by interfaces in the same network segment. So, it is like if we were directly plugin the two NS. Therefore IPv6 LL should be accessible.

![Pair of VETH devices](https://developers.redhat.com/blog/wp-content/uploads/2018/10/veth.png)

*(fig. from https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking)*

```bash
$> ip link add veth1 netns radio type  veth peer name veth2 netns ocp
```

After that we can access and configure each NS and its interfaces.

```bash
$>ip netns exec radio bash
# inside the radio NS
radio-ns$>ip link set dev veth2 up 
radio-ns$>ip netns exec ocp bash
ocp-ns$>ip link set dev veth2 up
# with the two interfaces up, IPv6 LL are created automtically (as expected)
ocp-ns$>ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth2@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 1e:c3:9c:14:8b:99 brd ff:ff:ff:ff:ff:ff link-netns radio
    inet6 fe80::1cc3:9cff:fe14:8b99/64 scope link 
       valid_lft forever preferred_lft forever
# from ocp we can check ips for radio
ocp-ns$> ip netns exec radio  ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ce:58:f8:77:5c:64 brd ff:ff:ff:ff:ff:ff link-netns ocp
    inet6 fe80::cc58:f8ff:fe77:5c64/64 scope link 
       valid_lft forever preferred_lft forever

```

So now we can play to ping. Simulating the desired environment: OCP will be pinging to Radio Unit:

```bash
$> ip netns exec ocp bash
ocp-ns$> ping6 -I veth2 fe80::cc58:f8ff:fe77:5c64
ping6: Warning: source address might be selected on device other than: veth2
PING fe80::cc58:f8ff:fe77:5c64(fe80::cc58:f8ff:fe77:5c64) from :: veth2: 56 data bytes
64 bytes from fe80::cc58:f8ff:fe77:5c64%veth2: icmp_seq=1 ttl=64 time=0.114 ms
64 bytes from fe80::cc58:f8ff:fe77:5c64%veth2: icmp_seq=2 ttl=64 time=0.087 ms
64 bytes from fe80::cc58:f8ff:fe77:5c64%veth2: icmp_seq=3 ttl=64 time=0.062 ms
64 bytes from fe80::cc58:f8ff:fe77:5c64%veth2: icmp_seq=4 ttl=64 time=0.076 ms
64 bytes from fe80::cc58:f8ff:fe77:5c64%veth2: icmp_seq=5 ttl=64 time=0.105 ms

```

But OCP is "simulating" an OCP SNO. The SNO networking is managed by the ovn-k operator. So after a while of booting the SNO, when the cluster is ready, OVN will reconfigure the networking. Actually, it will take the primary interface (veth2) as a master of a created bridge (br-ex). Lets simulate that, creating that bridge and adding the created veth2 interface. 



```bash
$> ip netns exec ocp bash
ocp-ns$> ip link add name br-ex type bridge
ocp-ns$> ip link set dev br-ex up
ocp-ns$> ip link set dev veth2 master br0

```

When the 'veth2' interface is master on the bridge. The ping will stop working.






