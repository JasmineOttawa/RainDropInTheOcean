---
layout: post
title: Install Rocky on CentOS7.5 
category: OpenStack
tags: [OpenStack]
---

Refer to https://docs.openstack.org/install-guide/

The two nodes are:  
controller  management_IP11  provider_IP12    
compute     management_IP21  provider_IP22    
 
steps include:   
1, setup environment   
2, install identity service   
3, install image service   
4, install compure service   
5, networking service   
6, dashboard service   

# 1, setup environment 
1.1) stop firewalld, disable selinux, update packages, reboot node, config hostname, /etc/hosts  
```
systemctl restart network
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/=enforcing/=disabled/' /etc/selinux/config
yum upgrade -y
reboot 

hostnamectl set-hostname controller
hostnamectl set-hostname compute
# cat << EOF >> /etc/hosts
management_IP11 controller
management_IP21 compute
EOF
```

1.2) configure NTP- Network Time Protocol   
```
# on controll node 
yum install chrony
cd /etc
cp chrony.conf chtony.conf.bak
vi chrony.conf 
server controller iburst
allow 192.168.0.0/16    

systemctl enable chronyd 
systemctl start chronyd 
systemctl status chronyd 
# same config on compute node, just omit 'allow 192.168.0.0/16'
```

Here we need to change timezone on compute node: 
```
[root@controller etc]# ls -ltr /etc/localtime
lrwxrwxrwx. 1 root root 37 Jan 23 09:36 /etc/localtime -> ../usr/share/zoneinfo/America/Toronto
[root@controller etc]# timedatectl | grep -i 'time zone'
       Time zone: America/Toronto (EST, -0500)

timedatectl set-timezone America/Toronto
```

1.3) OpenStack packages   
```
# enable OpenStack repository 
yum install centos-release-openstack-rocky
# upgrade packages, thousands of packages upgraded
yum upgrade   
# install OpenStack client 
yum install python-openstackclient
#RHEL & centOS enables SElinux by default, to automatically manage security policies for OpenStack services
yum install openstack-selinux    
```
1.4) SQL database  
```
yum install mariadb mariadb-server python2-PyMySQL
# set bind-address key to the management IP address of the controller node 
cd /etc/my.cnf.d
vi openstack.cnf 
[mysqld]
bind-address = management_IP11
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8   
   
# enable system start when system reboots  
systemctl enable mariadb.service
# start database  
systemctl start mariadb.service
systemctl status mariadb.service

# secure database service 
mysql_secure_installation
set root password to: DBPASS
```
1.5) Message queue   
OpenStack uses a message queue to coordinate operations and status information among services.   
Here we use RabbitMQ.   
```
# install package 
yum install rabbitmq-server
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
# add openstack user, password DBPASS
rabbitmqctl add_user openstack DBPASS
# Permit configuration, write, and read access for the openstack user:
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
1.6) memcached   
```
yum install memcached python-memcached
cd /etc/sysconfig 
cp memcached memcached.jan2019 
vi /etc/sysconfig/memcached 
# lock down the memcache server so that it only listens for connections from the hosts that need to be served, by default it listens to connections from all addresses. 
OPTIONS="-l 127.0.0.1,::1,management_IP11"
# systemctl enable memcached.service
# systemctl start memcached.service
```
1.7) etcd   
```
yum install etcd
cd /etc/etcd 
cp etcd.conf etcd.conf.jan2019
vi etcd.conf 
# set management IP address of the controller node to enable access by other nodes via the management network: 
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://management_IP11:2380"
ETCD_LISTEN_CLIENT_URLS="http://management_IP11:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://management_IP11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://management_IP11:2379"
ETCD_INITIAL_CLUSTER="controller=http://management_IP11:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"

systemctl enable etcd
systemctl start etcd
```
# 2, identity service 
```
mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'DBPASS';

cd /etc/keystone
cp keystone.conf keystone.conf.jan2019 
vi keystone.conf 
[database]
# ...
connection = mysql+pymysql://keystone:DBPASS@management_IP11/keystone
[token]
# ...
provider = fernet

# Populate the identity service database   
su -s /bin/sh -c "keystone-manage db_sync" keystone
# Initialize Fernet key repositories: 
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
# Bootstrap the Identity service
keystone-manage bootstrap --bootstrap-password DBPASS \
  --bootstrap-admin-url http://management_IP11:5000/v3/ \
  --bootstrap-internal-url http://management_IP11:5000/v3/ \
  --bootstrap-public-url http://management_IP11:5000/v3/ \
  --bootstrap-region-id RegionOne
  
