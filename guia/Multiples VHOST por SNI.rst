
#####################################################################
#
#              Inicio Frontend SNI-Desarrollo (Server Name Indication)
#
#####################################################################

frontend https-fe-SNI
    log 127.0.0.1 local6 debug
    option httplog
    reqadd X-Forwarded-Proto:\ https
    option forwardfor except 127.0.0.1
    log global
    capture request  header    Referrer             len 64
    capture request  header    Content-Length       len 10
    capture request  header    User-Agent           len 64
    capture request  header    X-Request-UID        len 500
    capture request  header    X-Forwarded-For      len 500
    capture request  header    Host                 len 500
    capture request  header    X-Unique-ID          len 16
    mode http

    # bind 10.134.4.62:9443 ssl crt /etc/ssl/ipgwebdes.mgmt.credicard.com.ve.pem verify none no-sslv3 force-tlsv12
    bind 10.134.4.62:9443 ssl crt /etc/ssl/iswebdes.mgmt.credicard.com.ve.pem crt /etc/ssl/ipgwebdes.mgmt.credicard.com.ve.pem verify none no-sslv3 force-tlsv12

    acl ipg_comercio_acl path_beg /tucuprueba

    # default_backend / ACL DEFINITIONS para definir cual fue el SNI seleccionado #
    use_backend bk_ipgwebdes_redirect if  ipg_comercio_acl  # Redirect if ACL
    use_backend bk_ipgwebdes if { ssl_fc_sni ipgwebdes.mgmt.credicard.com.ve } # content switching based on SNI
    use_backend bk_wso2is if { ssl_fc_sni iswebdes.mgmt.credicard.com.ve } # content switching based on SNI

#####################################################################
#
#              FIN Frontend SNI-Desarrollo (Server Name Indication)
#
#####################################################################






#####################################################################
#
#              Inicio Backend-SNI-Desarrollo de IPG (Server Name Indication)
#
#####################################################################


# Backend IPG-DES
backend bk_ipgwebdes
    server srv-vipgwebdes 10.134.3.63:443 ssl verify none

#####################################################################
#
#             FIN Backend-SNI-Desarrollo de IPG (Server Name Indication)
#
#####################################################################


#####################################################################
#
#              Inicio Backend-SNI-Desarrollo de WSOIS (Server Name Indication)
#
#####################################################################


# Backend WSO2IS-DES
backend bk_wso2is
    server srv-vwso2isd01 10.134.4.54:9443 ssl verify none

#####################################################################
#
#             FIN Backend-SNI-Desarrollo de WSO2 (Server Name Indication)
#
#####################################################################


#####################################################################
#
#              Inicio Backend-Redirect-SNI-Desarrollo de IPG (Server Name Indication)
#
#####################################################################

backend bk_ipgwebdes_redirect
   redirect location https://10.134.4.54:9443

#####################################################################
#
#              FIN Backend-Redirect-SNI-Desarrollo de IPG (Server Name Indication)
#
#####################################################################




https://ipgwebdes.mgmt.credicard.com.ve:9443/
https://ipgwebdes.mgmt.credicard.com.ve:9443/comercio

https://iswebdes.mgmt.credicard.com.ve:9443/


https://iswebdes.mgmt.credicard.com.ve:9443/tucuprueba
https://ipgwebdes.mgmt.credicard.com.ve:9443/tucuprueba