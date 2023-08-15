HAProxy DOS and DDOS attacks
==========================================================

Queremos bloquear los posibles ataques de DOS and DDOS attacks.

TCP syn flood attacks
--------------------------
The syn flood attacks consist in sending as many TCP syn packets as possible to a single server
trying to saturate it or at least, saturating its uplink bandwith::

	# Protection SYN flood
	net.ipv4.tcp_syncookies = 1
	net.ipv4.conf.all.rp_filter = 1
	net.ipv4.tcp_max_syn_backlog = 1024
	
Slowloris like attacks
---------------------------

For this kind of attack, the clients will send very slowly their requests to a server: header by
header, or even worst character by character, waiting long time between each of them.
The server have to wait until the end of the request to process it and send back its response.
The purpose of this attack is to prevent regular users to use the service, since the attacker would be
using all the available resources with very slow queries.
In order to protect your website against this kind of attack, just setup the HAProxy option “timeout
httprequest”.
You can set it up to 5s, which is long enough.
It tells HAProxy to let five (5) seconds to a client::

	global
		stats socket ./haproxy.stats level admin
	defaults
		[...]
		timeout http-request 5s
		[...]

To test this configuration, simply open a telnet to the frontend port and wait for 5 seconds::

	telnet 127.0.0.1 8080
	Trying 127.0.0.1...
	Connected to 127.0.0.1.
	Escape character is '^]'.
	HTTP/1.0 408 Request Time-out
	Cache-Control: no-cache
	Connection: close
	Content-Type: text/html
	<h1>408 Request Time-out</h1>
	Your browser didn't send a complete request in time.
	Connection closed by foreign host.

Have a WhiteList
-------------------

We can have a WhiteList::

	# Allow clean known IPs to bypass the filter
	tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
	
Limiting the number of connections per users
---------------------------------------

As seen before, a browser opens up 5 to 7 TCP connections to a website when it wants to download
objetcs and they are opened quite quickly.

	# Shut the new connection as long as the client has already 10 opened
	tcp-request connection reject if { src_conn_cur ge 10 }
	tcp-request connection track-sc1 src

Testing the configuration. run an apache bench to open 10 connections and doing request on these connections::

	ab -n 50000000 -c 10 http://127.0.0.1:8080/
	
Watch the table content on the haproxy stats socket::

	echo "show table ft_web" | socat unix:./haproxy.stats - 
	# table: ft_web, type: ip, size:102400, used:1
	0x7afa34: key=127.0.0.1 use=10 exp=29994 conn_cur=10

Let’s try to open an eleventh connection using telnet::

	telnet 127.0.0.1 8080
	Trying 127.0.0.1...
	Connected to 127.0.0.1.
	Escape character is '^]'.
	Connection closed by foreign host.

Basically, opened connections can keep on working, while a new one can’t be established.

Limiting the connection rate per user
---------------------------------------

In the previous chapter, we’ve seen how to protect ourselves from somebody who wants to open
more than X connections at the same time.
Well, this is good, but something which may kill performance would to allow somebody to open and
close a lot of tcp connections over a short period of time.
As we’ve seen previously, a browser will open up to 7 TCP connections in a very short period of time
(a few seconds). One can consider that somebody having more than 20 connections opened over a time. have a negative impact for them. You can whitelist these IPs::


	# Shut the new connection as long as the client has already 10 opened
	tcp-request connection reject if { src_conn_rate ge 10 }
	tcp-request connection track-sc1 src


Testing the configuration. run 10 requests with ApacheBench, everything may be fine::

	ab -n 10 -c 1 -r http://127.0.0.1:8080/	

Using socat we can watch this traffic in the sticktable::

	# table: ft_web, type: ip, size:102400, used:1
	0x11faa3c: key=127.0.0.1 use=0 exp=28395 conn_rate(3000)=10

Running a telnet to run a eleventh request and the connections get closed::

	telnet 127.0.0.1 8080
	Trying 127.0.0.1...
	Connected to 127.0.0.1.
	Escape character is '^]'.
	Connection closed by foreign host.

Limiting the HTTP request rate
------------------------------------

Even if in the previous examples, we were using HTTP as the protocol, we based our protection on
layer 4 information: number or opening rate of TCP connections.
An attacker could respect the number of connection we would set by emulating the behavior of a
regular browser.
Now, let’s go deeper and see what we can do on HTTP protocol.::

	frontend ft_web
	bind 0.0.0.0:8080

		# Use General Purpose Couter (gpc) 0 in SC1 as a global abuse counter
		# Monitors the number of request sent by an IP over a period of 10 seconds
		stick-table type ip size 1m expire 10s store gpc0,http_req_rate(10s)
		tcp-request connection track-sc1 src
		tcp-request connection reject if { src_get_gpc0 gt 0 }

	backend bk_web
		balance roundrobin
		cookie MYSRV insert indirect nocache
		# If the source IP sent 10 or more http request over the defined period,
		# flag the IP as abuser on the frontend
		acl abuse src_http_req_rate(ft_web) ge 10
		acl flag_abuser src_inc_gpc0(ft_web)
		tcp-request content reject if abuse flag_abuser

