# replication
pacemaker+corosync+replication


1.Installing and Enabling the Pacemaker and Corosync Service

# sudo dnf config-manager --enable ol8_appstream ol8_baseos_latest ol8_addons
# sudo dnf install pcs pacemaker resource-agents fence-agents-all

check the version:
# crmadmin --version
# corosync -v

2. Enable FIREWALL

# sudo firewall-cmd --permanent --add-service=high-availability
# sudo firewall-cmd --add-service=high-availability

this step typically enables the following ports: 
TCP port 2224 (used by the pcs daemon), port 3121 (for Pacemaker Remote nodes), port 21064 (for DLM resources)
UDP ports 5405 (for Corosync clustering), and 5404 (for Corosync multicast, if configured)

OR alternately do this one by one:
# sudo firewall-cmd --add-port=2224/tcp
# sudo firewall-cmd --add-port=3121/tcp
# sudo firewall-cmd --add-port=5403/tcp
# sudo firewall-cmd --add-port=5404/udp
# sudo firewall-cmd --add-port=5405/udp
# sudo firewall-cmd --add-port=21064/tcp
# sudo firewall-cmd --add-port=9929/tcp
# sudo firewall-cmd --add-port=9929/udp

A simple Corosync/Pacemaker Cluster needs the following firewall settings:

TCP port 2224 for pcsd, Web UI and node-to-node communication.
TCP port 3121 if cluster has any Pacemaker Remote nodes.
TCP port 5403 for quorum device with corosync-qnetd.
UDP port 5404 for corosync if it is configured for multicast UDP.
UDP port 5405 for corosync.


4.DNS resolution must work.
Nodes must be reachable (firewall).
Nodes must allow traffic between them.

# sudo vi /etc/hosts
10.0.0.76 master
10.0.0.108 slave

# ping master
# ping slave

5. Setup the hacluster user with password.

# sudo passwd hacluster
Changing password for user hacluster.
New password: Nihao_123
Retype new password: 
passwd: all authentication tokens updated successfully.

6.Running the pcsd

# sudo systemctl enable --now pcsd.service

7.Creating the Cluster

Authenticate Nodes:
# sudo pcs host auth master slave -u hacluster
Password: Nihao_123
slave: Authorized
master: Authorized

OR alternately:
# sudo pcs host auth master addr=10.0.0.76 slave addr=10.0.0.108 -u hacluster

Create Cluster:
# sudo pcs cluster setup mysqlcluster master addr=10.0.0.76 slave addr=10.0.0.108
...
Cluster has been successfully set up.

Start Cluster:
# sudo pcs cluster start --all
master: Starting Cluster...
slave: Starting Cluster...
# sudo pcs cluster enable --all //enable these services to start at boot time 

OR alternately:
# sudo systemctl start pacemaker.service
# sudo systemctl enable pacemaker.service

OR can setup cluster and start immediatlly:
# sudo pcs cluster setup mysqlcluster --start master slave --force

During the setup cluser creates configuration file
# sudo cat /etc/corosync/corosync.conf

If somthing fails:
# sudo pcs cluster destroy

8.Set cluster parameters:

# sudo pcs property set stonith-enabled=false //Disable the fencing feature just for testing,Beacause we do NOT have shared data
# sudo pcs property set no-quorum-policy=ignore //Two-node cluster, disabling the no-quorum policy makes the most sense
# sudo pcs resource defaults update // Migration policy

8.Creating a Service and Testing Failover

Add a Resource, A resource is a service which is managed by the Cluster :
# sudo pcs resource create dummy_service ocf:pacemaker:Dummy op monitor interval=120s

View Resources:
# sudo pcs status
Cluster name: mysqlcluster
Status of pacemakerd: 'Pacemaker is running' (last updated 2023-04-25 09:02:34Z)
Cluster Summary:
  * Stack: corosync
  * Current DC: master (version 2.1.4-5.0.1.el8_7.2-dc6eb4362e) - partition with quorum
  * Last updated: Tue Apr 25 09:02:34 2023
  * Last change:  Tue Apr 25 09:00:10 2023 by root via cibadmin on master
  * 2 nodes configured
  * 1 resource instance configured