# configure Apache HTTP server 
cd  /etc/httpd/conf
cp  httpd.conf httpd.conf.jan2019 
vi httpd.conf 
ServerName management_IP11
# Create a link to the /usr/share/keystone/wsgi-keystone.conf file:
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

systemctl enable httpd.service
systemctl start httpd.service
systemctl status httpd.service
```
Configure the admin environment  
```
# admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=DBPASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://management_IP11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

Create a domain, projects, users, and roles 
The defaul domain already exists from the keystone-manage bootstrap, you can create a new domain by: 
[root@controller ~]# openstack domain create --description "Domain Ottawa" example
An unexpected error prevented the server from fulfilling your request. (HTTP 500) (Request-ID: req-d68c8a5b-7e5f-4ddb-a514-0683f4ab41a0)
-- uses a service project that contains a unique user for each service that you add to your environment
[root@controller ~]# openstack project create --domain default --description "Service Project" service
An unexpected error prevented the server from fulfilling your request. (HTTP 500) (Request-ID: req-10978cfd-b05b-466f-9f6e-a9c60786f969)

### message1: SELinux policy enabled
If not diable firewall and disable Seselinux at the beginning, will see this message:   
```
[root@controller httpd]# pwd
/var/log/httpd
[root@controller httpd]# cat error_log
[Mon Jan 28 15:39:13.619491 2019] [core:notice] [pid 15229] SELinux policy enabled; httpd running as context system_u:system_r:httpd_t:s0
```

### message2: Table 'keystone.project' doesn't exist
```
/var/log/httpd/keystone_access.log shows:   
management_IP11 - - "POST /v3/auth/tokens HTTP/1.1" 500 143 "-" "openstacksdk/0.17.2 keystoneauth1/3.10.0 python-requests/2.19.1 CPython/2.7.5"
/var/log/keystone/keystone.log shows:  
2019-01-29 10:31:06.427 12237 ERROR keystone.common.wsgi ProgrammingError: (pymysql.err.ProgrammingError) (1146, u"Table 'keystone.project' doesn't exist")
```
It turns out following command was not run successfully, and there is no table keystone.project 
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
After sync data, project is created successfully 
```
[root@controller keystone]# openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 9a0cbf228aa24580bd45a4607dd7f50e |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```
Create a new project  
```
openstack project create --domain default --description "AAA Project" projectAAA 
openstack user create --domain default  --password-prompt userAAA    
openstack role create roleAAA 
openstack role add --project projectAAA --user userAAA roleAAA
openstack user list 
```
Verify operation
```
# Unset the temporary OS_AUTH_URL and OS_PASSWORD environment variable:
unset OS_AUTH_URL OS_PASSWORD
# As the admin user, request an authentication token
[root@controller keystone]# openstack --os-auth-url http://management_IP11:5000/v3 \
>   --os-project-domain-name Default --os-user-domain-name Default \
>   --os-project-name admin --os-username admin token issue

# As the myuser user created in the previous section, request an authentication token:
[root@controller keystone]# openstack --os-auth-url http://management_IP11:5000/v3 \
>   --os-project-domain-name Default --os-user-domain-name Default \
>   --os-project-name projectAAA --os-username userAAA token issue
```     

Create OpenStack client environment scripts   
```
mkdir -p ~/rocky
cd ~/rocky 
#admin-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=DBPASS
export OS_AUTH_URL=http://management_IP11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

#userAAA-openrc 
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=projectAAA
export OS_USERNAME=userAAA
export OS_PASSWORD=userAAA
export OS_AUTH_URL=http://management_IP11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
[root@controller rocky]# . admin-openrc
[root@controller rocky]# openstack token issue

#3, image service -- glance 
3.1) prerequisites   
```
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%'  IDENTIFIED BY 'DBPASS';

# Source the admin credentials to gain access to admin-only CLI commands:
. admin-openrc

openstack user create --domain default --password-prompt glance  # password=glance 
openstack role add --project service --user glance admin  # Add the admin role to the glance user and service project:
openstack service create --name glance --description "OpenStack Image" image  # create glance service 
openstack endpoint create --region RegionOne image public http://management_IP11:9292 # create image service API endpoints 
openstack endpoint create --region RegionOne image internal http://management_IP11:9292
openstack endpoint create --region RegionOne image admin http://management_IP11:9292
```
3.2) install and configure components   
```
yum install openstack-glance
cd /etc/glance/ 
cp glance-api.conf glance-api.conf.bak 
[database]
# ...
connection = mysql+pymysql://glance:DBPASS@management_IP11/glance
[keystone_authtoken]
# ...
www_authenticate_uri  = http://management_IP11:5000
auth_uri = http://management_IP11:5000
memcached_servers = management_IP11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance

