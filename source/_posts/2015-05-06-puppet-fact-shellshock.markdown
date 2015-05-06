---
layout: post
title: "Puppet Fact: shellshock"
date: 2015-05-06 11:40
comments: true
categories: puppet facter CVE_2014_6271 shellshock
---
{% img center https://puppetlabs.com/wp-content/uploads/2010/12/PL_logo_vertical_RGB_sm.png %}

~~~ ruby 
# Fact: shellshock
# Purpose: determine the bash program is vulnerable to CVE_2014_6271
#
# Resolution (FreeBSD Linux SunOS:
#    Output of:
#      "foo='() { echo not patched; }' bash -c foo 2>&1"
#   
# Caveats:
#   None
# 
# Notes:
#   None
#
Facter.add(:shellshock) do
confine :kernel => %w{FreeBSD Linux SunOS}  
  setcode do
    unless system("NOT_VULNERABLE='() { echo VULNERABLE; }' bash -c NOT_VULNERABLE 2>&1")
      shellshock = "NOT_VULNERABLE"
    else
      shellshock = "VULNERABLE"
    end
  end
end 
~~~    