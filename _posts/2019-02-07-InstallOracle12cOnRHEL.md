---
layout: post
title: Install Oracle 12c on RHEL7 
category: Oracle 
tags: [Oracle]
---

#### 1) Download the Oracle Database 12c installer package
linuxx64_12201_database.zip

#### 2) set swap -  16GB .
```
# cat /etc/redhat-release
Red Hat Enterprise Linux Server release 7.3 (Maipo)
cat /proc/swaps. or. Raw. # swapon -s

fallocate -l 16G /swapfile
dd if=/dev/zero of=/swapfile bs=1024 count=16777216  # this way takes longer time  
chmod 600 /swapfile
mkswap /swapfile
vim /etc/fstab
/swapfile          swap            swap    defaults        0 0
swapon 
``` 

#### 3) Enable X11 Forwarding to “yes” in sshd configuration file.
```
#on my laptop 
“Tick” the Enable X11 forwarding in PUTTY.
Launch the “Xming” application from your local machine.

#on server node, as root 
touch /root/.Xauthority
uncomment the X11Forwarding and set to “yes” in /etc/ssh/sshd_config file.
xauth list $DISPLAY
HOSTNAME/unix:10  MIT-MAGIC-COOKIE-1  b25654b722770f3465b276a45445ecb7
echo $DISPLAY
export DISPLAY= localhost:10.0  
xclock 

#as oracle user 
touch .Xauthority
xauth list $DISPLAY 
echo $DISPLAY
xauth add localhost.localdomain/unix:10  MIT-MAGIC-COOKIE-1  b25654b722770f3465b276a45445ecb7
export DISPLAY=localhost:10.0
xclock
PuTTY X11 proxy: Unsupported authorisation protocol
Error: Can't open display: localhost:10.0
# relogin as user directly, try xclock, works this time 
```

#### 4) Properly set the hostname.
```
cat /etc/hosts 
cat /etc/hostname 
```
#### 5) Set the Kernel Parameters.
```
vi /etc/sysctl.conf 
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
sysctl -p
sysctl -a
```

#### 6) Set the ulimit values.
```
vi /etc/security/limits.conf 
oracle   soft   nproc    131072
oracle   hard   nproc    131072
oracle   soft   nofile   131072
oracle   hard   nofile   131072
oracle   soft   core     unlimited
oracle   hard   core     unlimited
oracle   soft   memlock  50000000
oracle   hard   memlock  50000000
```

#### 7) Install the required rpm packages.
```
yum install  -y binutils  compat-libstdc++-33  compat-libstdc++-33.i686  gcc   gcc-c++  glibc   glibc.i686  glibc-devel   glibc-devel.i686   ksh   libgcc libgcc.i686  libstdc++  libstdc++.i686  libstdc++-devel  libstdc++-devel.i686  libaio   libaio.i686  libaio-devel  libaio-devel.i686   libXext   libXext.i686   libXtst  libXtst.i686   libX11 libX11.i686   libXau libXau.i686  libxcb  libxcb.i686  libXi  libXi.i686 make sysstat  unixODBC  unixODBC-devel  zlib-devel 
yum install compat-libcap1-1.10 
yum install smartmontools-6.2-4  # not available 
yum install smartmontools installed smartmontools.x86_64 1:6.2-7.el7
```

#### 8) create user and groups & folders
```
useradd oracle
passwd oracle
groupadd oinstall
usermod -G oinstall oracle 
 
mkdir /oracle
chown -R oracle:oinstall /oracle
chmod -R 775 /oracle
chmod g+s /oracle
```
#### 9) run installer 
```
./runInstaller
# connect to database 
vi /etc/oratab 
vi .bash_profile 
export ORACLE_UNQNAME=orcl
export ORACLE_BASE=/oracle/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.2.0/dbhome_1
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH

SQL> select name, cdb,con_id from v$database;
NAME      CDB     CON_ID
--------- --- ----------
ORCL      YES          0
```