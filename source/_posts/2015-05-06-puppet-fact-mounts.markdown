---
layout: post
title: "Puppet Fact: mounts"
date: 2015-05-06 11:40
comments: true
categories: puppet facter
---
{% img center https://pbs.twimg.com/profile_images/3672925108/954f7381089ac290b4690c5ffd9dd7d3_400x400.png %}

~~~ ruby 
# Fact: mounts
#
# Purpose:
#  Return mount point avail/size/used/estimated total size
#
# Resolution:
#  
#  Iterate through mount points on the system, and
#  extract statistics:
#    FreeBSD:
#      mount -t xfs,ufs,zfs
#    Linux: 
#      mount -t ext2,ext3,ext4,reiserfs,xfs
#   
#
# Caveats:
#  mount_totalsize_gb is an estimate based on the df output of each mount in bytes
#  solaris is unsupported
#
# Notes:
#   None
#

if (Facter.value(:kernel)) =~ /^FreeBSD$|^Linux$/

  mounts = []
  case Facter.value(:kernel)  
    when /^FreeBSD$/
      mntpoints = `mount -t xfs,ufs,zfs`
    when /^Linux$/
      mntpoints = `mount -t ext2,ext3,ext4,reiserfs,xfs`
    #when /^Solaris$/
    #  mntpoints = `mount -t ufs,zfs | grep -v tmp | grep -v swap`
  end

  Facter.add("mounts") do
    confine :kernel => %w{FreeBSD Linux}  
  
    mntpoints.split(/\n/).each {|m|
      mount = m.split(/ /)[2]
      mounts << mount
    }
  
    setcode do
      mounts.join(',')
    end
  
  end


  mounts.each {|mount|
    output = `df #{mount}`
  
    output.lines.to_a.each {|str|
      dsk_size  = nil
      dsk_used  = nil
      dsk_avail = nil

      if str =~ /^\S+\s+([0-9]+)\s+([0-9]+)\s+([0-9]+)\s+/
        dsk_size  = $1
        dsk_used  = $2
        dsk_avail = $3
  
        Facter.add("mount_#{mount}_size") do
          setcode do
            dsk_size
          end
        end
  
        Facter.add("mount_#{mount}_used") do
          setcode do
            dsk_used
          end
        end
  
        Facter.add("mount_#{mount}_avail") do
          setcode do
            dsk_avail
          end
        end
  
        Facter.add("mount_totalsize_gb") do        
          setcode do
          @size = []
          mp = Facter.value(:mounts).split(/\,/)
          mp.each {|m|
           @size << `facter "mount_#{m}_size"`.strip
          }
          mount_totalsize_gb = (@size.map(&:to_i).reduce(:+)).to_i / 1048576
          end
        end
  
      end
  
    }
  }
end
~~~    