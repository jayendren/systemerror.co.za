---
layout: post
title: "Building an ARM Cluster"
date: 2016-02-15 22:11
comments: true
categories: docker arm
---
{% img center http://mkaczanowski.com/wp-content/uploads/2015/03/docker.png %}
{% blockquote references: http://mkaczanowski.com/building-arm-cluster-part-3-docker-fleet-etcd-distribute-containers/ %}
{% endblockquote %}


# Building ARM cluster: Docker, fleet, etcd...Distribute containers!

* Preface
* ARCH Linux install 
	* ODROID C1 + [armv7] - TODO
	* RPI2 [armv7] - TODO
	* RPI [armv6]
* systemd files
* Install go, docker, fleet, etcd, cloudinit
* Build Docker images
	* ODROID C1 + [armv7] - TODO
	* RPI2 [armv7] - TODO
	* RPI [armv6]
* Run NGINX container with fleet
* Demo


# Preface

In this post we are going to install ARCH Linux on a armv{6,7} SOC and install golang + docker + fleet + etcd. 

As most of these tools is made for 64bit systems we will have to modify them a bit. 

The set of tools is used in coreos for managing containers in a cluster, so that is a perfect tool for us!

In this tutorial, we will use 2 SoC's:
infa01 - 192.168.2.80/24 and infra02 - 192.168.2.81/24

# ARCH Linux install

## ODROID C1 + [armv7] - TODO

## RPI2 [armv7] - TODO

## RPI [armv6]

*  Download image

``` 
curl -o http://downloads.sourceforge.net/project/archlinuxrpi/ArchLinuxARM-rpi-latest.zip\?r\=https%3A%2F%2Fsourceforge.net%2Fprojects%2Farchlinuxrpi%2F%3Fsource%3Ddirectory\&ts\=1455514931\&use_mirror\=tenet
``` 

* Extract image

``` 
unzip ArchLinuxARM-rpi-latest.zip
``` 

* Transfer image to sd card, replace rdiskX with the correct disk that represents your sd card using mount or df

```
# unmount sdcard
umount /dev/disk3
# transfer image to sdcard
sudo dd if=ArchLinuxARM-rpi-latest.img of=/dev/rdisk3 bs=1m conv=sync
``` 

* Boot into OS and resize partition

``` 
fdisk /dev/mmcblk0
Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/mmcblk0: 4.3 GiB
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x054fffb4

Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        2048   206847   204800  100M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      206848   413696 15316992  4.3G 83 Linux

Command (m for help): d
Partition number (1,2, default 2): 2

Partition 2 has been deleted.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2):
First sector (206847-15523839, default 206847): 206848
Last sector, +sectors or +size{K,M,G,T,P} (206847-15523839, default 15523839):

Created a new partition 2 of type 'Linux' and of size 7.3 GiB.
w
``` 

* Reboot or Partprobe

``` 
reboot # reboot
partprob /dev/mmcblk0 # without reboot
``` 

* Resize

``` 
resize2fs /dev/mmcblk0p2
``` 

* Verify disk size

``` 
fdisk -lu /dev/mmcblk0
Disk /dev/mmcblk0: 7.4 GiB, 7948206080 bytes, 15523840 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x054fffb4

Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        2048   206847   204800  100M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      206848 15523839 15316992  7.3G 83 Linux
```

# systemd files


* /lib/systemd/system/fleetd.service

```
[Unit]
Description=fleetd
After=etcd.service

[Service]
ExecStart=/usr/bin/fleetd
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

* /lib/systemd/system/etcd.service

```
[Unit]
Description=etcd
Wants=network.target network-online.target
After=network.target network-online.target cloudinit.service

[Service]
ExecStart=/usr/bin/etcd
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
``` 

* /lib/systemd/system/cloudinit.service

``` 
[Unit]
Description=cloudinit

[Service]
Type = oneshot
ExecStart=/usr/bin/coreos-cloudinit --from-file /usr/src/cluster/cloud-init-odroid.conf

[Install]
WantedBy=multi-user.target
```  

*  /usr/src/cluster/cloud-init-odroid.conf [node01]

``` 
#cloud-config

coreos:
  etcd:
    data_dir: /var/lib/etcd
    name: infra0
    initial-advertise-peer-urls: http://192.168.2.80:2380
    listen-peer-urls: http://192.168.2.80:2380
    listen-client-urls: http://192.168.2.80:2379
    advertise-client-urls: http://192.168.2.80:2379
    initial-cluster-token: etcd-cluster-arm
    initial-cluster: infra0=http://192.168.2.80:2380,infra1=http://192.168.2.81:2380
    initial-cluster-state: new
  fleet:
    public-ip: 192.168.2.80   # used for fleetctl ssh command
    metadata: node=soul-reaper.systemerror.co.za,env=lab,region=za/rb,cpu=4,arch=armv7,mem=1024MB
 
``` 

*  /usr/src/cluster/cloud-init-odroid.conf [node02]

``` 
#cloud-config

coreos:
  etcd:
    data_dir: /var/lib/etcd
    name: infra1
    initial-advertise-peer-urls: http://192.168.2.81:2380
    listen-peer-urls: http://192.168.2.81:2380
    listen-client-urls: http://192.168.2.81:2379
    advertise-client-urls: http://192.168.2.81:2379
    initial-cluster-token: etcd-cluster-arm
    initial-cluster: infra0=http://192.168.2.80:2380,infra1=http://192.168.2.81:2380
    initial-cluster-state: new

  fleet:
    public-ip: 192.168.2.81   # used for fleetctl ssh command
    metadata: node=soul-edge.systemerror.co.za,env=lab,region=za/rb,cpu=1,arch=armv6,mem=256MB
``` 

* /etc/fleet.conf [node01]

``` 
verbosity = 0
etcd_servers=["http://192.168.2.80:2379, http://192.168.2.81:2379"]
etcd_request_timeout=30.0
public_ip="192.168.2.80"
``` 

* /etc/fleet.conf [node02]

``` 
verbosity = 0
etcd_servers=["http://192.168.2.81:2379, http://192.168.2.80:2379"]
etcd_request_timeout=30.0
public_ip="192.168.2.81"
``` 

* System update and install tools

``` 
pacman -Syu && reboot
pacman -Sy tmux zsh git tree vnstat mlocate axel 7z patch gcc && updatedb
```  

# Install go, docker, fleet, etcd, cloudinit

* go

``` 
echo "[Step] Install go"
cd /usr/src/
git clone https://go.googlesource.com/go
cd go
git checkout go1.4.1
cd src
./make.bash
echo "export PATH=$PATH:/usr/src/go/bin/" >> ~/.bashrc
echo "export GOPATH=/usr/src/spouse" >> ~ / .bashrc
export PATH=$PATH:/usr/src/go/bin/
export GOPATH=/usr/src/spouse
mkdir /usr/src/spouse
``` 

* docker

```
pacman -Sy docker
# add to /boot/cmdline.txt - cgroup_enable=memory swapaccount=1

root=/dev/mmcblk0p2 rw rootwait console=ttyAMA0,115200 console=tty1 selinux=0 plymouth.enable=0 smsc95xx.turbo_mode=N dwc_otg.lpm_enable=0 kgdboc=ttyAMA0,115200 elevator=noop cgroup_enable=memory swapaccount=1
reboot

systemctl enable docker.service
systemctl enable docker.socket

docker version
Client:
 Version:      1.10.1
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   9e83765
 Built:        Thu Feb 11 22:25:21 2016
 OS/Arch:      linux/arm

Server:
 Version:      1.10.1
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   9e83765
 Built:        Thu Feb 11 22:25:21 2016
 OS/Arch:      linux/arm
```

* fleet

``` 
echo "[Step] Install fleet"
cd /usr/src/
go get golang.org/x/tools/cmd/cover
git clone https://github.com/coreos/fleet.git
cd fleet
./build
./test
ln -s /usr/src/fleet/bin/* /usr/bin/
systemctl enable fleet.service
source ~ /.bashrc
```  

* etcd with patches

``` 
echo "[Step] Install etcd"
ETCDI_VERSION=2.0.4
curl -sSL -k https://github.com/coreos/etcd/archive/v$ETCDI_VERSION.tar.gz | tar --touch -v -C /usr/src -xz
cd /usr/src/etcd-$ETCDI_VERSION
curl https://raw.githubusercontent.com/mkaczanowski/docker-archlinux-arm/master/archlinux-etcd/patches/raft.go.patch > raft.go.patch
curl https://raw.githubusercontent.com/mkaczanowski/docker-archlinux-arm/master/archlinux-etcd/patches/server.go.patch > server.go.patch
curl https://raw.githubusercontent.com/mkaczanowski/docker-archlinux-arm/master/archlinux-etcd/patches/watcher_hub.go.patch > watcher_hub.go.patch
patch etcdserver/raft.go < raft.go.patch
patch etcdserver/server.go < server.go.patch
patch store/watcher_hub.go < watcher_hub.go.patch
./build
ln -s /usr/src/etcd-$ETCDI_VERSION/bin/* /usr/bin/
mkdir /var/lib/etcd
systemctl enable cloudinit
systemctl enable etcd
systemctl enable fleetd
```  

*  cloudinit and replace variables $2 = hostname, $3 = host ipaddr

``` 
echo "[Step] Cloud config"
cd /usr/src/
git clone https://github.com/coreos/coreos-cloudinit.git
cd coreos-cloudinit && git checkout 0a46b32c889732874a6cec918629f9bbd88a9c54
./build
ln -s /usr/src/coreos-cloudinit/bin/coreos-cloudinit /usr/bin/coreos-cloudinit
echo -n "hostname> "
read host_name; echo
echo -n "ipaddress> "
read ipaddress; echo
sed -i "s/\$private_ipv4/$ipaddress/g" /usr/src/cluster/cloud-init-odroid.conf
sed -i "s/\$private_ipv4/$ipaddress/g" /usr/src/cluster/fleet.conf
sed -i "s/\$host/$host_name/g" /usr/src/cluster/cloud-init-odroid.conf
```  

# Build Docker images

### ODROID C1 + [armv7] - TODO

### RPI2 [armv7] - TODO

### RPI [armv6]

* download armv6 raspbian_wheezy image

```
axel -a http://files2.linuxsystems.it/raspbian_wheezy_20140726.img.7z
```

* extract armv6 raspbian_wheezy image

```
7z x raspbian_wheezy_20140726.img.7z
```

* obtain armv6 raspbian_wheezy image partition info

```
fdisk -lu raspbian_wheezy_20140726.img 
```

* partition mounting is offset = sectors x 512

```
mount -t auto -o loop,offset=xxxx /path/to/image.img /mnt
```

* import armv6 raspbian_wheezy image into docker

```
tar -C /mnt -c . | docker import - armv6/raspbian_wheezy
```

# How to build nginx container?

* Dockerfile

``` 
FROM armv6/raspbian_wheezy

MAINTAINER jayendren <jayendren@gmail.com>

# ENVARS
ENV TIME_ZONE Africa/Johannesburg

# INSTALL
RUN \
  #add-apt-repository -y ppa:nginx/stable && \
  apt-get update && \
  apt-get install -y nginx && \
  rm -rf /var/lib/apt/lists/* && \
  echo "\ndaemon off;" >> /etc/nginx/nginx.conf && \
  chown -R www-data:www-data /var/lib/nginx

# POST INSTALL
RUN cp /usr/share/zoneinfo/${TIME_ZONE} /etc/localtime

# CLEAN UP
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# PORTS
EXPOSE 80
EXPOSE 443

# VOLUMES
VOLUME ["/etc/nginx/sites-enabled", "/etc/nginx/certs", "/etc/nginx/conf.d", "/var/log/nginx", "/var/www/html"]
WORKDIR /etc/nginx

# CMD
ENTRYPOINT ["nginx"]
```  

* Build container

```
docker build -t nginx .
```

* Run container

``` 
docker run -d \
-p 80:80 \
-p 443:443 \
armv6/nginx

docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                      NAMES
0504c017f2bc        nginx         "nginx"             10 seconds ago      Up 3 seconds        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   pedantic_kowalevski

```  

# Run nginx container with fleet

* Eventually we are able to run nginx container on all of machines at once using fleet!
* Nginx service:

``` 
[Unit]
Description=nginx
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/bin/docker kill nginx
ExecStartPre=-/bin/docker rm nginx
ExecStartPre=/bin/docker pull nginx
ExecStart=/bin/docker run --name nginx \
-p 80:80 \
-p 443:443 \
armv6/nginx
ExecStop=/bin/docker stop nginx
```  

* Now using fleet we may execute container:

``` 
fleetctl --endpoint=http://192.168.2.80:2379 submit nginx
fleetctl --endpoint=http://192.168.2.80:2379 start nginx
``` 
It works! Now you have fully managed cluster!

# Demo

{% youtube HNNLyqlJbA4 800 600 %}