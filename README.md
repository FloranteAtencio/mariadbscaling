# Home Lab Environment

Final Plan: 3-Node Server with K3s, Containerized MariaDB (Master-Slave Replication), and ProxySQL (proxmox and mariadb workbench are optional) 

##### Goal: 
A highly available and scalable MariaDB deployment with a dedicated master, two slaves, and ProxySQL for database load balancing, all containerized and orchestrated by K3s.

## Components:

#### Hardware and Application:

* Three physical server nodes (Nodes 1, 2, and 3).
* Linux Operating System (e.g., Ubuntu Server, CentOS Stream)
* K3s (lightweight Kubernetes)
* Containerd (or other container runtime, managed by K3s)
* Rancher VE Cluster
* Longhorn (Shared Storage)
* KeepAlive (HA and Automation network level)

### Containerized Applications:

* MariaDB Master: A container running MariaDB, configured as the master server in the replication setup.
* MariaDB Slaves: Two Container  running MariaDB, configured as slaves replicating from the master with uniques configuration.
* ProxySQL: A container running ProxySQL to provide load balancing and query routing for the MariaDB cluster.

### Key Points/Confirmations:

* K3s Cluster: You'll have a three-node K3s cluster, providing high availability for the control plane and the ability to schedule your application pods across the nodes.
* Master-Slave Replication: MariaDB statefulset will use asynchronous master-slave replication for read scalability and basic HA.
* ProxySQL: A great option for load balancing, query routing, and connection pooling.
* Rancher: Native VE K3s for monitoring.
* longhorn: for persinstent shared storage
*  KeepAlive: HA and Automation network level

### Containerized Everything: 
All the components (MariaDB and ProxySQL) will be running in containerd managed by K3s.

### Manual ProxySQL Configuration: NOTE
You understand that you will be manually setting up and managing ProxySQL to manage traffic and to scale resources across your databases

## Detailed Steps:

### Prepare the Servers:

* Install Linux on each server.
* Configure networking (static IP addresses, DNS).
* Harden the operating system (firewall, security updates).
  
### Install K3s:

* Install K3s on each server.
* Setup master node 
* configure the working node get the token and ip of master node.
* Define Kubernetes Deployments and Services:
* Create Kubernetes Deployment manifests for the MariaDB master, MariaDB slaves, and ProxySQL.
* Create a Kubernetes Service manifest for ProxySQL, exposing it to the outside world.

### Configure MariaDB Replication:
* Create User the has most of the privelged each of database instance.
* Create a replication user on the MariaDB master server.
* Configure the MariaDB slave servers to connect to the master server and start replication.

### Configure ProxySQL:

* Configure ProxySQL to connect to the MariaDB master and slave servers.
* Configure query routing rules to send read queries to the slaves and write queries to the master.

### Test the Setup:

* Test the replication setup by writing data to the master server and verifying that it is replicated to the slave servers.
* Test the load balancing by sending traffic to ProxySQL and verifying that it is distributed across the MariaDB instances.
* Test the failover mechanism by simulating a failure of the master server and verifying that one of the slaves is automatically promoted to become the new master.
  
### Important Considerations:

* Persistent Storage: Ensure that the MariaDB data directory is stored on a persistent volume that survives container restarts. Use Kubernetes PersistentVolumeClaims (PVCs) to achieve this.
* Networking: Configure networking properly to allow MariaDB clients to connect to the database server.
* Security: Secure the K3s cluster, MariaDB instances, and ProxySQL.
* Monitoring: Implement monitoring to track the performance of the K3s cluster, MariaDB instances, and ProxySQL.
* Automated Failover: Implement an automated failover mechanism to promote a slave to master in case the main master node fails.
* Resource Allocation: Each server can run at most, a single master. Make sure that resources are not strained and the servers are all healthy

### Next Steps:

* Install Rancher setup High avalabity, failover and etc
* Install longhorn for shared storage to avoid single point of failure

