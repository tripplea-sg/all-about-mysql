# MySQL Enterprise Backup
## Create backup administrator
The mysqlbackup command connects to the MySQL server using the credentials supplied with the --user and --password options. 
The specified user needs certain privileges. 
```
CREATE USER 'mysqlbackup'@'%.test.oraclevcn.com' IDENTIFIED BY 'backup';
GRANT SELECT, BACKUP_ADMIN, RELOAD, PROCESS, SUPER, REPLICATION CLIENT ON *.* TO 'mysqlbackup'@'%.test.oraclevcn.com';
GRANT CREATE, INSERT, DROP, UPDATE ON mysql.backup_progress TO 'mysqlbackup'@'%.test.oraclevcn.com'; 
GRANT CREATE, INSERT, DROP, UPDATE, SELECT, ALTER ON mysql.backup_history TO 'mysqlbackup'@'%.test.oraclevcn.com';

CREATE USER 'mysqlbackup'@'localhost' IDENTIFIED BY 'backup';
GRANT SELECT, BACKUP_ADMIN, RELOAD, PROCESS, SUPER, REPLICATION CLIENT ON *.* TO 'mysqlbackup'@'localhost';
GRANT CREATE, INSERT, DROP, UPDATE ON mysql.backup_progress TO 'mysqlbackup'@'localhost'; 
GRANT CREATE, INSERT, DROP, UPDATE, SELECT, ALTER ON mysql.backup_history TO 'mysqlbackup'@'localhost';
```
For creating a backup to tape using SBT API
```
grant CREATE, INSERT, DROP, UPDATE on mysql.backup_sbt_history TO 'mysqlbackup'@'%.test.oraclevcn.com'; 

grant CREATE, INSERT, DROP, UPDATE on mysql.backup_sbt_history TO 'mysqlbackup'@'localhost'; 
```
For transportable tablespace and backup non-innodb tables
```
GRANT LOCK TABLES, CREATE, DROP, FILE, INSERT, ALTER ON *.* TO 'mysqlbackup'@'%.test.oraclevcn.com';

GRANT LOCK TABLES, CREATE, DROP, FILE, INSERT, ALTER ON *.* TO 'mysqlbackup'@'localhost';
```
For using redo log archiving
```
GRANT INNODB_REDO_LOG_ARCHIVE ON *.* TO 'mysqlbackup'@'%.test.oraclevcn.com';

GRANT INNODB_REDO_LOG_ARCHIVE ON *.* TO 'mysqlbackup'@'localhost';
```
For using encrypted tablespace
```
GRANT ENCRYPTION_KEY_ADMIN ON *.* TO 'mysqlbackup'@'%.test.oraclevcn.com';

GRANT ENCRYPTION_KEY_ADMIN ON *.* TO 'mysqlbackup'@'localhost';
```
For working with InnoDB Cluster or Group Replication
```
GRANT SELECT ON performance_schema.replication_group_members TO 'mysqlbackup'@'%.test.oraclevcn.com';

GRANT SELECT ON performance_schema.replication_group_members TO 'mysqlbackup'@'localhost';

exit;
```
## Prepare Backup Directory
Prepare directory for backup
```
mkdir -p /home/opc/meb/directory
```
Prepare directory for Single File Backup
```
mkdir -p /home/opc/meb/image
```
## Run Full Backup from secondary node
```
mysqlbackup -umysqlbackup -pbackup --host=127.0.0.1 --backup-dir=/home/opc/meb/directory --with-timestamp backup-and-apply-log 
```
Run Single File Full Backup from secondary node
```
mysqlbackup -umysqlbackup -pbackup --host=127.0.0.1 --backup-dir=/home/opc/meb/image --backup-image=/home/opc/meb/image/full.mbi --with-timestamp backup-to-image
```
## Check backup history
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select * from mysql.backup_history;"

mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select * from mysql.backup_progress;"
```
## Validating a Backup
Validate backup image
```
mysqlbackup --backup-image=/home/opc/meb/image/full.mbi validate
```
validate backup directory
```
mysqlbackup --backup-dir=/home/opc/meb/directory/<your backup directory> validate
```
## Incrememtal Backup
Create new transactions
```
mysql -umysqlbackup -pbackup -h127.0.0.1 -P6446 -e "create database dev"

