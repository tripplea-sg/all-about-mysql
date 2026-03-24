# MySQL Enterprise Security
## Transparent Data Encryption (TDE) using Hashicorp
Install plugin 
```
vi /home/opc/mysql-sandboxes/3307/my.cnf

## content
early-plugin-load=keyring_hashicorp.so
keyring_hashicorp_server_url='https://10.0.0.96:8200'
keyring_hashicorp_role_id='183343a0-5847-b601-0621-b5523ff7fd00'
keyring_hashicorp_secret_id='2e748f90-b802-3953-f1ff-671b6394b3e4'
keyring_hashicorp_store_path='/v1/kv/mysql'
keyring_hashicorp_auth_path='/v1/auth/approle/login'
```
Restart 
```
mysql -uroot -h::1 -P3307 -e "restart"
```
Update keyring Hashicorp
```
mysql -uroot -h::1 -P3307 --skip-binary-as-hex -e "SELECT keyring_hashicorp_update_config();"
mysql -uroot -h::1 -P3307 -e "show variables like '%keyring%'";
```
Gather table data from OS
```
strings -a /home/opc/mysql-sandboxes/3307/sandboxdata/world_x/countryinfo.ibd
```
Encrypt table using TDE
```
mysql -uroot -h::1 -P3307 -e "alter table world_x.countryinfo encryption='Y'"
```
Gather table data from OS
```
strings -a /home/opc/mysql-sandboxes/3307/sandboxdata/world_x/countryinfo.ibd
```
Select data from MySQL
```
mysql -uroot -h::1 -P3307 -e "select * from world_x.countryinfo limit 1"
```
Unencrypt table
```
mysql -uroot -h::1 -P3307 -e "alter table world_x.countryinfo encryption='Y'"
```
## MySQL Enterprise Audit
Install plugin 
```
mysql -uroot -D mysql -h::1 -P3307 < /usr/share/mysql-8.1/audit_log_filter_linux_install.sql

mysql -uroot -h::1 -P3307 -e "show plugins"
```
Create audit log filter and assign to all
```
mysql -uroot -h::1 -P3307

SELECT audit_log_filter_set_filter('log_all', '{ "filter": { "log": true } }');
SELECT audit_log_filter_set_user('%', 'log_all');
SELECT audit_log_filter_flush() AS 'Result';

exit;
```
## MySQL Enterprise Firewall
### 1. Install Firewall
```
mysql -uroot -h127.0.0.1 -P3307 < /usr/share/mysql-8.1/linux_install_firewall.sql 

mysql -uroot -h127.0.0.1 -P3307

show variables like '%firewall%';

show plugins;

select * from mysql.firewall_users;

		lists names and operational modes of registered firewall account profiles

select * from mysql.firewall_whitelist;

		lists allowlist rules of registered firewall account profiles.

select * from information_schema.mysql_firewall_whitelist;

		a view into the in-memory data cache for MySQL Enterprise Firewall

select * from mysql.firewall_groups;

		lists names and operational modes of registered firewall group profiles

select * from mysql.firewall_group_allowlist;

		lists allowlist rules of registered firewall group profiles

select * from mysql.firewall_membership;

    lists the members (accounts) of registered firewall group profile

show global status like '%firewall%';

		Firewall_access_denied: The number of statements rejected 
		Firewall_access_granted: The number of statements accepted
		Firewall_access_suspicious: The number of statements logged
		Firewall_cached_entries: The number of statements recorded, including duplicates. 
```
### 2. Set important privileges
```
grant FIREWALL_EXEMPT on *.* to root@'localhost';
grant FIREWALL_ADMIN on *.* to root@'localhost';
```
### 3. Train Firewall
As root:
```
create user demo@'%' identified by 'demo';
grant all privileges on world_x.* to demo@'%';
call mysql.sp_set_firewall_group_mode('group1','RECORDING');
select * from mysql.firewall_groups;
call mysql.sp_firewall_group_enlist('group1','demo@%');
select * from mysql.firewall_membership;
exit;
```
As demo:
```
mysql -udemo -pdemo -h127.0.0.1 -P3307

show databases;
use world_x;
show tables;
select * from city;
exit;
```
As root:
```
mysql -uroot -h127.0.0.1 -P3307

SELECT MODE FROM performance_schema.firewall_groups WHERE NAME = 'group1';
SELECT * FROM performance_schema.firewall_membership WHERE GROUP_ID = 'group1' ORDER BY MEMBER_ID;
SELECT RULE FROM performance_schema.firewall_group_allowlist WHERE NAME = 'group1';
```
### 4. Testing the Firewall
AS ROOT, turn on protecting mode
```
call mysql.sp_set_firewall_group_mode('group1','PROTECTING');
grant firewall_user on *.* to demo@'%';
exit
```
As demo:
```
mysql -udemo -pdemo -h127.0.0.1 -P3307

show databases;
use world_x;
show tables;
select * from city;
select * from city where id=1;
```
## Encryption and Data Masking
### 1. Install Enterprise Encryption
As root:
```
mysql -uroot -h127.0.0.1 -P3307 --skip-binary-as-hex

CREATE FUNCTION asymmetric_decrypt RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_derive RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_encrypt RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_sign RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION asymmetric_verify RETURNS INTEGER SONAME 'openssl_udf.so';
CREATE FUNCTION create_asymmetric_priv_key RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION create_asymmetric_pub_key RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION create_dh_parameters RETURNS STRING SONAME 'openssl_udf.so';
CREATE FUNCTION create_digest RETURNS STRING SONAME 'openssl_udf.so';

```
### 2. Encrypt table
```
create table world_x.city_info_encrypted as select id, name, countrycode, district, hex(aes_encrypt(info, hex('passphrase'))) info from world_x.city;

select * from world_x.city_info_encrypted;

select id, name, countrycode, district, aes_decrypt(unhex(info), hex('passphrase')) from world_x.city_info_encrypted;
```
### 3. Install Data Masking
```
INSTALL PLUGIN data_masking SONAME 'data_masking.so';
CREATE FUNCTION gen_blacklist RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_dictionary RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_dictionary_drop RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_dictionary_load RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_range RETURNS INTEGER SONAME 'data_masking.so';
CREATE FUNCTION gen_rnd_email RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_rnd_pan RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_rnd_ssn RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION gen_rnd_us_phone RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION mask_inner RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION mask_outer RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION mask_pan RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION mask_pan_relaxed RETURNS STRING SONAME 'data_masking.so';
CREATE FUNCTION mask_ssn RETURNS STRING SONAME 'data_masking.so';
```
### 4. Use Data Masking
```
select mask_outer(name, 1,1), countrycode from world_x.city;
select mask_inner(name, 1,1), countrycode from world_x.city;

exit;
```
## Clean Up
```
mysqlsh -e "dba.stopSandboxInstance(3307)"
mysqlsh -e "dba.deleteSandboxInstance(3307)"

rm -Rf /home/opc/mysql-sandboxes

```