[paste_deploy]
# ...
flavor = keystone
[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

cp glance-registry.conf glance-registry.conf.bak
vi glance-registry.conf
[database]
# ...
connection = mysql+pymysql://glance:DBPASS@management_IP11/glance
[keystone_authtoken]
# ...
www_authenticate_uri = http://management_IP11:5000
auth_url = http://management_IP11:5000
memcached_servers = management_IP11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance

[paste_deploy]
# ...
flavor = keystone

# Populate the Image service database:
su -s /bin/sh -c "glance-manage db_sync" glance
```

3.3) finalize installation  
systemctl enable openstack-glance-api.service  openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service

### message3, glance-api service failed 
[root@controller glance]# systemctl status openstack-glance-api.service
● openstack-glance-api.service - OpenStack Image Service (code-named Glance) API server
   Loaded: loaded (/usr/lib/systemd/system/openstack-glance-api.service; enabled; vendor preset: disabled)
   Active: activating (auto-restart) (Result: exit-code) since Tue 2019-01-29 11:45:37 EST; 12ms ago
  Process: 25730 ExecStart=/usr/bin/glance-api (code=exited, status=99)
 Main PID: 25730 (code=exited, status=99)

journalctl -xn    
it turned out to be config file error     
it's supposed to be   
auth_url = http://management_IP11:5000  
instead of     
auth_uri = http://management_IP11:5000, this uri is what's in the config file originally, need change to url  


3.4) verification   
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public

#4, compute service 
1) install controller node   
4.1.1) prerequisites   
```
mysql -u root -p
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
CREATE DATABASE placement;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'  IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%'  IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'  IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'  IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%'  IDENTIFIED BY 'DBPASS';
  
openstack user create --domain default --password-prompt nova   # password=nova 
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://management_IP11:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
openstack endpoint list 
openstack endpoint delete 9d328883ba554b7a88b95267bf873c48
openstack endpoint delete ad837bbb4f2048768bd7761c86e82951 
openstack endpoint create --region RegionOne compute internal http://management_IP11:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://management_IP11:8774/v2.1

openstack user create --domain default --password-prompt placement  #password=placement 
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://management_IP11:8778
openstack endpoint create --region RegionOne placement internal http://management_IP11:8778
openstack endpoint create --region RegionOne placement admin http://management_IP11:8778
```

4.1.2) install and configure components   
```
yum install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api
cd  /etc/nova/
cp nova.conf nova.conf.bak 
vi nova.conf
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
[api_database]
# ...
connection = mysql+pymysql://nova:DBPASS@management_IP11/nova_api

[database]
# ...
connection = mysql+pymysql://nova:DBPASS@management_IP11/nova

[placement_database]
# ...
connection = mysql+pymysql://placement:DBPASS@management_IP11/placement
  
[DEFAULT]
# ...
transport_url = rabbit://openstack:DBPASS@controller

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://management_IP11:5000/v3
memcached_servers = management_IP11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova

[DEFAULT]
# ...
my_ip = management_IP11     #management interface IP address of the controller node
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver  

Configure the [neutron] section of /etc/nova/nova.conf. leaving to networking part 

[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
# ...
api_servers = http://management_IP11:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://management_IP11:5000/v3
username = placement
password = placement

Due to a packaging bug, you must enable access to the Placement API by adding the following configuration to /etc/httpd/conf.d/00-nova-placement-api.conf:
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>

systemctl restart httpd
su -s /bin/sh -c "nova-manage api_db sync" nova   #populate nova-api, placement database 
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova  #register cell0 database 
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova   #create cell1 cell 
su -s /bin/sh -c "nova-manage db sync" nova   # populate nova database 
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova  #Verify nova cell0 and cell1 are registered correctly


[root@controller conf.d]# su -s /bin/sh -c "nova-manage db sync" nova
/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1831, u'Duplicate index `block_device_mapping_instance_uuid_virtual_name_device_name_idx`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
/usr/lib/python2.7/site-packages/pymysql/cursors.py:170: Warning: (1831, u'Duplicate index `uniq_instances0uuid`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
[root@DBPASS conf.d]# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+------------------------------------+------------------------------------------------------+----------+
|  Name |                 UUID                 |           Transport URL            |                 Database Connection                  | Disabled |
+-------+--------------------------------------+------------------------------------+------------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |               none:/               | mysql+pymysql://nova:****@management_IP11/nova_cell0 |  False   |
| cell1 | bcb4f5f8-36e3-49e5-aa74-cf8023a90f36 | rabbit://openstack:****@controller |    mysql+pymysql://nova:****@management_IP11/nova    |  False   |
+-------+--------------------------------------+------------------------------------+------------------------------------------------------+----------+
```
4.1.3) finalize installation 
```
systemctl enable openstack-nova-api.service \
  openstack-nova-scheduler.service openstack-nova-conductor.service \
  openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service 
systemctl start openstack-nova-scheduler.service 
systemctl start openstack-nova-conductor.service 
systemctl start openstack-nova-novncproxy.service
systemctl status openstack-nova-api.service       # not healthy 
systemctl status openstack-nova-scheduler.service
systemctl status openstack-nova-conductor.service
systemctl status openstack-nova-novncproxy.service   # fine 
```

### message4,   nova-api service failed 
/var/log/nova-api.log shows:  
```
2019-01-30 09:37:19.491 40867 CRITICAL nova [-] Unhandled error: ConfigFileValueError: Value for option server_listen is not valid: #my_ip is not a valid host address
[root@controller nova]# grep server_listen nova.conf
#server_listen=127.0.0.1
# * This option depends on the ``server_listen`` option.
#   ``server_listen`` using the value of this option.
# Deprecated group;name - DEFAULT;vncserver_listen
# Deprecated group;name - [vnc]/vncserver_listen
server_listen=#my_ip
```
change it to server_listen=$my_ip , now nova-api service up 

4.2) install compute node 
This configuration uses the Quick EMUlator (QEMU) hypervisor with the kernel-based VM (KVM) extension on compute nodes that support hardware acceleration for VMs.
4.2.1) install and configure components 
```
yum install openstack-nova-compute
cd /etc/nova 
cp nova.conf nova.conf.bak 
vi /etc/nova/nova.conf
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS
[DEFAULT]
# ...
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
[glance]
# ...
api_servers = http://controller:9292
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```
3.2) finalize installation  
Determine whether your computr node supports hardware acceleration for VM: 
```
egrep -c '(vmx|svm)' /proc/cpuinfo
```

If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.  
If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.  
```
[libvirt]
# ...
virt_type = qemu
```
```
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```
3.3) add the compute node to the cell database 
```
. admin-openrc
openstack compute service list --service nova-compute
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```
4) verify operation 
```
. admin-openrc
openstack compute service list
openstack catalog list    # list API endpoints in the identity service to verify connectivity with the identity service 
openstack image list
nova-status upgrade check  # check the cells and placement API are working successfully 
```
#5, networking service 
5.1) on controller node 
```
mysql -u root -p 
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';

. admin-openrc
openstack user create --domain default --password-prompt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

You can deploy the Networking service using one of two architectures represented by options 1 and 2.  
Option 1 deploys the simplest possible architecture that only supports attaching instances to provider (external) networks. No self-service (private) networks, routers, or floating IP addresses. Only the admin or other privileged user can manage provider networks.  
Option 2 augments option 1 with layer-3 services that support attaching instances to self-service networks. The demo or other unprivileged user can manage self-service networks including routers that provide connectivity between self-service and provider networks. Option 2 also supports attaching instances to provider networks. 

choose option2, Self-service networks 
```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
cd /etc/neutron
cp neutron.conf neutron.conf.bak 
vi neutron.conf 
[database]
# ...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
[DEFAULT]
# ...
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

#### Configure the Modular Layer 2 (ML2) plug-in
```
cd /etc/neutron/plugins/ml2
cp ml2_conf.ini ml2_conf.ini.bak 
vi ml2_conf.ini 
[ml2]
# enable flat, VLAN, and VXLAN networks
type_drivers = flat,vlan,vxlan
# enable VXLAN self-service networks
tenant_network_types = vxlan
# enable the Linux bridge and layer-2 population mechanisms
mechanism_drivers = linuxbridge,l2population
# enable the port security extension driver
extension_drivers = port_security
[ml2_type_flat]
# configure the provider virtual network as a flat network
flat_networks = provider
[ml2_type_vxlan]
# configure the VXLAN network identifier range for self-service networks
vni_ranges = 1:1000
[securitygroup]
# enable ipset to increase efficiency of security group rules
enable_ipset = true
```
Note, After you configure the ML2 plug-in, removing values in the type_drivers option can lead to database inconsistency

#### Configure the Linux bridge agent

#### Configure the layer-3 agent

#### Configure the DHCP agent

5.2) on compute node 

5.3) verify 

#6, dashboard 










