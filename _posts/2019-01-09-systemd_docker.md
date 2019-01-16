---
layout: post
title: create a docker service unit in systemd   
category: Others
tags: [systemd kubernetes]
---

Linux was using SysVInit and BSD init, Then came Upstart and systemd, now Upstart is being retired in favor of systemd.    
One of its key feature is faster init process. Here's the systemd basics before creating service unit:  

### systemd unit files locations 
```
/usr/lib/systemd/system: systemd unit files distributed with installed RPM packages 
/run/systemd/system    : systemd unit files created at runtime
/etc/systemd/system    : Systemd unit files created by systemctl enable as well as unit files added for extending a service， customized unit files are added here. 
```
### unit file names take the following form: 
   unit_name.type_extension 
### 12 unit types  
```
Unit Type	    File Extension	Description
Service unit	.service	    A system service.
Target unit	    .target	        A group of systemd units.
Automount unit	.automount	    A file system automount point.
Device unit	    .device	        A device file recognized by the kernel.
Mount unit	    .mount	        A file system mount point.
Path unit	    .path	        A file or directory in a file system.
Scope unit	    .scope	        An externally created process.
Slice unit	    .slice	        A group of hierarchically organized units that manage system processes.
Snapshot unit	.snapshot	    A saved state of the systemd manager.
Socket unit	    .socket	        An inter-process communication socket.
Swap unit	    .swap	        A swap device or a swap file.
Timer unit	    .timer	        A systemd timer.
```
### unit file structure 
Unit files typically consist of 3 sections: 
```
   [unit] -  contains generic options that are not dependent on the type of the unit.
             Description, PartOf, Requires, Wants, Conflicts, After, Before 
   [unit type] - if a unit has type-specific directives, these are grouped under a section named after the unit type. take service as an example: 
    type : configures the unit process startup type that affects the functionality of ExecStart and related option. possible values are: 
        simple - default, process started with ExecStart is the main process of the service 
        forking - The process started with ExecStart spawns a child process that becomes the main process of the service. The parent process exits when the startup is complete.
        oneshot - This type is similar to simple, but the process exits before starting consequent units, can multiple custom commands that are then executed sequentially.
        dbus -  This type is similar to simple, but consequent units are started only after the main process gains a D-Bus name.
        notify - This type is similar to simple, but consequent units are started only after a notification message is sent via the sd_notify() function.
        idle - similar to simple, the actual execution of the service binary is delayed until all jobs are finished, which avoids mixing the status output with shell output of services.
    ExecStart : Specifies commands or scripts to be executed when the unit is started. ExecStartPre and ExecStartPost specify custom commands to be executed before and after it
   [install] -  contains information about unit installation used by systemctl enable and disable commands
     Alias: Most systemctl commands, excluding systemctl enable, can use aliases instead of the actual unit name.
     RequiredBy: 
     WantedBy: A list of units that weakly depend on the unit. When this unit is enabled, the units listed in WantedBy gain a Want dependency on the unit.
```

### create a docker service on RHEL7.3 
#### step1, create service and socket unit files 
cat /etc/systemd/system/docker.service
```
[Unit]
Description=docker

[Service]
Type=simple
ExecStart=/usr/bin/dockerd -H fd:// --insecure-registry=youregistry:port

[Install]
Alias=docker
```

cat /etc/systemd/system/docker.socket
```
[Unit]
Description=Docker Socket for the API
PartOf=docker.service

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
```
#### step2, start docker service 
```
systemtl daemon-reload 
systemctl start docker 
```
Got message:  no sockets found via socket activation 

Tried to start socket manually by:
```
systemctl enable docker.socket
systemctl start docker.socket
```
Now got message: docker.socket control process exited, code=exited status=216

Reviewing docker,socket, note that it tries to set the ownership of the docker.socket to root:docker
check if group docker exists: 
```
[root@node1 system]# cat /etc/group | grep docker
[root@node1 system]# groupadd docker
[root@node1 system]# cat /etc/group | grep docker
docker:x:1001:
[root@node1 system]# systemctl start docker.socket
[root@node1 system]# systemctl status docker.socket
```
Now socket works, restarting docker service, works fine, too 
```
[root@node1 system]# systemctl start docker
[root@node1 system]# systemctl status docker
```

## Q & A   

### Q1 -  what does -H fd:// means?
-H fd:// will tell Docker that the sevice is being started by systemd and will use socket activation. 
systemd will then create target socket and pass it to the Docker daemon to use. 

  
### Q2 -  Since one of its key feature is faster init process, how fast it is? 
[root@node1 system]# systemd-analyze blame
         33.285s httpd.service
         15.064s unbound-anchor.service
          7.890s openstack-nova-api.service
          5.236s network.service
  
Here is the detail reference for systemd   
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sect-managing_services_with_systemd-unit_files

