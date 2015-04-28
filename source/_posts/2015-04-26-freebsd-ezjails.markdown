---
layout: post
title: "FreeBSD EZJAILS"
date: 2015-04-26 18:45
comments: true
categories: freebsd ezjails
---
{% img center https://forums.freebsd.org/styles/freebsd/xenforo/freebsd_logo.png %}

{% blockquote FreeBSD Forums, https://forums.freebsd.org/threads/howto-quick-setup-of-jail-on-zfs-using-ezjail-with-pf-nat.30063/ %}
{% endblockquote %}
July 26, 2014 update: different configuration for new pkg tool

Following scenario is presented: 

```
                   /----------- our host --------------\
--{ internet } --- [ 192.0.2.1 ] ---jail--- [ 10.6.6.6 ]

```

where:

em0 is an egress interface (internet facing) 
lo666 is a custom loopback interface (host only)

192.0.2.1 is a public IP address on em0
10.6.6.6 is a jail IP address on lo666

Goal is to create a jail where simple WWW service is running.

Prerequisites:

Installed sysutils/ezjail either from ports or from pre-built repository
ZFS pool where jail dataset will be created; pool zpool is used here

The /etc/rc.conf
Enable PF, ZFS, ezjail and IP forwarding. Create and configure lo666 interface. Lines in question from /etc/rc.conf:

```
cloned_interfaces="lo666"
ifconfig_lo666_alias0="inet 10.6.6.6 netmask 255.255.255.255"

gateway_enable="YES"

pf_enable="YES"
ezjail_enable="YES"
zfs_enable="YES"

```

Bring the interface up. In 9.0-RELEASE it's enough to do: 

```
# ifconfig lo666 create 

```
This creates the interface and assigns the alias from /etc/rc.conf. In case IP address is not up, bring it up manually: 

```
# ifconfig lo666 alias 10.6.6.6 netmask 255.255.255.255 up
```

Enable IP forwarding: 
# sysctl net.inet.ip.forwarding=1

Setup PF
Only the NAT part of the PF is shown here, configuration of PF is not subject of this howto. 
/etc/pf.conf:

```
ext_if="em0"
jail_if="lo666"

IP_PUB="192.0.2.1"
IP_JAIL_WWW="10.6.6.6"

NET_JAIL="10.6.6.0/24"

PORT_WWW="{80,443}"

scrub in all

# nat all jail traffic
nat pass on $ext_if from $NET_JAIL to any -> $IP_PUB

# WWW
rdr pass on $ext_if proto tcp from any to $IP_PUB port $PORT_WWW -> $IP_JAIL_WWW

# demo only, passing all traffic
pass out
pass in

```

Check /etc/pf.conf for any mistakes: 

```
# pfctl -nf /etc/pf.conf
```
If no error is shown, start the firewall. 

```
# /etc/rc.d/pf start

```
Verify the firewall is enabled, check the NAT rules (ALTQ warnings can be safely ignored). 
```

# pfctl -e
pfctl: pf already enabled


# pfctl -sn
nat pass on em0 inet from 10.6.6.0/24 to any -> 192.0.2.1
rdr pass on em0 inet proto tcp from any to 192.0.2.1 port = http -> 10.6.6.6
rdr pass on em0 inet proto tcp from any to 192.0.2.1 port = https -> 10.6.6.6
```


Configure ezjail
By default all jails are stored under /usr/jails directory. However I'll use /local/jails in my setup.

First create a ZFS dataset:

```
# zfs create -o mountpoint=/local/jails zpool/jails
# chmod 700 /local/jails && chown root:wheel /local/jails
```

Main ezjail configuration is stored under /usr/local/etc/ezjail.conf. Uncomment and set at least these parameters:

```
ezjail_jaildir=/local/jails
ezjail_ftphost=ftp.sk.freebsd.org
ezjail_use_zfs="YES"
ezjail_jailzfs="zpool/jails"
```


Use the ftp host closest to you. 
There are several options how to install the base. Here I'll just fetch the base from FTP, see the ezjail-admin(8) for details. 

```
# ezjail-admin install
```

Minimum userland - basejail - has been fetched. You can see it on separate dataset: 

```
# zfs list
zpool                  333M  1.63G    31K  none
zpool/jails            332M  1.63G    47K  /local/jails
zpool/jails/basejail   330M  1.63G   330M  /local/jails/basejail
zpool/jails/newjail   1.70M  1.63G  1.70M  /local/jails/newjail
```


I plan to create more than one jail, I don't want to set all system settings manually for each and every one of them. There's where jail flavor comes in place.
happycamper.local is my domain, I'll use the name happycamper for a flavor.

```
# mkdir -p /local/jails/flavours/happycamper/etc/rc.d
# cd /local/jails/flavours/happycamper/etc
# vi rc.conf
sshd_enable="YES"
syslogd_flags="-ss"

# cp -p /etc/resolv.conf .
# cp -p /local/jails/flavours/example/etc/rc.d/ezjail.flavour.example rc.d/ezjail.flavour.happycamper
```

Flavors are stored under $jailroot/flavours directory ($jailroot == /local/jails). I've created rc.conf and resolv.conf files - these will be copied to new jail with happycamper flavor.

For the demonstration I want to create custom group, user and install screen package. This is done upon first jail startup by ezjail.flavour script. 

In vi editor I have replaced all "example" words by "happycamper". All examples are shown there, easy to understand. In FreeBSD 10 there's a new package management. There are no more pkg_* commands. 

Prior to FreeBSD 10 you can use the following flavor config: 
```
pw group add users
echo -n '$1$p75bbfK.$Kz3dwkoVlgZrfLZdAXQt91' |\
pw user add martin -g users -G wheel -s /bin/csh -d /home/martin -m -H 0

chown -R martin:users /home/martin

pkg_add -r screen
```


If you are running FreeBSD 10 and later: 
```
pw group add users
echo -n '$1$p75bbfK.$Kz3dwkoVlgZrfLZdAXQt91' |\
pw user add martin -g users -G wheel -s /bin/csh -d /home/martin -m -H 0

chown -R martin:users /home/martin
# don't ask - just do
export ASSUME_ALWAYS_YES=YES
pkg bootstrap
pkg install screen
```

Now I'm finally ready to create new jail with a flavor. 

```
# ezjail-admin create -f happycamper -c zfs www 10.6.6.6
ZFS: create the jail filesystem
/local/jails/www/.
/local/jails/www/./etc
/local/jails/www/./etc/rc.d
/local/jails/www/./etc/rc.d/ezjail.flavour.happycamper
/local/jails/www/./etc/rc.conf
/local/jails/www/./etc/resolv.conf
5 blocks


Starting the jail:

# /usr/local/etc/rc.d/ezjail start www
Configuring jails:.
Starting jails: www.
```


It might take a second or two as the flavor script is executed upon first start; it does remove itself afterward. To check the jail status: 

```
# jls
   JID  IP Address      Hostname                      Path
     2  10.6.6.6        www                           /local/jails/www

```

To access the jail ezjail-admin command can be used:

```
# ezjail-admin console www
```

Now the apache can be installed and configured, jail itself is ready.
