Balanceador de carga como primera defensa contra ddos
========================================================

Link original
http://blog.haproxy.com/2012/02/27/use-a-load-balancer-as-a-first-row-of-defense-against-ddos/

FEBRUARY 27, 2012 BAPTISTE ASSMANN	41 COMMENTS
We’ve seen recently more and more DOS and DDOS attacks. Some of them were very big, requiring thousands of computers…
But in most cases, this kind of attacks are made by a few computers aiming to make a service or website unavailable, either by sending it too many requests or by taking all its available resources, preventing regular users to use the service.
Some attacks targets known vulnerabilities of widely used applications.

In the present article, we’ll explain how to take advantage of an application delivery controller to protect your website and application against DOS, DDOS and vulnerability scans.

Why using a LB for such protection since a firewall and a Web Application Firewall (aka WAF) could already do the job?
Well, the Firewall is not aware of the application layer but would be useful to pretect against SYN flood attacks. That’s why we saw recently application layer firewalls: Web Application Firewalls, also known as WAF.
Well, since the load balancer is in front of the platform, it can be a good partner for the WAF, filtering out 99% of the attacks, which are managed by script kiddies. The WAF can then happily clean up the remaining attacks.
Well, maybe you don’t need a WAF and you want to take advantage of your Aloha and save some money ;).

Note that you need an application layer load-balancer, like Aloha or OpenSource HAProxy to be efficient.

TCP syn flood attacks
++++++++++++++++++++++

The syn flood attacks consist in sending as many TCP syn packets as possible to a single server trying to saturate it or at least, saturating its uplink bandwith.
If you’re using the Aloha load-balancer, you’re already protected against this kind of attacks: the Aloha includes mechanism to protect you.
The TCP syn flood attack mitigation capacity may vary depending on your Aloha box.

It you’re running your own LB based on HAProxy or HAPEE, you should have a look at the sysctl below (edit /etc/sysctl.conf or play with sysctl command)::

	# Protection SYN flood
	net.ipv4.tcp_syncookies = 1
	net.ipv4.conf.all.rp_filter = 1
	net.ipv4.tcp_max_syn_backlog = 1024 

Note: If the attack is very big and saturates your internet bandwith, the only solution is to ask your internet access provider to null route the attackers IPs on its core network.

Slowloris like attacks
+++++++++++++++++++++++

For this kind of attack, the clients will send very slowly their requests to a server: header by header, or even worst character by character, waiting long time between each of them.
The server have to wait until the end of the request to process it and send back its response.
The purpose of this attack is to prevent regular users to use the service, since the attacker would be using all the available resources with very slow queries.
In order to protect your website against this kind of attack, just setup the HAProxy option “timeout http-request”.
You can set it up to 5s, which is long enough.
It tells HAProxy to let five seconds to a client to send its whole HTTP request, otherwise HAProxy would shut the connection with an error.

For example::

	# On Aloha, the global section is already setup for you
	# and the haproxy stats socket is available at /var/run/haproxy.stats
	global
	  stats socket ./haproxy.stats level admin
	 
	defaults
	  option http-server-close
	  mode http
	  timeout http-request 5s
	  timeout connect 5s
	  timeout server 10s
	  timeout client 30s
	 
	listen stats
	  bind 0.0.0.0:8880
	  stats enable
	  stats hide-version
	  stats uri     /
	  stats realm   HAProxy Statistics
	  stats auth    admin:admin
	 
	frontend ft_web
	  bind 0.0.0.0:8080
	 
	  # Spalreadylit static and dynamic traffic since these requests have different impacts on the servers
	  use_backend bk_web_static if { path_end .jpg .png .gif .css .js }
	 
	  default_backend bk_web
	 
	# Dynamic part of the application
	backend bk_web
	  balance roundrobin
	  cookie MYSRV insert indirect nocache
	  server srv1 192.168.1.2:80 check cookie srv1 maxconn 100
	  server srv2 192.168.1.3:80 check cookie srv2 maxconn 100
	 
	# Static objects
	backend bk_web_static
	  balance roundrobin
	  server srv1 192.168.1.2:80 check maxconn 1000
	  server srv2 192.168.1.3:80 check maxconn 1000
	To test this configuration, simply open a telnet to the frontend port and wait for 5 seconds:

::

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

Unfair users, AKA abusers
++++++++++++++++++++++++++

