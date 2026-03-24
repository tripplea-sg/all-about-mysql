# MySQL InnoDB Cluster on Kubernetes Workshop

## Prep: Install VM on OCI with minikube and PodMan
Refer to the following online reference: https://docs.oracle.com/en/learn/ol-minikube/index.html#for-more-information </br>
Install kubectl tool: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## A. Build Kubernetes Environment
### A.1. Deploy minikube
Login to your VM and run the following script to deploy a minikube with 2 Clusters (cluster1, cluster2)
```
./install-minikube-east-west.sh 
```
Exit and relogin to your VM.
### A.2. Install MySQL-Operator 
Run the following script to deploy mysql-operator on cluster1 and cluster2
```
./install-mysql-operator-east-west.sh
```
### A.3. Install skupper 
Run the following script to create east-cluster namespace on cluster1 and cluster2; and use skupper to link these 2 namespaces.
```
./install-skupper-east-west.sh
```
### A.4. Check environment
Run the following script to check environment readiness
```
./check-environment-east-west.sh 
```
### A.5. Check InnoDB Cluster
Observe pods running on cluster1 and cluster2, skupper link between namespaces, and the message about InnoDB Cluster
```
./check-ic-east-west.sh
```
### A.6. Test using cluster1
Switch to cluster1 anytime by running this script
```
./east_cluster.sh 
```
### A.7. Test using cluster2
Switch to cluster2 anytime by running this script
```
./west_cluster.sh
```
## B. Deploy InnoDB Cluster with MySQL Operator
Observe tmp/east-cluster.yaml below to deploy InnoDB Cluster on cluster1
```
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster-east
  namespace: east-cluster
spec:
  secretName: mypwds
  tlsUseSelfSigned: true
  baseServerId: 1000
  instances: 3
  router:
    instances: 1
```
Observe tmp/west-cluster.yaml below to deploy InnoDB Cluster on cluster2
```
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster-east
  namespace: west-cluster
spec:
  secretName: mypwds
  tlsUseSelfSigned: true
  baseServerId: 100
  instances: 3
  router:
    instances: 1
```
Observe the following command to deploy a secret for MySQL server root password 
```
./east_cluster.sh
kubectl create secret generic mypwds --from-literal=rootUser=root --from-literal=rootHost=% --from-literal=rootPassword="root"

./west_cluster.sh
kubectl create secret generic mypwds --from-literal=rootUser=root --from-literal=rootHost=% --from-literal=rootPassword="root"
```
Apply yaml file using kubectl after switch to cluster1 for east-cluster and cluster2 for west-cluster, or simply use the following script:
```
./deploy-cluster-east.sh
./deploy-cluster-west.sh
```
Monitor InnoDB Cluster creation (3 MySQL servers + 1 Router) on cluster1 and cluster2 by using the following:
```
./east_cluster.sh
kubectl get ic
./west_cluster.sh
kubectl get ic
```
Or, simply run the following script to get overall status:
```
./check-ic-east-west.sh
```
## C. Scaling Up InnoDB Cluster with MySQL Operator
Scaling up east-cluster into 5 MySQL servers and 2 MySQL Routers
```
./east_cluster.sh
kubectl edit ic mycluster-east
```
Change instances to 5 and router.instances to 2; save with :wq to commit. </br>
Monitor the deployment using the following script:
```
./check-ic-east-west.sh
```
Check MySQL InnoDB Cluster (mycluster-east):
```
./east_cluster.sh
kubectl exec -it mycluster-east-0 -- mysqlsh root:root@localhost:3306 -- cluster status
kubectl exec -it mycluster-east-0 -- mysqlsh root:root@localhost:3306 -- cluster listRouters
```
Check MySQL InnoDB Cluster (mycluster-west):
```
./west_cluster.sh
kubectl exec -it mycluster-west-0 -- mysqlsh root:root@localhost:3306 -- cluster status
kubectl exec -it mycluster-west-0 -- mysqlsh root:root@localhost:3306 -- cluster listRouters
```
## D. Scaling Down InnoDB Cluster with MySQL Operator
```
./east_cluster.sh
kubectl apply -f tmp/east-cluster.yaml
```
Check progress using the following script
```
./check-ic-east-west.sh
./east_cluster.sh
kubectl exec -it mycluster-east-0 -- mysqlsh root:root@localhost:3306 -- cluster status
kubectl exec -it mycluster-west-0 -- mysqlsh root:root@localhost:3306 -- cluster listRouters
```
Remove unused PVC
```
kubectl get pvc
kubectl delete pvc datadir-mycluster-east-4
kubectl delete pvc datadir-mycluster-east-3
```
## E. Backup InnoDB Cluster with MySQL Operator
Observe the following yaml to create backup pvc
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myexample-pv
  namespace: east-cluster
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myexample-pvc
  namespace: east-cluster
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
Apply the yaml to deploy PVC
```
./east_cluster.sh
kubectl apply -f tmp/backup-pvc-east.yaml
kubectl get pv
kubectl get pvc
```
Observe the following yaml to modify mycluster-east to have a backup profile:
```
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mycluster-east
  namespace: east-cluster
spec:
  secretName: mypwds
  tlsUseSelfSigned: true
  baseServerId: 1000
  instances: 3
  router:
    instances: 1
  backupProfiles:       
  - name: myfancyprofile  # Embedded backup profile
    dumpInstance:         # MySQL Shell Dump
      storage:
        persistentVolumeClaim:
          claimName: myexample-pvc # store to this pre-existing PVC
      backupSchedules:
        - name: mygreatschedule
          schedule: "0 0 * * *" # Daily, at midnight
          backupProfileName:  myfancyprofile # reference the desired backupProfiles's name 
          enabled: true # backup schedules can be temporarily disabled
  ```
  Apply the yaml using kubectl
  ```
  ./east_cluster.sh
  kubectl apply -f tmp/east-cluster-with-backup.yaml
  ./check-ic-east-west.sh
  ```
  Observe the following YAML for one-off backup
  ```
  apiVersion: mysql.oracle.com/v2
kind: MySQLBackup
metadata:
    name: a-cool-one-off-backup
    namespace: east-cluster
spec:
  clusterName: mycluster-east
  backupProfileName: myfancyprofile
```
Run One off backup
```
kubectl apply -f tmp/cluster-east-one-off-backup.yaml 
kubectl get mysqlbackup
```
## F. Create InnoDB ClusteSet with MySQL Operator
Run the following script to install ClusterSet on mycluster-east
```
./make_clusterset_east.sh
```
Run the following script to install ReplicaCluster on mycluster-west
```
./make_replicacluster_west.sh 
```




