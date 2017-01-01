Escribir logs HAProxy
=======================

En el archivo /etc/haproxy/haproxy.conf::

	# vi /etc/haproxy/haproxy.conf
	log         127.0.0.1 local2

::

	# service haproxy reload

En /etc/rsyslog.conf::

	# vi /etc/rsyslog.conf
	# Provides UDP syslog reception
	$ModLoad imudp
	$UDPServerRun 514

	# Provides TCP syslog reception
	$ModLoad imtcp
	$InputTCPServerRun 514

	local2.*                       /var/log/haproxy.log

::
 
	# service rsyslog restart

	Verificamos el puerto y la creacion del archivo de log

	# netstat -nat | grep 514
	tcp        0      0 0.0.0.0:514                 0.0.0.0:*                   LISTEN      
	tcp        0      0 :::514                      :::*                        LISTEN 

	# ls -l /var/log/haproxy.log 
	-rw------- 1 root root 2609 dic 31 20:40 /var/log/haproxy.log

::

	# tail /var/log/haproxy.log
	Dec 31 20:31:20 localhost haproxy[1559]: 192.168.1.3:57977 [31/Dec/2016:20:31:20.100] main app/app1 0/0/0/-1/1 502 748 - - PH-- 0/0/0/0/0 0/0 "GET /favicon.ico HTTP/1.1"
	Dec 31 20:32:40 localhost haproxy[1580]: Proxy main started.
	Dec 31 20:32:40 localhost haproxy[1580]: Proxy app started.
	Dec 31 20:32:44 localhost haproxy[1581]: 192.168.1.3:57979 [31/Dec/2016:20:32:44.845] main app/app1 0/0/1/0/1 301 533 - - ---- 1/1/0/1/0 0/0 "GET / HTTP/1.1"
	Dec 31 20:33:38 localhost haproxy[1581]: Server app/app1 is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
	Dec 31 20:33:38 localhost haproxy[1581]: backend app has no server available!
	Dec 31 20:34:08 localhost haproxy[1581]: Server app/app1 is UP, reason: Layer4 check passed, check duration: 0ms. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
	Dec 31 20:35:15 localhost haproxy[1581]: 192.168.1.3:57990 [31/Dec/2016:20:35:15.922] main app/app1 0/0/0/1/1 301 533 - - ---- 1/1/0/1/0 0/0 "GET / HTTP/1.1"
	Dec 31 20:39:55 localhost haproxy[1581]: 192.168.1.3:58006 [31/Dec/2016:20:39:55.539] main app/app1 0/0/0/0/0 301 533 - - ---- 1/1/0/1/0 0/0 "GET / HTTP/1.1"
	Dec 31 20:40:18 localhost haproxy[1581]: 192.168.1.3:58010 [31/Dec/2016:20:40:18.652] main app/app1 0/0/0/0/0 301 545 - - ---- 1/1/0/1/0 0/0 "GET /asdfas HTTP/1.1"

