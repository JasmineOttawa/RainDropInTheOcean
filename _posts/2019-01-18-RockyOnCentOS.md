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

Create a domain, projects, users, and roles.   
The defaul domain already exists from the keystone-manage bootstrap, you can create a new domain by:   
```
[root@controller ~]# openstack domain create --description "Domain Ottawa" example
An unexpected error prevented the server from fulfilling your request. (HTTP 500) (Request-ID: req-d68c8a5b-7e5f-4ddb-a514-0683f4ab41a0)
```

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

# 3, image service -- glance   
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
```
systemctl enable openstack-glance-api.service  openstack-glance-registry.service  
systemctl start openstack-glance-api.service openstack-glance-registry.service   
```

### message3, glance-api service failed 
```
[root@controller glance]# systemctl status openstack-glance-api.service
● openstack-glance-api.service - OpenStack Image Service (code-named Glance) API server
   Loaded: loaded (/usr/lib/systemd/system/openstack-glance-api.service; enabled; vendor preset: disabled)
   Active: activating (auto-restart) (Result: exit-code) since Tue 2019-01-29 11:45:37 EST; 12ms ago
  Process: 25730 ExecStart=/usr/bin/glance-api (code=exited, status=99)
 Main PID: 25730 (code=exited, status=99)

journalctl -xn      # display the most recent 10 entries of systemd   
it turned out to be config file error     
it's supposed to be   
auth_url = http://management_IP11:5000  
instead of     
auth_uri = http://management_IP11:5000, this uri is what's in the config file originally, need change to url  
```

3.4) verification   
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img  
openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public  

# 4, compute service   
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
vi nova.conf
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata
[DEFAULT]
# ...
transport_url = rabbit://openstack:kos2000@controller
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
password = nova
[DEFAULT]
# this is the management_interface IP of compute node 
my_ip = 192.168.151.116  
# By default, Compute uses an internal firewall service. Since Networking includes a firewall service, you must disable the Compute firewall service by using the nova.virt.firewall.NoopFirewallDriver firewall driver.
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
password = placement
```
4.2.2) finalize installation  
Determine whether your computr node supports hardware acceleration for VM: 
```
[root@compute nova]# egrep -c '(vmx|svm)' /proc/cpuinfo
48
```

If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.  
If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.  
```
[libvirt]
# ...
virt_type = qemu

systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
systemctl status libvirtd.service openstack-nova-compute.service
```

### message5, Timed out waiting for nova-conductor
systemctl start libvirtd.service openstack-nova-compute.service hangs, /var/log/nova/nova-compute.log shows message:  
```
Timed out waiting for nova-conductor.  Is it running? Or did this service start before nova-conductor? 
```
Review config by: 
```
grep ^[^#[] nova.conf
```
Notice that there is a transport_url typo on master node, correct this one, restart compute servive on master node.     
Now compute service running fine on compute node. 

4.2.3) add the compute node to the cell database     
On controller node:   
```
. admin-openrc
openstack compute service list --service nova-compute
An unexpected error prevented the server from fulfilling your request. (HTTP 500) (Request-ID: req-19037f15-bd4b-4404-987a-d55eeb6cbe57)
```

### message6,  Too many connections
/var/log/nova/nova-scheduler.log shows:  
```
ERROR oslo_messaging.rpc.server [req-da6ea36d-a91d-465a-9307-336b7a77cff0 - - - - -] Exception during message handling: OperationalError: (pymysql.err.OperationalError) (1040, u'Too many connections') (Background on this error at: http://sqlalche.me/e/e3q8)
```

check how many connections currently 
```
MariaDB [(none)]> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 214   |
+-----------------+-------+
1 row in set (0.00 sec)
MariaDB [(none)]> show global status like 'Max_used_connections';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Max_used_connections | 215   |
+----------------------+-------+
1 row in set (0.00 sec)
```
1) Note that in /etc/my.cnf.d/openstack.cnf, we set: max_connections = 4096       
How come the actual max_connections is 214 here?   
In CentOS 7, There is limit for system resources in systemd. to change resource limit for a specific service:   
```
# find service unit 
systemctl list-unit-files | grep -E 'mysql|mariadb'
mariadb.service enabled
# change resource limit 
systemctl edit mariadb.service
[Service]
LimitNOFILE=4096
# restart service 
systemctl restart  mariadb.service

MariaDB [(none)]> show variables like 'open_files_limit';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| open_files_limit | 4096  |
+------------------+-------+
1 row in set (0.00 sec)

MariaDB [(none)]>  show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 3286  |
+-----------------+-------+
1 row in set (0.00 sec)
```

2) MySQL reserves a connection for super user, so the actual maximum connections is max_connections+1.     
3) after restart mariadb, we need to restart all service dependant on mariadb   
```
# identity 
systemctl restart httpd.service
# image 
systemctl restart openstack-glance-api.service 
systemctl restart openstack-glance-registry.service
# compute 
systemctl restart openstack-nova-api.service 
systemctl restart openstack-nova-scheduler.service 
systemctl restart openstack-nova-conductor.service 
systemctl restart openstack-nova-novncproxy.service
```

Now the expected result: 
```
[root@controller ~]# openstack compute service list --service nova-compute
+----+--------------+----------------------------+------+---------+-------+----------------------------+
| ID | Binary       | Host                       | Zone | Status  | State | Updated At                 |
+----+--------------+----------------------------+------+---------+-------+----------------------------+
| 19 | nova-compute | kos2001.bridgewatersys.com | nova | enabled | up    | 2019-02-06T16:13:55.000000 |
+----+--------------+----------------------------+------+---------+-------+----------------------------+
```
Discover compute hosts: 
```
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': bcb4f5f8-36e3-49e5-aa74-cf8023a90f36
Checking host mapping for compute host 'kos2001.bridgewatersys.com': cf34e6db-e180-457e-8f38-0fe6b11ff7e3
Creating host mapping for compute host 'kos2001.bridgewatersys.com': cf34e6db-e180-457e-8f38-0fe6b11ff7e3
Found 1 unmapped computes in cell: bcb4f5f8-36e3-49e5-aa74-cf8023a90f36
```

4.3) verify operation 
```
. admin-openrc
openstack compute service list
openstack catalog list    # list API endpoints in the identity service to verify connectivity with the identity service 
openstack image list
nova-status upgrade check  # check the cells and placement API are working successfully 
```
# 5, networking service    
 on controller node     
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

### 5.1) Configure networking options
choose option2, Self-service networks   
Install and configure the networking components on the controller node:  
```
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
```

#### 5.1.1) configure the server component  
```
cd /etc/neutron
cp neutron.conf neutron.conf.bak 
vi neutron.conf 
[database]
#...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
[DEFAULT]
#enable the Modular Layer 2 (ML2) plug-in, router service, and overlapping IP addresses:
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
[DEFAULT]
#configure RabbitMQ message queue access:
transport_url = rabbit://openstack:RABBIT_PASS@controller
[DEFAULT]
#...
auth_strategy = keystone

[keystone_authtoken]
#configure Identity service access
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
#...
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
#...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
[oslo_concurrency]
#...
lock_path = /var/lib/neutron/tmp
```

#### 5.1.2) Configure the Modular Layer 2 (ML2) plug-in
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

#### 5.1.3) Configure the Linux bridge agent
The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.  
```
cd  /etc/neutron/plugins/ml2
cp linuxbridge_agent.ini linuxbridge_agent.ini.bak 
vi linuxbridge_agent.ini
[linux_bridge]
#map the provider virtual network to the provider physical network interface
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
[vxlan]
#enable VXLAN overlay networks, configure the IP address of the physical network interface that handles overlay networks, and enable layer-2 population
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
[securitygroup]
# enable security groups and configure the Linux bridge iptables firewall driver   
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

#Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1. these values are not there until you modprobe br_netfilter 
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables
#To enable networking bridge support, typically the br_netfilter kernel module needs to be loaded
[root@controller etc]# modprobe br_netfilter
[root@controller etc]# sysctl -a > sysctl.log
[root@controller etc]# grep bridge-nf-call sysctl.log
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

#### 5.1.4) Configure the layer-3 agent
The Layer-3 (L3) agent provides routing and NAT services for self-service virtual networks.  
```
cd /etc/neutron/
cp l3_agent.ini  l3_agent.ini.bak 
vi l3_agent.ini  
[DEFAULT]
# onfigure the Linux bridge interface driver and external network bridge:
interface_driver = linuxbridge
```
#### 5.1.5) Configure the DHCP agent
The DHCP agent provides DHCP services for virtual networks.   
```
cd /etc/neutron/
cp dhcp_agent.ini dhcp_agent.ini.bak 
vi dhcp_agent.ini  
[DEFAULT]
#configure the Linux bridge interface driver, Dnsmasq DHCP driver, and enable isolated metadata so instances on provider networks can access metadata over the network
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

Now th choice of option2 - self-service networks is done. back to configure network on controller node 

### 5.2) Configure the metadata agent
The metadata agent provides configuration information such as credentials to instances.   
```
cd /etc/neutron/
cp metadata_agent.ini  metadata_agent.ini.bak 
vi metadata_agent.ini 
[DEFAULT]
# configure the metadata host and shared secret
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```
### 5.3) Configure the Compute service to use the Networking service 
```
vi /etc/nova/nova.conf
[neutron]
#configure access parameters, enable the metadata proxy, and configure the secret
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
``` 
### 5.4) Finalize installation
create soft link 
```
[root@controller nova]# ls -ltr /etc/neutron/plugin.ini
ls: cannot access /etc/neutron/plugin.ini: No such file or directory
[root@controller nova]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
[root@controller nova]# ls -ltr /etc/neutron/plugin.ini
lrwxrwxrwx 1 root root 37 Feb  6 22:37 /etc/neutron/plugin.ini -> /etc/neutron/plugins/ml2/ml2_conf.ini
```

Populate the database:
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
......
sqlalchemy.exc.ArgumentError: Could not parse rfc1738 URL from string ''
```
Add the missing DB connection in /etc/neutron/neutron.conf , retry fine   
Restart the Compute API service:  
```
systemctl restart openstack-nova-api.service
systemctl status openstack-nova-api.service
```

Start the Networking services and configure them to start when the system boots.For both networking options:
```
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service  neutron-metadata-agent.service
systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service  neutron-metadata-agent.service
systemctl status neutron-server.service 
systemctl status neutron-linuxbridge-agent.service 
systemctl status neutron-dhcp-agent.service  
systemctl status neutron-metadata-agent.service
```

For networking option 2, also enable and start the layer-3 service:
```
systemctl enable neutron-l3-agent.service
systemctl start neutron-l3-agent.service
systemctl status neutron-l3-agent.service
```

5.2) on compute node    
Install components 
```
yum install openstack-neutron-linuxbridge ebtables ipset
```

Configure the common component, which includes authentication mechanism, message queue, and plug-in
```
vi /etc/neutron/neutron.conf
#In the [database] section, comment out any connection options because compute nodes do not directly access the database.
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
[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

configure network options 
```
#Configure the Linux bridge agent
vi  /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

modprobe br_netfilter
sysctl -a > sysctl.log
grep bridge-nf-call sysctl.log
```

Configure the Compute service to use the Networking service
```
vi /etc/nova/nova.conf 
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

Finalize installation 
```
systemctl restart openstack-nova-compute.service
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
systemctl status neutron-linuxbridge-agent.service
```

5.3) verify 
```
. admin-openrc
openstack extension list --network
openstack network agent list
```