Node List:
  * Online: [ master slave ]

Full List of Resources:
  * dummy_service	(ocf::pacemaker:Dummy):	 Started master

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
  

Simulate service failure:
# sudo crm_resource --resource dummy_service --force-stop

View the Failed Actions message:
# sudo crm_mon

# sudo pcs resource status

Update resource
# sudo pcs resource update

Move Resource to other node and checke the status:
# sudo pcs resource move dummy_service slave
# sudo pcs status
...
...
Full List of Resources:
  * dummy_service	(ocf::pacemaker:Dummy):	 Started slave
  

9. Add VIP Resource:

# sudo pcs resource create Cluster_VIP ocf:heartbeat:IPaddr2 ip=10.0.0.100 cidr_netmask=24 op monitor interval=5s
# sudo pcs status
# sudo pcs status cluster
# sudo pcs resource move Cluster_VIP master
# sudo pcs status resources
  * dummy_service	(ocf::pacemaker:Dummy):	 Started master
  * Cluster_VIP	(ocf::heartbeat:IPaddr2):	 Started master
  

Reference For GRACEFUL MANUAL SWITCHOVER:

# sudo pcs cluster stop master
# sudo pcs cluster start master
# sudo pcs node standby master
# sudo pcs node unstandby master
# sudo pcs resource status
# pcs resource move VirtualIP slave
# pcs resource status

For Enable OCI VM's VIP service need to do following manual steps:
Please referto :
https://www.linkedin.com/pulse/automatic-virtual-ip-failover-oracle-cloud-demo-ahmed-mekawy 


10. Download and Install MySQL

Download:
wget https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm
Download TAR pakage: https://dev.mysql.com/downloads/mysql/
mysql-8.0.33-linux-glibc2.17-x86_64-minimal.tar.xz

sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum -y install mysql80-community-release-el8-1.noarch.rpm
sudo yum repolist all
sudo yum module disable mysql
sudo yum install mysql
sudo yum install mysql-shell

sudo groupadd mysqlgrp
sudo useradd -r -g mysqlgrp -s /bin/false mysqluser
sudo usermod -a -G mysqlgrp opc(Reopen the terminal)

sudo mkdir /mysql/ /mysql/etc /mysql/data /mysql/log /mysql/temp /mysql/binlog
echo 'PATH=/mysql/mysql-latest/bin:$PATH' >> /home/opc/.bashrc
echo 'export MYSQL_PS1="\\u on \\h>\\_"' >> /home/opc/.bashrc

cd /mysql/
sudo tar xvf /home/opc/mysql-8.0.33-linux-glibc2.17-x86_64-minimal.tar.xz
sudo ln -s mysql-8.0.33-linux-glibc2.17-x86_64-minimal/ mysql-latest

sudo vi etc/my.cnf
------------------------------------------------------
[mysqld]
# General configurations
port=3306
mysqlx_port=33060
server_id=10 //should defferent on every member
socket=/mysql/temp/mysql.sock
mysqlx_socket=/mysql/temp/mysqlx.sock

user=mysqluser

# File locations
basedir=/mysql/mysql-latest
plugin-dir=/mysql/mysql-latest/lib/plugin
datadir=/mysql/data
tmpdir=/mysql/temp
log-error=/mysql/log/err_log.log
log-bin=/mysql/binlog/binlog

# permit LOAD DATA LOCAL 
local_infile = ON

# Security setting for file load
secure-file-priv=/mysql/data

# InnoDB settings
innodb_buffer_pool_size=4G

# Enable GTID
gtid-mode=on
enforce-gtid-consistency=true


------------------------------------------------------

sudo chown -R mysqluser:mysqlgrp /mysql
sudo chmod -R 750 /mysql

