# Installation, Setup and Configuration
## MySQL Server and MySQL Shell Installation
Extract MySQL Server binary
```
cd installer
unzip MySQL-Server-8.4.0.zip
```
Install MySQL Server
```
sudo yum localinstall -y mysql-commercial-*
```
Extract MySQL Shell
```
unzip MySQL-Shell-8.4.0.zip
```
Install MySQL Shell
```
sudo yum localinstall -y mysql-shell-commercial-*.rpm
```
Clean up
```
rm $HOME/installer/*.rpm
```
## OS Parameter Tuning
Edit /etc/fstab using "sudo vi /etc/fstab" and change 
```
/dev/mapper/ocivolume-root /                       xfs     defaults        0 0
```
to
```
/dev/mapper/ocivolume-root /                       xfs     defaults,noatime,nodiratime     0 0
```
Reboot the VM.
```
sudo reboot
```
Fine tuning Linux Kernel
```
sudo vi /etc/sysctl.conf


fs.file-max=3248957
fs.aio-max-nr=1048576
net.core.netdev_max_backlog=8000
net.core.somaxconn=8000
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.ip_local_port_range=9000  65000
net.netfilter.nf_conntrack_tcp_timeout_time_wait=10
vm.swappiness=1
vm.dirty_ratio=10
vm.dirty_background_ratio=5
vm.dirty_expire_centisecs=500
vm.dirty_writeback_centisecs=100

net.core.rmem_default=262144
net.core.rmem_max=16777216
net.core.wmem_default=262144
net.core.wmem_max=16777216
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_wmem=4096 65536 16777216
net.ipv4.tcp_syn_retries=0
net.ipv4.tcp_synack_retries=0
net.ipv4.tcp_keepalive_time=30
net.ipv4.tcp_keepalive_intvl=1
net.ipv4.tcp_keepalive_probes=2

kernel.shmmax=30923764531
kernel.shmall=7549748
kernel.sem=32000 1024000000 500 32000
kernel.msgmax=65535
kernel.msgmnb=65535

```
Apply parameters
```
sudo sysctl -p
```
Check IO Scheduler
```
cat /sys/block/sda/queue/scheduler
```
Change IO Scheduler to none
```
sudo su
echo none > /sys/block/sda/queue/scheduler
exit
cat /sys/block/sda/queue/scheduler
```
Disable SELinux
```
sudo setenforce 0
```
## Setup and Configure Instance 3306
Create directories:
```
sudo mkdir /redo /binlog
sudo chmod ugo+xwr /redo
sudo chmod ugo+xwr /binlog
```
edit my.cnf
```
sudo vi /etc/my.cnf

innodb_log_group_home_dir=/redo
log-bin=/binlog/bin
```
Rebuid MySQL
```

sudo systemctl stop mysqld
sudo su
rm -Rf /var/lib/mysql/* /redo/* /binlog/*
ls /var/lib/mysql/
exit

sudo systemctl start mysqld
```
Check MySQL Temp Password
```
sudo cat /var/log/mysqld.log | grep password
```
Configure MySQL Instance 3306
```
mysql -uroot -h::1 -p

alter user root@'localhost' identified by 'R00t_123';

set persist_only innodb_redo_log_capacity=2147483648;
set persist_only innodb_flush_neighbors=2;
set persist_only innodb_io_capacity=3000;
set persist_only innodb_io_capacity_max=3000;
set persist_only innodb_buffer_pool_size=25769803776;
set persist_only innodb_buffer_pool_instances=12;
set persist_only innodb_lru_scan_depth=250;
set persist_only innodb_page_cleaners=12;
set persist_only innodb_checksum_algorithm=strict_crc32;
set persist_only binlog_row_image=MINIMAL;

set persist_only innodb_thread_sleep_delay=500; 
set persist_only innodb_spin_wait_delay=2;
set persist_only innodb_spin_wait_pause_multiplier=10;

set persist_only sql_generate_invisible_primary_key=on;

set persist_only innodb_use_fdatasync=on;
set persist_only innodb_numa_interleave=on;

set persist_only read_buffer_size=131072;
set persist_only read_rnd_buffer_size=262144;
set persist_only sort_buffer_size=8388608;
set persist_only join_buffer_size=16777216;
set persist_only binlog_cache_size=16777216;
set persist_only binlog_stmt_cache_size=8192;
set persist_only thread_stack=1048576;
set persist_only tmp_table_size=33554432;
set persist_only net_buffer_length=16384;

set persist_only transaction_isolation='READ-COMMITTED';

restart;

exit;
```
Install thread pool
```
sudo vi /etc/my.cnf

plugin-load-add=thread_pool.so
thread_pool_size=2
thread_pool_max_transactions_limit=200
thread_pool_algorithm=1
thread_pool_query_threads_per_group=100
thread_pool_dedicated_listeners=ON

```
Restart instance
```
mysql -uroot -h::1 -p"R00t_123" -e "restart"
```
Download and install Airport-DB
```
cd $HOME/installer
wget https://downloads.mysql.com/docs/airport-db.tar.gz
tar -zxvf airport-db.tar.gz
```
Install airport-db using MySQL Shell
```
mysql -uroot -h::1 -p"R00t_123" -e "set global local_infile=on"
mysqlsh root@localhost:3306 -- util loadDump 'airport-db'
```
Repeat using redo log disabled
```
mysql -uroot -h::1 -p"R00t_123" -e "alter instance disable innodb redo_log;"
mysql -uroot -h::1 -p"R00t_123" -e "SHOW GLOBAL STATUS LIKE 'Innodb_redo_log_enabled';"
mysql -uroot -h::1 -p"R00t_123" -e "drop database airportdb"

mysqlsh root@localhost:3306 -- util loadDump 'airport-db' --resetProgress=true
```
Repeat using flush log at trx commit disabled
```
mysql -uroot -h::1 -p"R00t_123" -e "alter instance enable innodb redo_log;"
mysql -uroot -h::1 -p"R00t_123" -e "SHOW GLOBAL STATUS LIKE 'Innodb_redo_log_enabled';"
mysql -uroot -h::1 -p"R00t_123" -e "set persist innodb_flush_log_at_trx_commit=0"
mysql -uroot -h::1 -p"R00t_123" -e "drop database airportdb"

mysqlsh root@localhost:3306 -- util loadDump 'airport-db' --resetProgress=true
```
create procedure do while
```
mysql -uroot -h::1 -p"R00t_123"

create database test;
use test;
create table test (i int);

DELIMITER // 
CREATE PROCEDURE doWhile() 
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE (i <= 5000) DO
    INSERT INTO test.test values (i);
    SET i = i+1;
  END WHILE;
END;
//
DELIMITER ;

call doWhile();

innodb_flush_log_at_trx_commit to 1

set persist innodb_flush_log_at_trx_commit=1;
call doWhile();
```

