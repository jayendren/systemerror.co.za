---
layout: post
title: "Puppet Fact: secureshell"
date: 2015-05-06 11:40
comments: true
categories: puppet facter ssh
---
{% img center https://pbs.twimg.com/profile_images/3672925108/954f7381089ac290b4690c5ffd9dd7d3_400x400.png %}

~~~ ruby 
# Fact: secure_shell
# Purpose: 
#  ssh_version: determine ssh version
#  ssh_supported_protocols: determine support protocols
#
# Resolution:
#   ssh_version:
#    Output of: ssh -V 2>&1
#   ssh_supported_protocols:
#    Output of: ssh -v localhost -o BatchMode=yes -v 2>&1|grep kex_parse_kexinit|grep 'aes'
#     
#
# Caveats:
#   None
# 
# Notes:
#   None
#
Facter.add(:ssh_version) do
  confine :operatingsystem => %w{Solaris}
  setcode do
    test = Facter::Util::Resolution.exec("ssh -V 2>&1")
    if test.nil?
      ssh_version = "NOT_INSTALLED"
    else
      if test.match(/Sun_SSH/)
        result = test.split(/,/)[0].split(/_/)[2]
      else
        result = test.strip.split(" ")[0].split(/_/)[1].split(/p/)[0]
        if not result.empty?
          ssh_version = result
        end
      end
    end
  end
end 

Facter.add(:ssh_version) do
  confine :operatingsystem => %w{RedHat CentOS Ubuntu FreeBSD}
  setcode do
    test = Facter::Util::Resolution.exec("ssh -V 2>&1")
    if test.nil?
      ssh_version = "NOT_INSTALLED"
    else
      result = test.strip.split(" ")[0].split(/_/)[1].split(/p/)[0]
      if not result.empty?
        ssh_version = result
      end
    end
  end
end 

Facter.add(:ssh_supported_protocols) do
  confine :operatingsystem => %w{RedHat CentOS Ubuntu FreeBSD Solaris}
  setcode do
    test = Facter::Util::Resolution.exec("ssh -v localhost -o BatchMode=yes -v 2>&1|grep kex_parse_kexinit|grep 'aes'")
    if test.nil?
      ssh_supported_protocols = "UNKNOWN"
    else
      result = test.lines.first.split(/kex_parse_kexinit:/)[1].strip
      if not result.empty?
        ssh_supported_protocols = result
      end
    end
  end
end 

~~~    