sudo rm -rf /mysql/data/* /mysql/log/* /mysql/temp/* /mysql/binlog/*
sudo /mysql/mysql-latest/bin/mysqld --defaults-file=/mysql/etc/my.cnf --initialize-insecure --user=mysqluser
sudo /mysql/mysql-latest/bin/mysqld_safe --defaults-file=/mysql/etc/my.cnf --user=mysqluser &
mysql -uroot -h127.0.0.1 -P3306  -p
OR
mysqlsh root@localhost
\sql
SET PASSWORD='Welcome1';
CREATE USER 'admin'@'%' IDENTIFIED WITH mysql_native_password BY 'Welcome1';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;

11. Configure Master->Slave Replication

sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --zone=public --add-port=33060/tcp --permanent
sudo firewall-cmd --reload


On Master:
mysqlsh admin@master:3306
create user 'repuser'@'%' IDENTIFIED WITH mysql_native_password by 'Welcome1'; 
grant replication slave on *.* to 'repuser'@'%';

Creating a Data Snapshot On Master :
mysqladmin -uadmin -hmaster -P3306 shutdown -pWelcome1
cd /mysql
sudo rm -rf db.tar 
sudo tar cf /home/opc/db.tar data
sudo scp -i /home/opc/ssh.key /home/opc/db.tar opc@slave:/home/opc
sudo /mysql/mysql-latest/bin/mysqld_safe --defaults-file=/mysql/etc/my.cnf --user=mysqluser &

Extract db.tar on Slave:
mysqladmin -uadmin -hslave -P3306 shutdown -pWelcome1
cd /mysql
sudo rm -rf data
sudo tar -xvf /home/opc/db.tar 
sudo rm -rf data/auto.cnf
sudo /mysql/mysql-latest/bin/mysqld_safe --defaults-file=/mysql/etc/my.cnf --user=mysqluser &
mysqlsh admin@slave:3306 

CHANGE REPLICATION SOURCE TO 
SOURCE_HOST = 'master', 
SOURCE_PORT = 3306, 
SOURCE_USER = 'repuser', 
SOURCE_PASSWORD = 'Welcome1', 
SOURCE_AUTO_POSITION = 1;

START REPLICA;
SHOW REPLICA STATUS\G

------------------------------------------------------
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: master
                  Source_User: repuser
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: binlog.000004
          Read_Source_Log_Pos: 197
               Relay_Log_File: slave-relay-bin.000004
                Relay_Log_Pos: 367
        Relay_Source_Log_File: binlog.000004
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
          ......
          Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
          ......
------------------------------------------------------

Test Master->Slave Replication:

 # mysqlsh admin@master:3306
 # \sql
 # create schema master_schema;
 # show databases;
 +--------------------+
| Database           |
+--------------------+
| information_schema |
| master_schema      |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.0008 sec)


12. Configure Slave->Master Replication

On Master:

mysqlsh admin@master:3306
\sql
CHANGE REPLICATION SOURCE TO 
SOURCE_HOST = 'slave', 
SOURCE_PORT = 3306, 
SOURCE_USER = 'repuser', 
SOURCE_PASSWORD = 'Welcome1', 
SOURCE_AUTO_POSITION = 1;

START REPLICA;
SHOW REPLICA STATUS\G

------------------------------------------------------
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: slave
                  Source_User: repuser
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: binlog.000005
          Read_Source_Log_Pos: 157
               Relay_Log_File: master-relay-bin.000002
                Relay_Log_Pos: 367
        Relay_Source_Log_File: binlog.000005
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
          ...
          Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
          ...
 ------------------------------------------------------         
          
 Test Master<-Master Replication:
 
 # mysqlsh admin@slave:3306
 # \sql
 # create schema slave_schema;
 # CREATE TABLE slave_schema.slave_table(id int(3), title VARCHAR(100));
 # INSERT INTO slave_schema.slave_table (id, title) VALUES (001, 'Hello Pacemaker');
 # SELECT * from slave_schema.slave_table;
+----+-----------------+
| id | title           |
+----+-----------------+
|  1 | Hello Pacemaker |
+----+-----------------+
1 row in set (0.0006 sec)
 # show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| master_schema      |
| mysql              |
| performance_schema |
| slave_schema       |
| sys                |
+--------------------+
6 rows in set (0.0008 sec)


Install mysql.server automatically:
sudo cp /mysql/mysql_latest/bin/mysql.server /etc/init.d/mysql
chmod +x /etc/init.d/mysql
chkconfig --add mysql
chkconfig --level 345 mysql on


13. Options to Install MySQL Router with Master/Master Replication:

Install MySQL Router on your Client Server:

# sudo wget https://dev.mysql.com/get/Downloads/MySQL-Router/mysql-router-community-8.0.33-1.el8.x86_64.rpm
# sudo rpm -ivh mysql-router-community-8.0.33-1.el8.x86_64.rpm
# sudo vi /etc/mysqlrouter/mysqlrouter.conf 

Add following lines:
------------------------------------------------------ 
[routing:read_write]
bind_address = 127.0.0.1
bind_port = 7001
mode = read-write
destinations = 10.0.0.76:3306,10.0.0.108:3306
max_connections = 65535
max_connect_errors = 1000
client_connect_timeout = 9
routing_strategy=first-available

[routing:read_only]
bind_address = 127.0.0.1
bind_port = 7002
mode = read-only
destinations = 10.0.0.76:3306,10.0.0.108:3306
max_connections = 65535
max_connect_errors = 1000
client_connect_timeout = 9
routing_strategy=round-robin
------------------------------------------------------ 

# sudo systemctl enable mysqlrouter.service
# sudo systemctl start mysqlrouter

If you getting any errors:
# sudo tail -f /var/log/mysqlrouter/mysqlrouter.log
For check MySQL Router help options:
# mysqlrouter --help


sudo firewall-cmd --zone=public --add-port=7001/tcp --permanent
sudo firewall-cmd --zone=public --add-port=7002/tcp --permanent
sudo firewall-cmd --reload

# mysqlsh admin@localhost:7001 --sql --execute='select @@hostname;'
# mysqlsh admin@localhost:7002 --sql --execute='select @@hostname;'

Options to Add MySQL Router HA with Pacemaker on Client Server:
https://lefred.be/content/mysql-router-ha-with-pacemaker/

# sudo pcs resource list | grep router
service:mysqlrouter - systemd unit file for mysqlrouter
systemd:mysqlrouter - systemd unit file for mysqlrouter

# sudo pcs resource create mysqlrouter systemd:mysqlrouter clone
# sudo pcs constraint colocation add Cluster_VIP with mysqlrouter-clone score=INFINITY
# sudo pcs resource status


14. Options to Create ReplicaSet instead of Master/Master Replication(Only Asnyc Master/Replica Replication is supported)

mysqladmin -uadmin -hmaser -P3306 -pWelcome1 shutdown
mysqladmin -uadmin -hslave -P3306 -pWelcome1 shutdown

sudo rm -rf /mysql/data/* /mysql/log/* /mysql/temp/* /mysql/binlog/*
sudo /mysql/mysql-latest/bin/mysqld --defaults-file=/mysql/etc/my.cnf --initialize-insecure --user=mysqluser
sudo /mysql/mysql-latest/bin/mysqld_safe --defaults-file=/mysql/etc/my.cnf --user=mysqluser &
mysqlsh root@localhost:3306
SET PASSWORD='Welcome1';CREATE USER 'admin'@'%' IDENTIFIED BY 'Welcome1';GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;

mysqlsh admin@master:3306
dba.configureInstance("admin@master:3306")
dba.configureInstance("admin@slave:3306")

seoul = dba.createReplicaSet('seoul')
seoul.addInstance('admin@slave:3306')

Install MySQL Router:
sudo yum install mysql-router
sudo rm -rf /etc/mysqlrouter/*
sudo mysqlrouter --bootstrap admin@10.0.0.100:3306 --user mysqlrouter(/etc/mysqlrouter/mysqlrouter.conf)
sudo systemctl start mysqlrouter


During Master Fail, needed manul failover by following command:
mysqlsh admin@10.0.0.100:6447
seoul = dba.getReplicaSet()
seoul.forcePrimaryInstance()
