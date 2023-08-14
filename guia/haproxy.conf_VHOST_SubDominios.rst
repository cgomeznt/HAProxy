HAProxy For Load Balancing and Subdomain-Port Redirection
==========================================================

Queremos atedens todos los subdominios "*.dominio.com" o cualquier URL que haga el request 
y sea capaz de hacer el forward un servidor en donde estan los VHOST y le pase el http-request set-header
de esta forma el VHOST podra saber cual es el ServerName que le corresponde



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

frontend http-in
    bind *:80
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
