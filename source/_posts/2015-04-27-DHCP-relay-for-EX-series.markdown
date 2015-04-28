---
layout: post
title: "How to configure and verify the DHCP relay for EX-series switches"
date: 2015-04-28 23:45
comments: true
categories: junos
---
{% img center http://kb.juniper.net/shared/img/header/logo-top-m.gif %}
{% blockquote sample config for dhcp-relay on juniper EX switchs, http://kb.juniper.net/InfoCenter/index?page=content&id=KB11020 %}
{% endblockquote %}
SUMMARY:
This article provides information on how to configure and verify the DHCP relay for the EX-series switches.

PROBLEM OR GOAL:
How to configure and verify the DHCP relay for the EX-series switches.
CAUSE:

SOLUTION:
Topology:
```
[Client PC] --- ge-0/0/0 [EX Switch] ge0/0/1 --- [DHCP Server]
```
Client PC is in VLAN 10.

The DHCP server is in VLAN 20 with the 20.20.20.2 IP address.

The EX switch is configured as DHCP relay and performs inter VLAN routing between VLANs 10 and 20.

Configuration:
```
set vlans vlan10 vlan-id 10
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members vlan10
set vlans vlan10 l3-interface vlan.10
set interfaces vlan unit 10 family inet address 10.10.10.1/24

set vlans vlan20 vlan-id 20
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members vlan20
set interfaces vlan unit 20 family inet address 20.20.20.1/24
set vlans vlan20 l3-interface vlan.20
set forwarding-options helpers bootp server 20.20.20.2
set forwarding-options helpers bootp interface vlan.10
```
Note: VLAN.10 Routed VLAN Interface (RVI) is configured for bootp relay, so that bootp packets in VLAN 10 will be routed to the '20.20.20.2' server in VLAN 20. Configure the specific RVIs, to which the bootp packets should be forwarded. A separate server can be configured under the 'bootp interface vlan.X' stanza for only that specific VLAN. A global server IP can also be specified (as above), which is applicable for all RVI interfaces. 

Verifying the DHCP Relay configuration on an EX switch:
```
juniper@EX> show configuration forwarding-options
helpers {
    bootp {
        server 20.20.20.2;
        interface {
            vlan.20;
        }
    }
}
```
Verifying relay agent activity on EX:
```
juniper@EX> show helper statistics 
bootps:
Received packets: 4
Forwarded packets: 4
Dropped packets: 0
Due to no interface in fud database: 0
Due to no matching routing instance: 0
Due to an error during packet read: 0
Due to an error during packet send: 0
Due to invalid server address: 0
Due to no valid local address: 0
Due to no route to server/client: 0
```
Note: The 4 packets shown above are packets that are relayed from the client to the server (Discover, Request) and from the server to the client (Offer, Ack) by the EX switch.

To see DHCP packets entering or exiting an interface, the monitor traffic interface <int-name> command can be used.

To debug relay agent activity on EX :
```
[edit]
set forwarding-options helpers traceoptions file helper
set forwarding-options helpers traceoptions flag bootp
set forwarding-options helpers traceoptions level level
```
After forwarding activity, the /var/log/helper file can be viewed. By default, helper activity is logged in /var/log/fud, which is viewed by using show log fud.

Troubleshooting Tip: When the debugs and show commands that verify bootp packets are being relayed or forwarded, if the IP address is still not obtained, check the path between the client's gateway and the DHCP server. Ensure that routes are in place and no firewall rules are blocking these bootp packets.

Caution:


Debugs using traceoptions should be used with care; especially on a busy router or switch.  It is advisable to turn them on only for specific feature(s) at any given time, and for specific flag and level options of interest and turn them off, when no longer needed. Else, this can cause high CPU activity and affect other processes.

Monitoring live output by 'monitor start <trace-file-name>' can generate excess network traffic, if there is too much data; so avoid it in a busy production environment and use the trace files for viewing later.