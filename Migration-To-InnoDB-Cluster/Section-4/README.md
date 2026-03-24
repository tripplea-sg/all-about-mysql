# Restore MEB to Create New Instance
## create environment 5306
Create directory
```
mkdir -p /home/opc/db/3306/data /home/opc/db/5306/innodb_data_home_dir /home/opc/db/5306/innodb_undo_directory /home/opc/db/5306/innodb_temp_tablespace_dir /home/opc/db/5306/innodb_temp_data_file_path /home/opc/db/5306/innodb_log_group_home_dir /home/opc/db/5306/log_bin
```
Create my.cnf
```
vi /home/opc/db/5306/my.cnf
```
Copy the following content
```
[mysqld]
datadir=/home/opc/db/5306/data
binlog-format=ROW
log-bin=/home/opc/db/5306/log_bin/bin
innodb_data_home_dir=/home/opc/db/5306/innodb_data_home_dir
innodb_undo_directory=/home/opc/db/5306/innodb_undo_directory
innodb_temp_tablespaces_dir=/home/opc/db/5306/innodb_temp_tablespace_dir 
innodb_temp_data_file_path=/home/opc/db/5306/innodb_temp_data_file_path/ibtmp1:12M:autoextend
innodb_log_group_home_dir=/home/opc/db/5306/innodb_log_group_home_dir
port=5306
server_id=100
socket=/home/opc/db/5306/data/mysqld.sock
log-error=/home/opc/db/5306/data/mysqld.log
enforce_gtid_consistency = ON
gtid_mode = ON
log_slave_updates = ON
innodb_buffer_pool_size=1G
innodb_buffer_pool_instances=1
innodb_log_file_size=1G
innodb_log_files_in_group=3
innodb_flush_log_at_trx_commit=1
```
Restore full backup 
```
rm -Rf /home/opc/meb/image/tmp/*
mysqlbackup --defaults-file=/home/opc/db/5306/my.cnf --backup-dir=/home/opc/meb/tmp --backup-image=/home/opc/meb/image/full.mbi copy-back-and-apply-log
```
Restore Incremental Backup
```
rm -Rf /home/opc/meb/image/tmp/*
mysqlbackup --defaults-file=/home/opc/db/5306/my.cnf --backup-dir=/home/opc/meb/image/tmp --backup-image=/home/opc/meb/image/incremental.mbi --incremental copy-back-and-apply-log
```
Starting up 5306
```
mysqld_safe --defaults-file=/home/opc/db/5306/my.cnf &
```
Check data
```
mysql -uroot -h::1 -P5306 -e "show databases"

mysql -uroot -h::1 -P5306 -e "select * from dev.dev"
```
