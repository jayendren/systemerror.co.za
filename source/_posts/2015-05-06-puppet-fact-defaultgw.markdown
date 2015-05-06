---
layout: post
title: "Puppet Fact: defaultgw"
date: 2015-05-06 11:25
comments: true
categories: puppet facter
---
{% img center https://pbs.twimg.com/profile_images/3672925108/954f7381089ac290b4690c5ffd9dd7d3_400x400.png %}

~~~ ruby 
# Fact: defaultgw
# Purpose: determine the hosts default gateway
#
# Resolution:
#   Output of:
#      netstat -nr | grep default   
#
# Caveats:
#   None
# 
# Notes:
#   None
#
Facter.add(:defaultgw) do
confine :kernel => %w{FreeBSD Linux Solaris}  
  setcode do
    test = Facter::Util::Resolution.exec("netstat -nr | grep default 2>&1").split(" ")[1].strip
    if (test.to_s.empty?)
      defaultgw = "UNKNOWN"
    else
      defaultgw = test
    end
  end
end  
~~~    