## Sample Instance : Database and Table for Master Slave
```SQL
 CREATE DATABASE football;
 USE football;
 CREATE TABLE players (name varchar(50) DEFAULT NULL,position varchar(50) DEFAULT NULL);
 INSERT INTO players VALUES ('Lionel Messi','Forward');
```
## Create user : replication user for Master Slave 
##### Note: replication user is exclusive only for master instance
```SQL
 GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY '123'; #enter password
 FLUSH PRIVILEGES;
 show master status;
```
## Set up :  Do this for all the rest of salve database  instance
```SQL
STOP SLAVE;
RESET SLAVE ALL;
 CHANGE MASTER TO 
   MASTER_HOST='10.89.0.6', -- local ip addres  of master database 
   MASTER_USER='slave_user', -- previous user create 
   MASTER_PASSWORD='123',
   MASTER_LOG_FILE='master-bin.000004', -- this will appear in the  master database using Show master staus
   master_use_gtid=slave_pos, -- set the  GTID in slave_POS
   MASTER_LOG_POS=343; -- same goes here 
START SLAVE;
show slave status \G;
```
### Create user : master and slave 
```SQL
CREATE USER 'maxscale'@'%' IDENTIFIED BY 'maxscale';
GRANT SUPER, REPLICA MONITOR, REPLICATION CLIENT, REPLICATION SLAVE, SHOW DATABASES, EVENT, PROCESS, SLAVE MONITOR, READ_ONLY ADMIN ON *.* TO 'maxscale'@'%';
GRANT SELECT ON mysql.* TO 'maxscale'@'%';
FLUSH PRIVILEGES;
```

### Setup : Copy the following

```SQL
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight)<br/> 
VALUES (10, '10.89.0.6', 3306, 100), -- offline node
       (20, '10.89.0.6', 3306, 100), -- write only
       (40, '10.89.0.7', 3306, 100), -- back up write
       (30, '10.89.0.7', 3306, 100), -- read only
       (30, '10.89.0.8', 3306, 100); -- read only

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply)
VALUES (4, 1, "^SELECT.*", 30, 1),
       (1, 1, ".*", 10, 1),
       (2, 1, ".*", 20, 1),
       (3, 1, ".*", 40, 1);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;

INSERT INTO mysql_users (username, password, default_hostgroup)
VALUES ('maxscale', 'maxscale', 10);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;

UPDATE global_variables SET variable_value='maxscale'
WHERE variable_name='mysql-monitor_username';
UPDATE global_variables SET variable_value='maxscale'
WHERE variable_name='mysql-monitor_password';

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

INSERT INTO mysql_group_replication_hostgroups
(writer_hostgroup, -- The hostgroup to which all write traffic should be sent by default

backup_writer_hostgroup, -- The hostgroup for additional writers if the number of writers exceeds

reader_hostgroup, -- he hostgroup to which read traffic should be sent

offline_hostgroup, --This ensures that any traffic is redirected away from it until it comes back online and is ready to handle operations again. The setup helps maintain high availability and reliability in your replication environment.

active, -- A flag indicating whether the hostgroup is active (1) or not (0)

max_writers, -- The maximum number of writers allowed in the writer_hostgroup

writer_is_also_reader, -- indicates whether the writer can also serve read requests (0 = no, 1 = yes)

max_transactions_behind -- The maximum number of transactions that can be behind before a hostgroup is considered out of sync
)
VALUES (20, -- hostgroup
        40, -- hostgroup
        30, -- hostgroup
        10, -- hostgroup
         1, -- (1) or (0)
         2, -- number of availabity
         0, -- (1) or (0)
         100);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```
#

# Debugging
`podman logs --tail 1000 mariadb-master`

podman pull docker.io/library/mariadb:latest
podman pull docker.io/proxysql/proxysql

relay_master_log_file:master-bin.000005 
exec_master_log_pos_343

SELECT BINLOG_GTID_POS('master-bin.000002', '1930');

SET GLOBAL gtid_slave_pos='0-1-9';

SHOW VARIABLES LIKE 'log_bin';

DROP USER 'maxscale'@'%';

SELECT user, host FROM mysql.user;

SHOW GRANTS FOR 'slave_user'@'%';


SELECT @@gtid_current_pos;
SELECT @@gtid_slave_pos;

SELECT variable_name, variable_value 
FROM global_variables 
WHERE variable_name LIKE 'mysql-monitor_%';

podman inspect mariadb-slave | grep "IPAddress"

podman stop $(podman ps -q)
podman rm $(podman ps -aq)
podman volume prune -f
podman network prune -f
