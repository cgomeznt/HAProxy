
Activar el LOG en Haproxy
===================

Editar el archivo /etc/haproxy/haproxy.cfg y agregar la line "log         127.0.0.1 local2"

	# vi /etc/haproxy/haproxy.cfg
		[...]
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

		[...]
		

Editar el archivo /etc/rsyslog.conf y agregar o editar estas lineas::

	$ModLoad imudp
	$UDPServerRun 514

Esta es opcional::

	$UDPServerAddress 127.0.0.1
	
Editamos el archivo de configuración del Rsyslog::

	vi /etc/rsyslog.conf
		[...]
		$ModLoad imudp
		$UDPServerRun 514

		local2.*                       /var/log/haproxy.log
		[...]
	
Si queremos los LOGs por separado::

	local2.=info   /var/log/haproxy-in.log
	local2.notice  /var/log/haproxy-stat.log

Reiniciamos el Rsyslog y el Haproxy::

	systemctl restart rsyslog.service
	systemctl restart haproxy.service
