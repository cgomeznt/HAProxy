# cat /etc/haproxy/haproxy.cfg

    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

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



frontend intranet
        mode http
        bind :80
        bind :443 ssl crt /etc/httpd/conf.d/certs/intranet.credicard.com.ve.pem
        http-request redirect scheme https unless { ssl_fc }
        acl acl_admin path_beg /Intranet_Credicard
        acl acl_Autogestion path_beg /Autogestion
        use_backend server_tomcat if acl_admin
        use_backend server_tomcat if acl_Autogestion
        default_backend server_intranet

backend server_intranet
        server lcsprdappintranet 10.132.0.232:9443 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 ssl verify none

backend server_tomcat
        server lcsprdappintranet 10.132.0.232:8443 check inter 2s downinter 5s slowstart 60s rise 2 fall 3 ssl verify none
