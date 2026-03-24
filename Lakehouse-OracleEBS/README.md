# MySQL HeatWave integration with EBS

This GitHub repository explains and demonstrates how to set up a simple environment that showcases the integration between Oracle E-Business Suite (EBS) and 
MySQL HeatWave through the Lakehouse architecture. It provides step-by-step guidance, examples, and configurations to help users understand how data can seamlessly flow from EBS into HeatWave, enabling advanced analytics, 
reporting, and insights without complex ETL processes. The goal is to make it easier to learn and replicate the integration in a controlled, simplified setup.

## Create OCI environment

### Create Virtual Cloud Network (VCN)

Launch the OCI Console, go to Networking → Virtual Cloud Networks, and use the VCN with Internet Connectivity wizard. 
[Oracle Docs](https://docs.oracle.com/en/learn/lab_virtual_network/index.html?utm_source=chatgpt.com)

Provides sample values for fields like “VCN Name” and IPv4 CIDR blocks, complete with visual guidance and review steps.
[Oracle Docs](https://docs.oracle.com/en/learn/lab_virtual_network/index.html?utm_source=chatgpt.com)

**VCN Name** = EBS_workshop<br>
Click **Next**, click **Create.**, and Click **View VCN** <br><br>
Go to **Subnet** , click on **private subnet-EBS_workshop**, click **security**, click **security list for private subnet-EBS_workshop**, and click **Security Rules** <br>
* Click **Add Ingress Rules**, and: Source CIDR: 0.0.0.0/0, Source port range: All, Destination port range: 3306, 33060, Description: mysql/mysqlx ports <br>
Click **Add Ingress Rules** <br><br>

Go back to **EBS_workshop** , click on **Subnets**, click **public subnet-EBS_workshop**, click **security**, click **Default security for EBS_workshop**, and click **Security Rules** <br>
* Click **Add Ingress Rules**, and: Source CIDR: 0.0.0.0/0, Source port range: All, Destination port range: 3306, 33060, Description: mysql/mysqlx ports <br>
Click **Add Ingress Rules**
* Click **Add Ingress Rules**, and: Source CIDR: 0.0.0.0/0, Source port range: All, Destination port range: 5901, Description: vnc server ports <br>
Click **Add Ingress Rules**
* Click **Add Ingress Rules**, and: Source CIDR: 0.0.0.0/0, Source port range: All, Destination port range: 8000,9000, Description: EBS ports <br>
Click **Add Ingress Rules**
* Click **Add Ingress Rules**, and: Source CIDR: 0.0.0.0/0, Source port range: All, Destination port range: 1521, Description: Oracle port <br>
Click **Add Ingress Rules**
* Click **Add Ingress Rules**, and: Source CIDR: 0.0.0.0/0, Source port range: All, Destination port range: 80,443,8501, Description: others <br>
Click **Add Ingress Rules**

### Create MySQL HeatWave 

Provisioning MySQL HeatWave: This guide outlines the steps to create a MySQL HeatWave DB System and enable the HeatWave cluster. It covers prerequisites, including setting up a Virtual Cloud Network (VCN) and configuring networking between OCI and Azure. 
[Oracle Docs](https://docs.oracle.com/en-us/iaas/odsaz/odsa-provisioning-mysql-heatwave.html?utm_source=chatgpt.com)

Provisioning HeatWave Nodes: After setting up the DB System, this document explains how to add and manage HeatWave nodes, including resizing the cluster and utilizing MySQL HeatWave Autopilot for node estimation. 
[Oracle Docs](https://docs.oracle.com/en-us/iaas/Content/database-for-azure-provision/odsa-provisioning-heatwave-nodes.html?utm_source=chatgpt.com)

* Name: EBS_workshop
* Username: admin
* Password: <you define>
* Setup: standalone
* VCN: EBS_workshop
* Subnet: private subnet-EBS_workshop
* Shape: MySQL.4 + HeatWave.512GB (make sure Lakehouse is enabled)
* Initial data storage: 50GB
* Disable backup plan

### Create Object Storage Bucket

To create an Object Storage bucket in Oracle Cloud Infrastructure (OCI), you can follow the official documentation provided by Oracle. [Oracle Documentation](https://docs.oracle.com/iaas/Content/Object/Tasks/managingbuckets_topic-To_create_a_bucket.htm?utm_source=chatgpt.com)

Object Storage Bucket name: EBS_workshop

### Create Compute Node Running EBS Vision Instance

Creating a Compute Instance: [Oracle Docs](https://docs.oracle.com/en-us/iaas/compute-cloud-at-customer/topics/compute/compute-instances.htm?utm_source=chatgpt.com)
* Name: EBS_workshop
* Change Image: marketplace, E-Business Suite Demo Install Image ver 12.2.14
* Shape: 8 OCPU, 128 GB RAM
* VCN: EBS_workshop
* Boot disk size: 800GB
* Subnet: public subnet-EBS_workshop

## Configure Compute Node

### Install S3FS
Login to the Compute node and install:
```
sudo dnf install -y epel-release
sudo dnf update -y
sudo dnf install -y gcc libstdc++-devel fuse-devel curl-devel libxml2-devel mailcap automake autoconf
sudo yum-config-manager --enable ol8_baseos_latest ol8_appstream ol8_addons ol8_developer_EPEL
sudo yum install s3fs-fuse
s3fs --version
sudo setenforce 0
sudo systemctl stop firewalld
mkdir -p /home/opc/object_storage
```
Create a credentials file (~/.passwd-s3fs) with this format. To create an API Signing Key in Oracle Cloud Infrastructure (OCI), you can follow the official documentation provided by Oracle. This process involves generating a key pair (private and public keys) that allows secure authentication for API requests. Generating an API Signing Key: 
[Oracle Documentation](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm?utm_source=chatgpt.com)
```
ACCESS_KEY_ID:SECRET_ACCESS_KEY
```
Secure the file and mount object storage bucket
```
chmod 400 ~/.passwd-s3fs
```
Resize block volume to 800G from OCI console 
```
sudo dnf install -y cloud-utils-growpart
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ocivolume-root
sudo xfs_growfs /
df -h
```
mount object storage
```
s3fs EBS_workshop /home/opc/object_storage -o endpoint={region} -o passwd_file=${HOME}/.passwd-s3fs -o url=https://{namespace}.compat.objectstorage.{region}.oraclecloud.com/ -onomultipart -o use_path_request_style
```
### Install vncserver
```
sudo setenforce 0
sudo systemctl stop firewalld
sudo dnf groupinstall "Server with GUI" -y
sudo dnf install tigervnc-server -y
vncpasswd
sudo cp /lib/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:1.service
vncserver :1
```
### Configure E-Business Suite
```
sudo hostnamectl set-hostname apps.example.com
```
Add "apps.example.com db.example.com apps" at the back of a line with IP address in /etc/hosts <br>
Start EBS database:
```
cd /u01/install/APPS
. EBSapps.env
/u01/install/APPS/script/startdb.sh
```
Fix context file
```
vi /u01/install/APPS/fs1/inst/apps/EBSDB_apps/appl/admin/EBSDB_apps.xml
s_webentryhost value = apps
s_webentrydomain value = example.com
login_page value = http://apps.example.com:8000/OA_HTML/AppsLogin
```
Run Autoconfig
```
/u01/install/APPS/fs1/inst/apps/EBSDB_apps/admin/scripts/adautocfg.sh
```
Start EBS apps
```
/u01/install/APPS/scripts/startapps.sh
```