By unfair users, I mean users (or scripts) which have an abnormal behavior on your website:
* too many connections opened
* new connection rate too high
* http request rate too high
* bandwith usage too high
* client not respecting RFCs (IE for SMTP)

How does a regular browser works?
++++++++++++++++++++++++++++++++++

Before trying to protect your website from weird behavior, we have to define what a “normal” behavior is!
This paragraphs gives the main lines of how a browser works and there may be some differences between browsers.
So, when one wants to browse a website, we use a browser: Chrome, Firefox, Internet Explorer, Opera are the most famous ones.
After typing the website name in the URL bar, the browser will look like for the IP address of your website.
Then it will establish a tcp connection to the server, downloading the main page, analyze its content and follow its links from the HTML code to get the objects required to build the page: javascript, css, images, etc…
To get the objects, it may open up to 6 or 7 TCP connections per domain name.
Once it has finished to download the objects, it starts aggregating everything then print out the page.

Limiting the number of connections per users
+++++++++++++++++++++++++++++++++++++++++++++

As seen before, a browser opens up 5 to 7 TCP connections to a website when it wants to download objetcs and they are opened quite quickly.
One can consider that somebody having more than 10 connections opened is not a regular user.
The configuration below shows how to do this limitation in the Aloha and HAProxy:
This configuration also applies to any kind of TCP based application.

The most important lines are from 25 to 32.::

	# On Aloha, the global section is already setup for you
	# and the haproxy stats socket is available at /var/run/haproxy.stats
	global
	  stats socket ./haproxy.stats level admin
	 
	defaults
	  option http-server-close
	  mode http
	  timeout http-request 5s
	  timeout connect 5s
	  timeout server 10s
	  timeout client 30s
	 
	listen stats
	  bind 0.0.0.0:8880
	  stats enable
	  stats hide-version
	  stats uri     /
	  stats realm   HAProxy Statistics
	  stats auth    admin:admin
	 
	frontend ft_web
	  bind 0.0.0.0:8080
	 
	  # Table definition  
	  stick-table type ip size 100k expire 30s store conn_cur
	 
	  # Allow clean known IPs to bypass the filter
	  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
	  # Shut the new connection as long as the client has already 10 opened 
	  tcp-request connection reject if { src_conn_cur ge 10 }
	  tcp-request connection track-sc1 src
	 
	  # Split static and dynamic traffic since these requests have different impacts on the servers
	  use_backend bk_web_static if { path_end .jpg .png .gif .css .js }
	 
	  default_backend bk_web
	 
	# Dynamic part of the application
	backend bk_web
	  balance roundrobin
	  cookie MYSRV insert indirect nocache
	  server srv1 192.168.1.2:80 check cookie srv1 maxconn 100
	  server srv2 192.168.1.3:80 check cookie srv2 maxconn 100
	 
	# Static objects
	backend bk_web_static
	  balance roundrobin
	  server srv1 192.168.1.2:80 check maxconn 1000
	  server srv2 192.168.1.3:80 check maxconn 1000

NOTE: if several domain name points to your frontend, then you may want to increase the conn_cur limit. (Remember a browser opens its 5 to 7 TCP connections per domain name).
NOTE2: if several users are hidden behind the same IP (NAT or proxy), this configuration may have a negative impact for them. You can whitelist these IPs.

Testing the configuration

run an apache bench to open 10 connections and doing request on these connections::

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
+++++++++++++++++++++++++++++++++++++

In the previous chapter, we’ve seen how to protect ourselves from somebody who wants to open more than X connections at the same time.
Well, this is good, but something which may kill performance would to allow somebody to open and close a lot of tcp connections over a short period of time.
As we’ve seen previously, a browser will open up to 7 TCP connections in a very short period of time (a few seconds). One can consider that somebody having more than 20 connections opened over a period of 3 seconds is not a regular user.
The configuration below shows how to do this limitation in the Aloha and HAProxy:
This configuration also applies to any kind of TCP based application.

