# Short MySQL on Kubernetes Workshop
### Get node
The following command shows list of kubernetes nodes
```
kubectl get node
```
### Get namespaces
List of kubernetes cluster namespace
```
kubectl get ns
```
### Get MySQL Operator
The operator setup
```
kubectl -n mysql-operator get all
```
### Install MySQL as statefulset without operator
Create new namespace
```
kubectl create ns mysql-cluster
```
Use the following YAML and save as mysql.yaml
```
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-root-password
  namespace: mysql-cluster
type: Opaque
data:
  password: cm9vdA==
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql-cluster
spec:
  serviceName: "mysql"
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql/mysql-server:latest
        env:
         - name: MYSQL_ROOT_PASSWORD
           valueFrom:
             secretKeyRef:
                name: mysql-root-password
                key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data-vol
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data-vol
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: mysql-cluster
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql-port
  clusterIP: None
  selector:
    app: mysql
```
Apply mysql.yaml
```
kubectl apply -f mysql.yaml
```
It will deploy 1 MySQL instance as statefulset
```
kubectl -n mysql-cluster get all
```
### How to Login from outside using kubectl
Login to MySQL instance
```
kubectl -n mysql-cluster exec -it mysql-0 -- mysql -uroot -proot -h127.0.0.1
```
### How to scale up the statefulset
Edit configuration to make 3 MySQL
```
kubectl -n mysql-cluster edit sts mysql
```
3 MySQL will be running
### Using MySQL Operator to deploy InnoDB Cluster
Create a yaml with name mycluster.yaml
```
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster
  namespace: mysql-cluster
spec:
  secretName: mypwds
  tlsUseSelfSigned: true
  instances: 3
  router:
    instances: 1
```
Apply mycluster.yaml to deploy 3 nodes cluster + 1 MySQL Router
```
kubectl apply -f mycluster.yaml
```
### Scaling Up
Edit innodb cluster router replica from 1 to 2
```
kubectl -n mysql-cluster edit ic mycluster
```
