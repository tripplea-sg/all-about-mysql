## MySQL Operator Workshop

### Start minikube
The following command aims to start minikube using 4 OCPU and 11 GB RAM
```
minikube delete --all
minikube start --driver=podman
```
See what are images available on minikube's docker local registry
```
minikube ssh 'docker images'
```
### Import MySQL Enterprise Edition and MySQL Enterprise Operator Container Images
Copy container images from local machine into minikube
```
scp -i ~/.minikube/machines/minikube/id_rsa /home/opc/images/*.tar docker@$(minikube ip):/home/docker/
scp -i ~/.minikube/machines/minikube/id_rsa /home/opc/images/*.tar.gz docker@$(minikube ip):/home/docker/
```
The following steps aim to load MySQL Enterprise container images into minikube's docker local registry
```
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) -- docker load -i /home/docker/mysql-enterprise-operator-8.0.32-2.0.8-docker.tar.gz
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) -- docker load -i /home/docker/mysql-enterprise-server-8.0.33.tar
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) -- docker load -i /home/docker/mysql-enterprise-router-8.0.33.tar
```
TAGS those images as follow:
```
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) -- docker tag mysql/enterprise-operator:8.0.32-2.0.8 container-registry.oracle.com/mysql/enterprise-operator:8.0.32-2.0.8
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) -- docker tag mysql/enterprise-server:8.0 container-registry.oracle.com/mysql/enterprise-server:8.0.32
ssh -i ~/.minikube/machines/minikube/id_rsa docker@$(minikube ip) -- docker tag mysql/enterprise-router:8.0 container-registry.oracle.com/mysql/enterprise-router:8.0.32
```
See what are images available on minikube's docker local registry
```
minikube ssh 'docker images'
```
### Install MySQL Enterprise Operator
First install the Custom Resource Definition (CRD) used by MySQL Operator for Kubernetes: 
```
kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/trunk/deploy/deploy-crds.yaml
```
Download Manifest
```
wget https://raw.githubusercontent.com/mysql/mysql-operator/trunk/deploy/deploy-operator.yaml
```
Edit manifest to change image and imagePullPolicy as follow:
```
image: container-registry.oracle.com/mysql/enterprise-operator:8.0.32-2.0.8
imagePullPolicy: Never
```
Apply manifest
```
kubectl apply -f deploy-operator.yaml
```
Wait until mysql-operator is running. Once running, then check below:
```
kubectl -n mysql-operator exec -it mysql-operator-5f75574846-64w2c -- mysqlsh --version
```
Check if MySQL Enterprise Operator deployment is successful
```
kubectl get ns

kubectl -n mysql-operator get pod
```
### Deploy InnoDB Cluster
Create namespace
```
kubectl create ns mysql-cluster
```
Create Secret
```
kubectl -n mysql-cluster create secret generic mypwds \
        --from-literal=rootUser=root \
        --from-literal=rootHost=% \
        --from-literal=rootPassword="root"
```
See our YAML
```
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
  namespace: mysql-cluster
spec:
  imagePullPolicy: Never
  secretName: mypwds
  tlsUseSelfSigned: true
  edition: enterprise
  instances: 3
  router:
    instances: 1
```
Run our YAML to deploy innodb cluster
```
kubectl apply -f mycluster.yaml
```
Check our InnoDB Cluster
```
kubectl -n mysql-cluster get ic --watch
```
Check our InnoDB Cluster pods
```
kubectl -n mysql-cluster get pod
```
Check PV/PVC
```
kubectl -n mysql-cluster get pv
kubectl -n mysql-cluster get pvc
```
Check secret
```
kubectl -n mysql-cluster get secret
```
Check all
```
kubectl -n mysql-cluster get all

kubectl -n mysql-cluster get svc
```
Get more information using describe
```
kubectl -n mysql-cluster describe ic mycluster
kubectl -n mysql-cluster describe sts mycluster
kubectl -n mysql-cluster describe deployment mycluster-router
kubectl -n mysql-cluster describe svc mycluster
kubectl -n mysql-cluster describe svc mycluster-instances

```
Check our InnoDB Cluster status using MySQL Shell
```
kubectl -n mysql-cluster exec -it mycluster-0 -c mysql -- mysqlsh root:root@localhost:3306 -- cluster status
```
Check MySQL version:
```
kubectl -n mysql-cluster exec -it mycluster-0 -c mysql -- mysqld --version
kubectl -n mysql-cluster exec -it mycluster-0 -- mysqlsh --version
```
Check MySQL Router version
```
kubectl -n mysql-cluster exec -it mycluster-router-c6f658786-2klw2 -- mysqlrouter --version
```
Scaling Up MySQL Router
```
kubectl -n mysql-cluster edit ic mycluster

# change router instances from 1 to 2 and save to exit

kubectl -n mysql-cluster get pod
```
### Connect using MySQL Shell through MySQL Router
Run MySQL Shell pod
```
kubectl -n mysql-cluster run --rm -it myshell --image=container-registry.oracle.com/mysql/community-operator -- mysqlsh
```
Connect to InnoDB Cluster via Router's clusterIP
```
\connect root:root@mycluster:6446
\sql
select @@hostname;
```
### Backup MySQL using MySQL Shell
Modify our cluster to add backup volume
```
kubectl apply -f mycluster-with-backup.yaml

kubectl -n mysql-cluster get pv
kubectl -n mysql-cluster get pvc
kubectl -n mysql-cluster describe ic mycluster
kubectl -n mysql-cluster describe sts mycluster
```
Run One Time Backup
```
kubectl apply -f one-time-backup.yaml
```
Check backup files in minikube file system
```
minikube ssh 'sudo ls /tmp/'
```
More information: https://dev.mysql.com/doc/mysql-operator/en/mysql-operator-introduction.html
