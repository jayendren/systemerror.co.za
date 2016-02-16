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

* Reboot

``` 
reboot
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
ExecStart=/usr/bin/DCE
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

*  /usr/src/cluster/cloud-init-odroid.conf

``` 
#cloud-config

coreos:
  etcd:
    name: $host
    data_dir: /var/lib/etcd/
    discovery: https://discovery.etcd.io/$TOKEN # TOKEN=`curl https://discovery.etcd.io/new` 
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  fleet:
    public-ip: $private_ipv4   # used for fleetctl ssh command
 
``` 

* /etc/fleet.conf

``` 
verbosity = 2
etcd_servers=["http://$private_ipv4:4001"]
etcd_request_timeout=30.0
public_ip="$private_ipv4"
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

docker info
Containers: 9
 Running: 0
 Paused: 0
 Stopped: 9
Images: 12
Server Version: 1.10.1
Storage Driver: devicemapper
 Pool Name: docker-179:2-137390-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 3.885 GB
 Data Space Total: 107.4 GB
 Data Space Available: 1.777 GB
 Metadata Space Used: 3.645 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 1.777 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: false
 Deferred Deletion Enabled: false
 Deferred Deleted Device Count: 0
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 WARNING: Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.115 (2016-01-25)
Execution Driver: native-0.2
Logging Driver: json-file
Plugins:
 Volume: local
 Network: bridge null host
Kernel Version: 4.1.17-3-ARCH
Operating System: Arch Linux ARM
OSType: linux
Architecture: armv6l
CPUs: 1
Total Memory: 226.4 MiB
Name: soul-edge.systemerror.co.za
ID: 2BN2:KADT:ELMP:7WH7:HWFK:PMYL:W4O6:BYFG:UCRT:6LPQ:43EL:SRM7

```

* docker install by hand [RHEL clones]


``` 
echo "[Step] Install Docker"
mkdir /usr/src/docker-install
cd /usr/src/docker-install
yum install -y rpm-build glibc-static
rpmbuild --rebuild http://copr-be.cloud.fedoraproject.org/results/gipawu/kernel-aufs/fedora-21-x86_64/aufs-util-3.9-1.fc20/aufs-util-3.9-1.fc21.src.rpm
rpm -i /usr/src/docker-install/rpmbuild/RPMS/armv7hl/aufs-util-3.9-1.fc21.armv7hl.rpm
yum install -y bridge-utils device-mapper device-mapper-libs libsqlite3x docker-registry docker-storage-setup
mkdir /usr/src/docker
cd /usr/src/docker
wget https://kojipkgs.fedoraproject.org//packages/docker-io/1.5.0/18.git92e632c.fc23/src/docker-io-1.5.0-18.git92e632c.fc23.src.rpm
rpm2cpio docker-io-1.5.0-18.git92e632c.fc23.src.rpm | cpio -idmv
tar -xzf docker-92e632c.tar.gz
curl -L https://github.com/umiddelb/armhf/raw/master/bin/docker-1.5.0> docker
### INSTALL
install -p -m 755 docker / usr / bin / docker
# install bash completion
install -p -m 644 docker-92e632c84e7b1abc1a2c5cb3a22e0725951a82af/contrib/completion/bash/docker /usr/share/bash-completion/completions
# install container logrotate cron script
install -p -m 755 docker-logrotate.sh /etc/cron.daily/docker-logrotate
# install vim syntax highlighting
install -p -m 644 docker-92e632c84e7b1abc1a2c5cb3a22e0725951a82af/contrib/syntax/vim/doc/dockerfile.txt /usr/share/vim/vimfiles/doc
install -p -m 644 docker-92e632c84e7b1abc1a2c5cb3a22e0725951a82af/contrib/syntax/vim/ftdetect/dockerfile.vim /usr/share/vim/vimfiles/ftdetect
install -p -m 644 docker-92e632c84e7b1abc1a2c5cb3a22e0725951a82af/contrib/syntax/vim/syntax/dockerfile.vim /usr/share/vim/vimfiles/syntax
# install udev rules
install -p docker-92e632c84e7b1abc1a2c5cb3a22e0725951a82af/contrib/udev/80-docker.rules /etc/udev/rules.d
# install systemd / init scripts
install -p -m 644 docker.service /usr/lib/systemd/system
# for additional args
install -p -m 644 docker.sysconfig / etc / sysconfig / docker
install -p -m 644-docker network.sysconfig / etc / sysconfig / network-docker
install -p -m 644-docker storage.sysconfig / etc / sysconfig / docker-storage
# install docker config directory
sudo install -dp /etc/docker
getent passwd dockerroot > /dev/null || sudo /usr/sbin/useradd -r -d /var/lib/docker -s /sbin/nologin -c "Docker User" dockerroot
systemctl enable docker.service
systemctl enable docker.socket
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
docker build -t armv6/nginx .
```

* Run container

``` 
docker run -d \
-p 80:80 \
-p 443:443 \
armv6/nginx

docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                      NAMES
0504c017f2bc        armv6/nginx         "nginx"             10 seconds ago      Up 3 seconds        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   pedantic_kowalevski

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
ExecStartPre=/bin/docker pull armv6/nginx
ExecStart=/bin/docker run --name nginx \
-p 80:80 \
-p 443:443 \
armv6/nginx
ExecStop=/bin/docker stop nginx
```  

* Now using fleet we may execute container:

``` 
fleetctl --endpoint=http://192.168.4.100:4001 submit nginx
fleetctl --endpoint=http://192.168.4.100:4001 start nginx
``` 
It works! Now you have fully managed cluster!

# Demo

{% youtube HNNLyqlJbA4 800 600 %}