haproxy, remove 'app' after selecting backend
============================================



Code to change a request from / to /app1/::

  reqirep  ^([^\ :]*)\ /(.*) \1\ /app1/\2

If urls in the response contain absolute urls it might be required to use this::

  acl    no_redir url_beg   /app1/
  reqirep  ^([^\ :]*)\ /(.*) \1\ /app1/\2 if !no_redir

The code makes sure that the method and url-path behind the / stays the same. Which method you need exactly might depend on the application thats running.
For readability of the above how change a request from /app1/ to /app1/app1redir/::

  reqirep  ^([^\ :]*)\ /app1/(.*) \1\ /app1/app1redir/\2

If those above dont work you might still be able to get a acceptable workaround by using a redirect::

  acl    no_redir url_beg   /app1/
  http-request redirect location http://%[req.hdr(Host)]/app1/ if !no_redir
  
  ================================================================================
  
Ejemplo de un haproxy.conf
++++++++++++++++++++++++++++++

haproxy.conf, ejemplo de como haproxy, remueve 'sample' o lo remplaza por 'Autogestion' despues selecciona el backend. Si queremos que sea en blanco, pues eliminas la palabra Autogestion::
  
	# cat /etc/haproxy/haproxy.cfg
	#---------------------------------------------------------------------
	# Example configuration for a possible web application.  See the
	# full configuration options online.
	#
	#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
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
	    timeout http-request    10s
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
	#frontend  main *:5000
	#    acl url_static       path_beg       -i /static /images /javascript /stylesheets
	#    acl url_static       path_end       -i .jpg .gif .png .css .js
	#
	##    use_backend static          if url_static
	##    default_backend             app

	frontend intranet
		mode http
		bind :80
		bind :443 ssl crt /etc/httpd/conf.d/certs/intranetqa.credicard.com.ve.pem
		http-request redirect scheme https unless { ssl_fc }
		acl acl_admin path_beg /Intranet_Credicard
		acl acl_Autogestion path_beg /Autogestion
		acl acl_BI path_beg /BI/
		use_backend server_tomcat if acl_admin
		use_backend server_tomcat if acl_Autogestion
		use_backend server_BI if acl_BI

	#============================================
	#haproxy, remove 'sample' o lo remplaza por 'Autogestion' despues selecciona el backend 
	#============================================
	acl acl_no_redir url_beg /sample/
	reqirep  ^([^\ :]*)\ /sample/(.*) \1\ /Autogestion\2 if acl_no_redir
	acl acl_sample url_beg /sample
	use_backend server_sample if acl_sample

	acl acl_reserva url_beg /reserva
	use_backend server_reserva if acl_reserva


		default_backend server_intranet


	backend server_intranet
		server lcsqaappintranet  10.134.0.81:9443 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 ssl verify none

	backend server_tomcat
		server lcsqaappintranet  10.134.0.81:8443 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 ssl verify none

	backend server_sample
		mode http
		server roomreser1 10.134.0.81:8443 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 ssl verify none

	backend server_reserva
		mode http
		redirect location https://roomreser.credicard.com.ve:31900/
		#server roomreser1 10.134.0.167:31900 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 ssl verify none

	backend server_BI
		mode http
		#option httpchk
		#option forwardfor
		#redirect location http://lcsqaapptableau.credicard.com.ve
		#reqrep ^([^\ ]*\ /)BI[/]?(.*)     \1\2
		#reqrep ^([^\ :]*)\ /BI[/]?(.*)  \1\ /signin/\2
		reqrep ^([^\ ]*\ /)BI[/]?(.*)     \1\2
		server lcsqaapptableau  10.134.3.195:80 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 # ssl verify none


	#---------------------------------------------------------------------
	# static backend for serving up images, stylesheets and such
	#---------------------------------------------------------------------
	#backend static
	#    balance     roundrobin
	#    server      static 127.0.0.1:4331 check
	#
	#---------------------------------------------------------------------
	# round robin balancing between the various backends
	#---------------------------------------------------------------------
	#backend app
	#    balance     roundrobin
	#    server  app1 127.0.0.1:5001 check
	#    server  app2 127.0.0.1:5002 check
	#    server  app3 127.0.0.1:5003 check
	#    server  app4 127.0.0.1:5004 check
	###