The most important lines are from 25 to 32.::

	# On Aloha, the global section is already setup for you
	# and the haproxy stats socket is available at /var/run/haproxy.stats
	global
	  stats socket ./haproxy.stats level admin
	 
	defaults
	  option http-server-close
	  mode http
	  timeout http-request 5s
	  timeout connect 5s
	  timeout server 10s
	  timeout client 30s
	 
	listen stats
	  bind 0.0.0.0:8880
	  stats enable
	  stats hide-version
	  stats uri     /
	  stats realm   HAProxy Statistics
	  stats auth    admin:admin
	 
	frontend ft_web
	  bind 0.0.0.0:8080
	 
	  # Table definition  
	  stick-table type ip size 100k expire 30s store conn_rate(3s)
	 
	  # Allow clean known IPs to bypass the filter
	  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
	  # Shut the new connection as long as the client has already 10 opened 
	  tcp-request connection reject if { src_conn_rate ge 10 }
	  tcp-request connection track-sc1 src
	 
	  # Split static and dynamic traffic since these requests have different impacts on the servers
	  use_backend bk_web_static if { path_end .jpg .png .gif .css .js }
	 
	  default_backend bk_web
	 
	# Dynamic part of the application
	backend bk_web
	  balance roundrobin
	  cookie MYSRV insert indirect nocache
	  server srv1 192.168.1.2:80 check cookie srv1 maxconn 100
	  server srv2 192.168.1.3:80 check cookie srv2 maxconn 100
 
	# Static objects
	backend bk_web_static
	  balance roundrobin
	  server srv1 192.168.1.2:80 check maxconn 1000
	  server srv2 192.168.1.3:80 check maxconn 1000

NOTE2: if several users are hidden behind the same IP (NAT or proxy), this configuration may have a negative impact for them. You can whitelist these IPs.

Testing the configuration

run 10 requests with ApacheBench, everything may be fine::

	ab -n 10 -c 1 -r http://127.0.0.1:8080/

Using socat we can watch this traffic in the stick-table::

	# table: ft_web, type: ip, size:102400, used:1
	0x11faa3c: key=127.0.0.1 use=0 exp=28395 conn_rate(3000)=10

Running a telnet to run a eleventh request and the connections get closed::

	telnet 127.0.0.1 8080
	Trying 127.0.0.1...
	Connected to 127.0.0.1.
	Escape character is '^]'.
	Connection closed by foreign host.

Limiting the HTTP request rate
+++++++++++++++++++++++++++++++++

Even if in the previous examples, we were using HTTP as the protocol, we based our protection on layer 4 information: number or opening rate of TCP connections.
An attacker could respect the number of connection we would set by emulating the behavior of a regular browser.
Now, let’s go deeper and see what we can do on HTTP protocol.
The configuration below tracks HTTP request rate per user on the backend side, blocking abusers on the frontend side if the backend detects abuse.::

	# On Aloha, the global section is already setup for you
	# and the haproxy stats socket is available at /var/run/haproxy.stats
	global
	  stats socket ./haproxy.stats level admin
	 
	defaults
	  option http-server-close
	  mode http
	  timeout http-request 5s
	  timeout connect 5s
	  timeout server 10s
	  timeout client 30s
	 
	listen stats
	  bind 0.0.0.0:8880
	  stats enable
	  stats hide-version
	  stats uri     /
	  stats realm   HAProxy Statistics
	  stats auth    admin:admin
	 
	frontend ft_web
	  bind 0.0.0.0:8080
	 
	  # Use General Purpose Couter (gpc) 0 in SC1 as a global abuse counter
	  # Monitors the number of request sent by an IP over a period of 10 seconds
	  stick-table type ip size 1m expire 10s store gpc0,http_req_rate(10s)
	  tcp-request connection track-sc1 src
	  tcp-request connection reject if { src_get_gpc0 gt 0 }
	 
	  # Split static and dynamic traffic since these requests have different impacts on the servers
	  use_backend bk_web_static if { path_end .jpg .png .gif .css .js }
	 
	  default_backend bk_web
	 
	# Dynamic part of the application
	backend bk_web
	  balance roundrobin
	  cookie MYSRV insert indirect nocache
	 
	  # If the source IP sent 10 or more http request over the defined period, 
	  # flag the IP as abuser on the frontend
	  acl abuse src_http_req_rate(ft_web) ge 10
	  acl flag_abuser src_inc_gpc0(ft_web)
	  tcp-request content reject if abuse flag_abuser
	 
	  server srv1 192.168.1.2:80 check cookie srv1 maxconn 100
	  server srv2 192.168.1.3:80 check cookie srv2 maxconn 100
	 
	# Static objects
	backend bk_web_static
	  balance roundrobin
	  server srv1 192.168.1.2:80 check maxconn 1000
	  server srv2 192.168.1.3:80 check maxconn 1000

