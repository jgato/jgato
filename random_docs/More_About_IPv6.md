# More about IPv6

Some tips, tricks, and configuraitons

## NMState configuration for IPv6

```yaml
ipv6:
  enabled: true
  autoconf: true
  dhcp: true
  auto-dns: false
  auto-gateway: true
  auto-routes: true
```

* autoconf means, that the IPv6 will be assigned based on received RA. [How SLAAC is calculated](https://www.ciscopress.com/articles/article.asp?p=2154680). So, true means the IP will be assigned by SLAAC.

* dhcp means that the IPv6 will be assigned by a DHCP server

NetworkManager seems to have a bug that avoids to set 'autoconf: true' and 'dhcp: false'. More [here](https://issues.redhat.com/browse/RFE-3671)



## NetworkManager configuration for IPv6



```bash
# nmcli conn show  br-ex | grep ipv6           
ipv6.method:                            auto
ipv6.dns:                               --
ipv6.dns-search:                        --
ipv6.dns-options:                       --
ipv6.dns-priority:                      0
ipv6.addresses:                         --
ipv6.gateway:                           --
ipv6.routes:                            --
ipv6.route-metric:                      -1
ipv6.route-table:                       0 (unspec)
ipv6.routing-rules:                     --
ipv6.ignore-auto-routes:                no
ipv6.ignore-auto-dns:                   no
ipv6.never-default:                     no
ipv6.may-fail:                          yes
ipv6.required-timeout:                  -1 (default)
ipv6.ip6-privacy:                       -1 (unknown)
ipv6.addr-gen-mode:                     stable-privacy
ipv6.ra-timeout:                        0 (default)
ipv6.dhcp-duid:                         --
ipv6.dhcp-iaid:                         --
ipv6.dhcp-timeout:                      0 (default)
ipv6.dhcp-send-hostname:                yes
ipv6.dhcp-hostname:                     --
ipv6.dhcp-hostname-flags:               0x0 (none)
ipv6.token:                             --

```

* ipv6.method
  
  * Auto means to use SLAAC
  
  * DHCP use DHCPv6

In an IPv6 network you will be receiving a RA. This is also important about how the NM is configured. Because it contains information saying if the IP is configured by SLAAC or not. 