mysql -umysqlbackup -pbackup -h127.0.0.1 -P6446 -e "create table dev.dev (i int primary key)"

mysql -umysqlbackup -pbackup -h127.0.0.1 -P6446 -e "insert into dev.dev values (1), (2), (3)"
```
Backup incremental using directory backup
```
mysqlbackup -umysqlbackup -pbackup --host=127.0.0.1 --incremental-backup-dir=/home/opc/meb/directory --incremental --with-timestamp --incremental-base=history:last_full_backup backup
```
Backup incremental using single file backup
```
mysqlbackup -umysqlbackup -pbackup --host=127.0.0.1 --backup-dir=/home/opc/meb/image --backup-image=/home/opc/meb/image/incremental.mbi --incremental=optimistic --with-timestamp --incremental-base=history:last_full_backup backup-to-image
```
## Table-Level-Recovery (TLR)
Backup DDLonly
```
mkdir -p /home/opc/backup
rm -Rf /home/opc/backup/*
mysqlsh root@localhost:3307 -- util dumpInstance '/home/opc/backup' --ddlOnly=true
```

Drop table nation.guests
```
mysql -uroot -h::1 -P3307 -e "drop table nation.guests"
```
Start restore metadata first
```
mysql -uroot -h127.0.0.1 -P3307 -e "set global local_infile=on"
mysqlsh -uroot -h127.0.0.1 -P3307 -- util loadDump '/home/opc/backup' --ignoreVersion=true --loadData=false --createInvisiblePKs=true --include-tables='nation.guests'
```
Remove 3306 and 3308 from Cluster
```
mysqlsh gradmin:grpass@localhost:3307 -- cluster remove-instance localhost:3308
mysqlsh gradmin:grpass@localhost:3307 -- cluster remove-instance localhost:3306
```
Restoring the table from full backup into 3307 
```
mkdir -p /home/opc/meb/image/tmp
mysqlbackup --user=mysqlbackup --password=backup --host=127.0.0.1 --port=3307 --include-tables="^nation\.guests" --backup-dir=/home/opc/meb/image/tmp --backup-image=/home/opc/meb/image/full.mbi copy-back-and-apply-log
```
Restoring table from full backup into 3308 and add this instance back to the cluster
```
rm -Rf /home/opc/meb/image/tmp/*
mysql -uroot -h::1 -P3308 -e "set global super_read_only=off"
mysqlbackup --user=mysqlbackup --password=backup --host=127.0.0.1 --port=3308 --include-tables="^nation\.guests" --backup-dir=/home/opc/meb/image/tmp --backup-image=/home/opc/meb/image/full.mbi copy-back-and-apply-log

mysqlsh gradmin:grpass@localhost:3307 -- cluster add-instance localhost:3308 --recoveryMethod=incremental
mysqlsh gradmin:grpass@localhost:3307 -- cluster status
```
Restoring table from full backup into 3306 and add this instance back to the cluster
```
rm -Rf /home/opc/meb/image/tmp/*
mysql -uroot -h::1 -P3306 -e "set global super_read_only=off"
mysqlbackup --user=mysqlbackup --password=backup --host=127.0.0.1 --port=3306 --include-tables="^nation\.guests" --backup-dir=/home/opc/meb/image/tmp --backup-image=/home/opc/meb/image/full.mbi copy-back-and-apply-log

mysqlsh gradmin:grpass@localhost:3307 -- cluster add-instance localhost:3306 --recoveryMethod=incremental
mysqlsh gradmin:grpass@localhost:3307 -- cluster status
```
check connection from MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6446 -e "select @@port"
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
```
Check tables on 3306, 3307, 3308
```
mysql -uroot -h::1 -P3306 -e "select * from nation.guests"
mysql -uroot -h::1 -P3307 -e "select * from nation.guests"
mysql -uroot -h::1 -P3308 -e "select * from nation.guests"
```