NOTE: if several users are hidden behind the same IP (NAT or proxy), this configuration may have a negative impact for them. You can whitelist these IPs.

Testing the configuration

run 10 requests with ApacheBench, everything may be fine::

	ab -n 10 -c 1 -r http://127.0.0.1:8080/

Using socat we can watch this traffic in the stick-table:::

	# table: ft_web, type: ip, size:1048576, used:1
	0xbebbb0: key=127.0.0.1 use=0 exp=8169 gpc0=1 http_req_rate(10000)=10

Running a telnet to run a eleventh request and the connections get closed::

	telnet 127.0.0.1 8080
	Trying 127.0.0.1...
	Connected to 127.0.0.1.
	Escape character is '^]'.
	Connection closed by foreign host.

Detecting vulnerability scans
+++++++++++++++++++++++++++++++

Vulnerability scans could generate different kind of errors which can be tracked by Aloha and HAProxy:

* invalid and truncated requests
* denied or tarpitted requests
* failed authentications
* 4xx error pages

HAProxy is able to monitor an error rate per user then can take decision based on it.::

	# On Aloha, the global section is already setup for you
	# and the haproxy stats socket is available at /var/run/haproxy.stats
	global
	  stats socket ./haproxy.stats level admin
	 
	defaults
	  option http-server-close
	  mode http
	  timeout http-request 5s
	  timeout connect 5s
	  timeout server 10s
	  timeout client 30s
	 
	listen stats
	  bind 0.0.0.0:8880
	  stats enable
	  stats hide-version
	  stats uri     /
	  stats realm   HAProxy Statistics
	  stats auth    admin:admin
	 
	frontend ft_web
	  bind 0.0.0.0:8080
	 
	  # Use General Purpose Couter 0 in SC1 as a global abuse counter
	  # Monitors the number of errors generated by an IP over a period of 10 seconds
	  stick-table type ip size 1m expire 10s store gpc0,http_err_rate(10s)
	  tcp-request connection track-sc1 src
	  tcp-request connection reject if { src_get_gpc0 gt 0 }
	 
	  # Split static and dynamic traffic since these requests have different impacts on the servers
	  use_backend bk_web_static if { path_end .jpg .png .gif .css .js }
	 
	  default_backend bk_web
	 
	# Dynamic part of the application
	backend bk_web
	  balance roundrobin
	  cookie MYSRV insert indirect nocache
	 
	  # If the source IP generated 10 or more http request over the defined period, 
	  # flag the IP as abuser on the frontend
	  acl abuse src_http_err_rate(ft_web) ge 10
	  acl flag_abuser src_inc_gpc0(ft_web)
	  tcp-request content reject if abuse flag_abuser
	 
	  server srv1 192.168.1.2:80 check cookie srv1 maxconn 100
	  server srv2 192.168.1.3:80 check cookie srv2 maxconn 100
	 
	# Static objects
	backend bk_web_static
	  balance roundrobin
	  server srv1 192.168.1.2:80 check maxconn 1000
	  server srv2 192.168.1.3:80 check maxconn 1000

Testing the configuration

run an apache bench, pointing it on a purposely wrong URL::

	ab -n 10 http://127.0.0.1:8080/dlskfjlkdsjlkfdsj

Watch the table content on the haproxy stats socket::

	echo "show table ft_web" | socat unix:./haproxy.stats -
	# table: ft_web, type: ip, size:1048576, used:1
	0x8a9770: key=127.0.0.1 use=0 exp=5866 gpc0=1 http_err_rate(10000)=11

Let’s try to run the same ab command and let’s get the error::

	apr_socket_recv: Connection reset by peer (104)

which means that HAProxy has blocked the IP address

Notes
+++++++++

We could combine configuration example above together to improve protection. This will be described later in an other article
The numbers provided in the examples may be different for your application and architecture. Bench your configuration properly before applying in production.
Related articles
Fight spam with early talking detection
Protect Apache against Apache-killer script
Protect your web server against slowloris
Links
HAProxy Technologies
Aloha load balancer: HAProxy based LB appliance
HAPEE: HAProxy Enterprise Edition