# 6, dashboard 
6.1) install & configure   
Install and configure dashboard on controller node, the only core service required by dashboard is identity service.   

6.1.1) install package 
```
yum install openstack-dashboard
```
6.1.2) config 
```
cd /etc/openstack-dashboard/
cp local_settings local_settings.bak 
vi local_settings 
#Configure the dashboard to use OpenStack services on the controller node:
OPENSTACK_HOST = "controller"
#Allow your hosts to access the dashboard, could be [‘*’] to accept all hosts
ALLOWED_HOSTS = ['one.example.com', 'two.example.com']
#Configure the memcached session storage service:
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
#Enable the Identity API version 3:
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
#Enable support for domains:
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
#Configure API versions:
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
#Configure Default as the default domain for users that you create via the dashboard:
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
#Configure user as the default role for users that you create via the dashboard:
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
#configure the time zone:
TIME_ZONE = "TIME_ZONE"

#Add the following line to /etc/httpd/conf.d/openstack-dashboard.conf if not included.
WSGIApplicationGroup %{GLOBAL}
```
6.1.3) finalize installation
```
systemctl restart httpd.service memcached.service
systemctl status httpd.service 
systemctl status  memcached.service

```
6.2) verify   
http://controller/dashboard   
Bad Request (400)     

change ALLOWED_HOSTS to '*', now login page shows up.    
domain = Default  
user=admin   
password=   

