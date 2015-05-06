---
layout: post
title: "Puppet Fact: firewall_type"
date: 2015-05-06 11:35
comments: true
categories: puppet facter
---
{% img center https://pbs.twimg.com/profile_images/3672925108/954f7381089ac290b4690c5ffd9dd7d3_400x400.png %}

~~~ ruby 
# Fact: firewall_type
#
# Purpose: determine the firewall type on the host
#
# Resolution:
#   FreeBSD: check rc.conf
#   Solaris: check state using /usr/sbin/ipf
#   Linux: check for presence of iptable_filter in kernel
#
# Caveats:
#   none
# 
# Notes:
#   None
Facter.add(:firewall_type) do
  if Facter.value(:operatingsystem).match("FreeBSD")  
      setcode do
        result = %x{grep -E 'pf_enable="YES"|firewall_enable="YES"' /etc/rc.conf|grep -v '#'}.chomp
        if result.match("pf_enable")
            firewall_type = "FreeBSD_PF"
            elsif result.match("firewall_enable")
                firewall_type = "FreeBSD_IPFW"
            elsif result.match("")
                firewall_type = "UNKNOWN"
        end
      firewall_type
      end     
  elsif Facter.value(:operatingsystem).match("Solaris")     
      setcode do
        result = %x{/usr/sbin/ipf -V|head -3|tail -1}.chomp
        if result.match("Running: yes")
            firewall_type = "Solaris_IPFilter"      
            elsif result.match("")
                firewall_type = "UNKNOWN"
        end
      firewall_type        
      end  
  elsif Facter.value(:operatingsystem).match(/RedHat|Ubuntu|CentOS/)
      setcode do
        result = %x{/sbin/lsmod |grep iptable_filter|head -1}.chomp
        if result.match("iptable_filter")
            firewall_type = "Linux_IPTables"      
            elsif result.match("")
                firewall_type = "UNKNOWN"
        end
      firewall_type        
      end  
  end    
end
~~~    