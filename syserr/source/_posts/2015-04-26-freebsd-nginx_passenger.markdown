---
layout: post
title: "FreeBSD nginx+passenger"
date: 2015-04-27 18:45
comments: true
categories: freebsd puppet passenger nginx
---
{% img center https://forums.freebsd.org/styles/freebsd/xenforo/freebsd_logo.png %}

{% blockquote passenger installation, https://gist.githubusercontent.com/geoffgarside/855302/raw/cc24215b0a2601911b84f48092627b9976fc3332/install-passenger-nginx.sh %}
{% endblockquote %}
```
#!/bin/sh
#
# Pre-requisites:
#   FreeBSD
#   Passenger installed as a gem
# Optional:
#   Ruby Enterprise Edition instead of lang/ruby18
#

make -C /usr/ports/www/nginx clean patch apply-slist install-rc-script
source_dir=`ls -d /usr/ports/www/nginx/work/nginx-*`	# should be only one

passenger-install-nginx-module --prefix=/usr/local \
	--nginx-source-dir="${source_dir}" \
	--extra-configure-flags="--with-cc-opt=\"-I /usr/local/include\" \
				--with-ld-opt=\"-L /usr/local/lib\" \
				--conf-path=/usr/local/etc/nginx/nginx.conf \
				--sbin-path=/usr/local/sbin/nginx \
				--pid-path=/var/run/nginx.pid \
				--error-log-path=/var/log/nginx-error.log \
				--user=www --group=www \
				--http-client-body-temp-path=/var/tmp/nginx/client_body_temp \
				--http-proxy-temp-path=/var/tmp/nginx/proxy_temp \
				--http-fastcgi-temp-path=/var/tmp/nginx/fastcgi_temp \
				--http-log-path=/var/log/nginx-access.log \
				--with-http_addition_module \
				--with-http_gzip_static_module \
				--with-http_ssl_module \
				--with-http_stub_status_module \
				--with-ipv6"
```
