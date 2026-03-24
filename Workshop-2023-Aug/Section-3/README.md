# MySQL Enterprise Backup
## Backup
Create backup directory
```
mkdir /home/opc/backup-dir
mkdir /home/opc/backup-file
```
Run full backup using directory
```
mysqlbackup --user=root --host=127.0.0.1 --port=3306 --backup-dir=/home/opc/backup-dir --with-timestamp backup-and-apply-log
```
Run full backup using single FILE
```
mysqlbackup --user=root --host=127.0.0.1 --port=3306 --backup-dir=/home/opc/backup-file --backup-image=/home/opc/backup-file/full.mbi --with-timestamp backup-to-image
```
Create database, table and record
```
mysql -uroot -h::1 -e "create database test; create table test.test (i int); insert into test.test values (1); select * from test.test"
```
Run incremental backup using directory
```
mysqlbackup --user=root --host=127.0.0.1 --incremental-backup-dir=/home/opc/backup-dir --incremental --with-timestamp --incremental-base=history:last_full_backup backup
```
Run incremental backup using file
```
mysqlbackup --user=root --host=127.0.0.1 --backup-dir=/home/opc/backup-file --backup-image=/home/opc/backup-file/incremental.mbi --incremental=optimistic --with-timestamp --incremental-base=history:last_full_backup backup-to-image
```
Validating a backup
```
mysqlbackup --backup-image=/home/opc/backup-file/full.mbi validate
```
## Full Restore
Create MySQL Instance 3307
```
mysqlsh -e "dba.deploySandboxInstance(3307)"

## do not set root password, leave it empty
```
Stop and delete database files
```
mysqlsh -e "dba.stopSandboxInstance(3307)"
rm -Rf mysql-sandboxes/3307/sandboxdata/*
```
Restore from full backup
```
mkdir /home/opc/dummy
mysqlbackup --defaults-file=mysql-sandboxes/3307/my.cnf --backup-image=/home/opc/backup-file/full.mbi --backup-dir=/home/opc/dummy copy-back-and-apply-log
rm -Rf /home/opc/dummy/*
```
Restore from incremental backup
```
mysqlbackup --defaults-file=mysql-sandboxes/3307/my.cnf --backup-image=/home/opc/backup-file/incremental.mbi --backup-dir=/home/opc/dummy copy-back-and-apply-log
```
Finalizing restoration
```
mv /home/opc/mysql-sandboxes/3307/sandboxdata/backup-mysqld-auto.cnf /home/opc/mysql-sandboxes/3307/sandboxdata/mysqld-auto.cnf
```
Stop and delete database instance 3306
```
mysqlsh -e "dba.stopSandboxInstance(3306)"

rm -Rf mysql-sandboxes/3306/sandboxdata/*
```
Start database instance 3307
```
mysqlsh -e "dba.startSandboxInstance(3307)"

## check variable values and latest data
mysql -uroot -h::1 -P3307 -e "show variables like '%invisible%'; show variables like '%buffer_pool%'; show variables like '%redo_log%'; select * from test.test"
```