Testing the configuration. run 10 requests with ApacheBench, everything may be fine::

	ab -n 10 -c 1 -r http://127.0.0.1:8080/


Using socat we can watch this traffic in the sticktable::

	# table: ft_web, type: ip, size:1048576, used:1
	0xbebbb0: key=127.0.0.1 use=0 exp=8169 gpc0=1 http_req_rate(10000)=10


Running a telnet to run a eleventh request and the connections get closed::

	telnet 127.0.0.1 8080
	Trying 127.0.0.1...
	Connected to 127.0.0.1.
	Escape character is '^]'.
	Connection closed by foreign host.

Detecting vulnerability scans
-----------------------------------

Vulnerability scans could generate different kind of errors which can be tracked by Aloha and HAProxy:
invalid and truncated requests
denied or tarpitted requests
failed authentications
4xx error pages

HAProxy is able to monitor an error rate per user then can take decision based on it.::

	frontend ft_web
		bind 0.0.0.0:8080
		# Use General Purpose Couter 0 in SC1 as a global abuse counter
		# Monitors the number of errors generated by an IP over a period of 10 seconds
		stick-table type ip size 1m expire 10s store gpc0,http_err_rate(10s)
		tcp-request connection track-sc1 src
		tcp-request connection reject if { src_get_gpc0 gt 0 }


	backend bk_web
		balance roundrobin
		cookie MYSRV insert indirect nocache
		# If the source IP generated 10 or more http request over the defined period,
		# flag the IP as abuser on the frontend
		acl abuse src_http_err_rate(ft_web) ge 10
		acl flag_abuser src_inc_gpc0(ft_web)
		tcp-request content reject if abuse flag_abuser
		
		

run an apache bench, pointing it on a purposely wrong URL::

	ab -n 10 http://127.0.0.1:8080/dlskfjlkdsjlkfdsj
	
Watch the table content on the haproxy stats socket::

	echo "show table ft_web" | socat unix:./haproxy.stats -
	# table: ft_web, type: ip, size:1048576, used:1
	0x8a9770: key=127.0.0.1 use=0 exp=5866 gpc0=1 http_err_rate(10000)=11
	
Let’s try to run the same ab command and let’s get the error::

	apr_socket_recv: Connection reset by peer (104)
	which means that HAProxy has blocked the IP address



	======================================================
		The File Example
	======================================================
	
[root@srv-haproxy-02 haproxy]# cat haproxy.cfg
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   https://www.haproxy.org/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    5s # Slowloris like attacks
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------

frontend http-in
    bind *:80
	
	#################### SEGURIDAD ###################################
	# Shut the new connection as long as the client has already 10 opened
	tcp-request connection reject if { src_conn_cur ge 10 }
	tcp-request connection track-sc1 src
	
	# Shut the new connection as long as the client has already 10 opened
	tcp-request connection reject if { src_conn_rate ge 10 }
	tcp-request connection track-sc1 src

	# Limiting the HTTP request rate
	# Monitors the number of request sent by an IP over a period of 10 seconds
	stick-table type ip size 1m expire 10s store gpc0,http_req_rate(10s)
	tcp-request connection track-sc1 src
	tcp-request connection reject if { src_get_gpc0 gt 0 }

	# Detecting vulnerability scans
	# Use General Purpose Couter 0 in SC1 as a global abuse counter
	# Monitors the number of errors generated by an IP over a period of 10 seconds
	stick-table type ip size 1m expire 10s store gpc0,http_err_rate(10s)
	tcp-request connection track-sc1 src
	tcp-request connection reject if { src_get_gpc0 gt 0 }
	#################### SEGURIDAD ###################################
	
    acl sub1 hdr_sub(host) -i www.free.com
    acl sub2 hdr_sub(host) -i www.private.com
    acl sub3 hdr_sub(host) -i www.public.com

    use_backend free_backend if sub1
    use_backend private_backend if sub2
    use_backend public_backend if sub3

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------

backend free_backend
    mode http
    option forwardfor
	
	#################### SEGURIDAD ###################################
	# Limiting the HTTP request rate
	# If the source IP sent 10 or more http request over the defined period,
	# flag the IP as abuser on the frontend
	acl abuse src_http_req_rate(ft_web) ge 10
	acl flag_abuser src_inc_gpc0(ft_web)
	tcp-request content reject if abuse flag_abuser

	# Detecting vulnerability scans
	# If the source IP generated 10 or more http request over the defined period,
	# flag the IP as abuser on the frontend
	acl abuse src_http_err_rate(ft_web) ge 10
	acl flag_abuser src_inc_gpc0(ft_web)
	tcp-request content reject if abuse flag_abuser
	#################### SEGURIDAD ###################################
	
    #http-send-name-header Host
    http-request set-header Host www.free.com #if { srv_id 1 }
    server alpha_server 172.24.100.147:80


backend private_backend
    mode http
    option forwardfor
    #http-send-name-header Host
    http-request set-header Host www.private.com #if { srv_id 1 }
    server alpha_server 172.24.100.147:80


backend public_backend
    mode http
    option forwardfor
    #http-send-name-header Host
    http-request set-header Host www.pubblic.com #if { srv_id 1 }
    server alpha_server 172.24.100.147:80
