# MySQL High Availability
## InnoDB Cluster
Create Instances
```
mysqlsh -e "dba.deploySandboxInstance(3306)"
mysqlsh -e "dba.deploySandboxInstance(3307)"
mysqlsh -e "dba.deploySandboxInstance(3308)"

mysql -uroot -h::1 -e "set persist sql_generate_invisible_primary_key=on;"
mysql -uroot -h::1 -P3307 -e "set persist sql_generate_invisible_primary_key=on;"
mysql -uroot -h::1 -P3308 -e "set persist sql_generate_invisible_primary_key=on;"
```
Install world_x on 3306
```
mysql -uroot -h::1 < world_x-db/world_x.sql

mysql -uroot -h::1 -e "show databases"
```
Configure instance
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3306 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3307 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3308 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
```
Create InnoDB Cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- dba create-cluster mycluster
```
Tuning InnoDB Cluster
```
mysql -uroot -h::1

set persist group_replication_paxos_single_leader=on;
set persist_only group_replication_compression_threshold=100;
set persist group_replication_poll_spin_loops=100;

restart;
exit
```
Start InnoDB Cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- dba rebootClusterFromCompleteOutage

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Add instances
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@localhost:3307 --recoveryMethod=incremental

mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@localhost:3308 --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Fine Tune instance 3307
```
mysql -uroot -h::1 -P3307

set persist group_replication_poll_spin_loops=100;
set persist_only group_replication_compression_threshold=100;

restart.
exit;
```
Fine Tune instance 3308
```
mysql -uroot -h::1 -P3308

set persist group_replication_poll_spin_loops=100;
set persist_only group_replication_compression_threshold=100;

restart.
exit;
```
Check InnoDB Cluster Status
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
## Install and Configure MySQL Router
Install MySQL Router
```
unzip MySQL-router-8.1.zip

sudo yum localinstall -y mysql-router-commercial-8.1.0-1.1.el8.x86_64.rpm
```
Configure MySQL Router
```
mysqlrouter --bootstrap gradmin:grpass@localhost:3306 --directory router
```
Start MySQL Router
```
router/start.sh
```
Connecting to PRIMARY through MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6446 -e "select @@port"
```
Connecting to SECONDARY through MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"

mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
```
## InnoDB ClusterSet
Create instance 5506
```
mysqlsh -e "dba.deploySandboxInstance(5506)"
```
Configure instance
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=5506 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
```
Create clusterset and add a replica cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster createClusterSet mycs

mysqlsh gradmin:grpass@localhost:3306 -- clusterset status

mysqlsh gradmin:grpass@localhost:3306 -- clusterset createReplicaCluster gradmin:grpass@localhost:5506 myrep --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:3306 -- clusterset status

mysqlsh gradmin:grpass@localhost:5506 -- cluster status
```
Fine Tune Replica Cluster
```
mysql -uroot -h::1 -P5506

set persist group_replication_paxos_single_leader=on;
set persist group_replication_poll_spin_loops=100;
set persist_only group_replication_compression_threshold=100;

restart;
exit;
```
Start REPLICA cluster 
```
mysqlsh gradmin:grpass@localhost:5506 -- dba rebootClusterFromCompleteOutage

mysqlsh gradmin:grpass@localhost:5506 -- cluster status
```
Stop MySQL Router
```
router/stop.sh

rm -Rf router
```
Configure MySQL Router
```
mysqlrouter --bootstrap gradmin:grpass@localhost:3306 --directory router --force
```
Start MySQL Router
```
router/start.sh
```
## InnoDB Cluster Read Replica
Create instance 2206
```
mysqlsh -e "dba.deploySandboxInstance(2206)"
```
Configure instance
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=2206 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
```
Adding node as read replica
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster addReplicaInstance gradmin:grpass@localhost:2206 --recoveryMethod=clone
```
Check cluster status
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Change replication sources from PRIMARY to SECONDARY
```
mysqlsh gradmin:grpass@localhost:3306 --cluster -e "cluster.setInstanceOption('localhost:2206','replicationSources','secondary')"

mysqlsh gradmin:grpass@localhost:3306 -- cluster rejoinInstance localhost:2206
```
Check cluster status
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Connecting to PRIMARY through MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6446 -e "select @@port"
```
Connecting to SECONDARY through MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"

mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
```
Change MySQL Router default R/O to Read Replica
```
mysqlsh gradmin:grpass@localhost:3306 --cluster -e "cluster.getClusterSet().setRoutingOption('read_only_targets','read_replicas')"
```
Now, test connection to MySQL Router port 6447
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
```
