---
layout: post
title: "Trolling xferlogs with awk"
date: 2015-05-06 11:11
comments: true
categories: awk
---
{% img center http://ethanschoonover.com/solarized/img/solarized-yinyang.png %}


    #!/usr/bin/env bash
    # find incoming/outgoing/delete files in unix /var/log/xferlog*
    # sample usage:
    # find_xfer pattern [o]outgoing [i]incoming [d]deleted
    # Author: jayendren@gmail.com

    CMD=${0##*/}
    LOGFILE="/var/log/xferlog*"

    if [ $# = 0 ]
    then
      cat <<EOF
    USAGE:
     $CMD pattern direction
     $CMD 201311120101.zip d

    EOF
      exit 1
    fi

    case $2 in
     o)
        bzgrep $1 $LOGFILE|cut -f2-17 -d ':' \
        |awk 'BEGIN {FS=" "} $12="o" {print}'|sort;;

     i)
        bzgrep $1 $LOGFILE|cut -f2-17 -d ':' \
        |awk 'BEGIN {FS=" "} $12="i" {print}'|sort;;

     d)
        bzgrep $1 $LOGFILE|cut -f2-17 -d ':' \
        |awk 'BEGIN {FS=" "} $12="d" {print}'|sort;;

     *)
        echo "$2 direction unsupported,
    Can be one of:
    o
    outgoing
    i
    incoming
    d
    deleted" ;;

    esac
