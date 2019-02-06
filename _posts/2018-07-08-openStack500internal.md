---
layout: post
title: Create instance in OpenStack Queens got message "500 Internal Server Error"
category: OpenStack
tags: [OpenStack]
---

# Question 
Create instance on openstack queens got error “500 Internal Server Error”

# Investigation 
/var/log/nova/nova-scheduler.log shows 500 internel server error : 
```
The server encountered an internal error or misconfiguration and was unable to complete your request.
```

Go to server error log, /var/log/httpd/placement_wsgi_error.log: 
## message 1  
```
ConfigFileParseError: Failed to parse /etc/nova/nova.conf: at /etc/nova/nova.conf:11307, No ‘:’ or ‘=’ found in assignment: ‘openstack-config –set
```

So, /etc/nova/nova.conf was modified, following 2 lines were added at the end: 
openstack-config –set /etc/nova/nova.conf DEFAULT compute_driver libvirt.LibvirtDriver
openstack-config –set /etc/nova/nova.conf libvirt virt_type kvm

remove these 2 lines, the config is already in the file, leave it as is, they’re same as default value.

## now got message 2 
```
mod_wsgi (pid=14970): Target WSGI script ‘/var/www/cgi-bin/nova/nova-placement-api’ cannot be loaded as Python module.
mod_wsgi (pid=14970): Exception occurred processing WSGI script ‘/var/www/cgi-bin/nova/nova-placement-api’.
ArgsAlreadyParsedError: arguments already parsed: cannot register CLI option
```
as per https://ask.openstack.org/en/question/6626/nova-arguments-already-parsed-cannot-register-cli-option/
This error comes up when the attribute has been specified repeatedly and with different values at different config files.  

Restarting httpd service by: systemctl restart httpd   
now instance could be created succesfully.  

Manually testing placement API: 
```
sudo -H -u nova bash -c ‘/var/www/cgi-bin/nova/nova-placement-api’ 
bash: /var/www/cgi-bin/nova/nova-placement-api: Permission denied 
chmod 744 /var/www/cgi-bin/nova/nova-placement-api 
```
## manually test placement API got message 3 
```
sudo -H -u nova bash -c ‘/var/www/cgi-bin/nova/nova-placement-api’ 
/usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:332: NotSupportedWarning: Configuration option(s) [‘use_tpool’] not supported 
exception.NotSupportedWarning STARTING test server nova.api.openstack.placement.wsgi.init_application 
Available at http://localhost.localdomain:8000/
DANGER! For testing only, do not use in production
```

Refer to https://github.com/openstack/oslo.db/commit/c432d9e93884d6962592f6d19aaec3f8f66ac3a2
made following change to eliminate this message:   
```
cd /usr/lib/python2.7/site-packages/oslo_db/sqlalchemy   
diff enginefacade.py enginefacade.py.20180712   
175c175 
‘db_max_retry_interval’, ‘backend’,’use_tpool’])
‘db_max_retry_interval’, ‘backend’])
```