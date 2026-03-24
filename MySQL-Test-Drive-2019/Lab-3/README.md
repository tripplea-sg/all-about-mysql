# InnoDB Cluster
Disconnect and destroy our replicaset
```
mysqlsh gradmin:grpass@localhost:3306 -i -e "var x = dba.getReplicaSet(); x.disconnect()"
mysqlsh gradmin:grpass@localhost:3307 -i -e "var x = dba.getReplicaSet(); x.disconnect()"
mysqlsh gradmin:grpass@localhost:3308 -i -e "var x = dba.getReplicaSet(); x.disconnect()"
mysqlsh gradmin:grpass@localhost:3306 --sql -e "set global super_read_only=off; drop database mysql_innodb_cluster_metadata"
mysqlsh gradmin:grpass@localhost:3307 --sql -e "set global super_read_only=off; drop database mysql_innodb_cluster_metadata"
mysqlsh gradmin:grpass@localhost:3308 --sql -e "set global super_read_only=off; drop database mysql_innodb_cluster_metadata"
mysqlsh root@localhost:3307 --sql -e "stop replica; reset replica all"
mysqlsh root@localhost:3308 --sql -e "stop replica; reset replica all"
```
Create Cluster on 3306 and adding node 3307, 3308 into cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- dba createCluster 'myCluster'
mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@localhost:3307 --recoveryMethod=clone
mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@localhost:3308 --recoveryMethod=clone
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Setup router
```
/home/opc/router/stop.sh
mysqlrouter --bootstrap gradmin:grpass@localhost:3306 --directory router --force
/home/opc/router/start.sh
mysqlsh gradmin:grpass@localhost:3306 -i -e "var x =dba.getCluster(); x.listRouters()"
mysqlsh gradmin:grpass@localhost:6446 --sql -e "select @@port"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@port"
```
Show/set cluster options
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster options
mysqlsh gradmin:grpass@localhost:3306  --cluster -e "cluster.setInstanceOption('localhost:3306','memberWeight',70)"
mysqlsh gradmin:grpass@localhost:3306  --cluster -e "cluster.setInstanceOption('localhost:3308','memberWeight',20)"
mysqlsh gradmin:grpass@localhost:3306  --cluster -e "cluster.setInstanceOption('localhost:3307','memberWeight',80)"
```
Set Primary Instance to 3308
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster setPrimaryInstance gradmin:grpass@localhost:3308
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
mysqlsh gradmin:grpass@localhost:6446 --sql -e "select @@port"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@port"
mysqlsh root@localhost:3308 --sql -e "alter table dummy.test add constraint pk_1 primary key (i)"
mysqlsh gradmin:grpass@localhost:6446 --sql -e "insert into dummy.test values (10)"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select * from dummy.test"
mysqlsh gradmin:grpass@localhost:3306 --sql -e "select * from dummy.test"
mysqlsh gradmin:grpass@localhost:3307 --sql -e "select * from dummy.test"
mysqlsh gradmin:grpass@localhost:3308 --sql -e "select * from dummy.test"
```
How about our 3311 ?
```
mysqlsh root@localhost:3311 --sql -e "start replica for channel 'channel1'"
mysqlsh root@localhost:3311 --sql -e "select * from dummy.test"
mysqlsh root@localhost:3311 --sql -e "show replica status for channel 'channel1' \G"
mysqlsh root@localhost:3311 --sql -e "stop replica for channel 'channel1'"
mysqlsh root@localhost:3311 --sql -e "set sql_log_bin=0; create user gradmin@'%' identified by 'grpass'; grant insert on dummy.* to gradmin@'%'"
mysqlsh root@localhost:3311 --sql -e "start replica for channel 'channel1'"
mysqlsh root@localhost:3311 --sql -e "show replica status for channel 'channel1' \G"
mysqlsh root@localhost:3311 --sql -e "select * from dummy.test"
```
How about if cluster loss primmary node?
```
mysqladmin -uroot -h127.0.0.1 -P3308 shutdown
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
mysqlsh gradmin:grpass@localhost:6446 --sql -e "select @@port"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@port"
```
How about if 3308 comes up ?
```
mysqld_safe --defaults-file=$HOME/config/3308.cnf &
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Switch to Multi Primary mode and switch back to single primary mode
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster switchToMultiPrimaryMode
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
mysqlsh gradmin:grpass@localhost:3306 -- cluster switchToSinglePrimaryMode
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Set cluster consistency
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster setOption 'consistency' 'AFTER'
mysqlsh gradmin:grpass@localhost:3306 -- cluster setOption 'consistency' 'BEFORE'
mysqlsh gradmin:grpass@localhost:3306 -- cluster setOption 'consistency' 'BEFORE_AND_AFTER'
mysqlsh gradmin:grpass@localhost:3306 -- cluster setOption 'consistency' 'EVENTUAL'
```
How to recover from loss of quorum
```
ps -ef | grep mysqld | grep -v mysqld_safe.  # terminate 3307, 3308 
kill -9 <3307 PID> <3308 PID>
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
mysqlsh gradmin:grpass@localhost:3306 -- cluster forceQuorumUsingPartitionOf gradmin:grpass@localhost:3306
mysqld_safe --defaults-file=$HOME/config/3307.cnf &
mysqld_safe --defaults-file=$HOME/config/3308.cnf &
mysqlsh gradmin:grpass@localhost:3306 -- cluster rejoinInstance gradmin:grpass@localhost:3307
mysqlsh gradmin:grpass@localhost:3306 -- cluster rejoinInstance gradmin:grpass@localhost:3308
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
How to shutdown and startup cluster
```
mysqladmin -uroot -h127.0.0.1 -P3307 shutdown
mysqladmin -uroot -h127.0.0.1 -P3308 shutdown
mysqladmin -uroot -h127.0.0.1 -P3306 shutdown
mysqld_safe --defaults-file=$HOME/config/3306.cnf &
mysqld_safe --defaults-file=$HOME/config/3307.cnf &
mysqld_safe --defaults-file=$HOME/config/3308.cnf &
mysqlsh gradmin:grpass@localhost:3306 -- dba rebootClusterFromCompleteOutage

```
