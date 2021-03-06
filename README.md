# High Availability  of Oracle DB Pod Practice via Kubernetes Statefulset

###  Requirement:
It is similar as Oracle Rac One Architecture.  The target is to use Kubernetes to manage Oracle DB Pods like Rac One.  It has all the benefits of Rac One has. Details of Rac One benefits, please refer oracle Rac One [official website ][1]  
When one db pod dies or node dies, Kubernetes would start a new DB pod in the same node or another node. The datafiles are on Oracle File system (NFS) and they can be accessed by all nodes associated with Oracle DB Pods. In this example it is labeled as ha=livesqldb  

###  Solution:

* Need to make sure the nodes which can run DB pods has the same access to the NFS
* Label nodes with  ha=livesqlsb, in our case we have 2 nodes labeled
```
kubectl label  node instance-cas-db2 ha=livesqlsb  
node "instance-cas-db2" labeled  
kubectl label  node instance-cas-mt2  ha=livesqlsb  
node "instance-cas-mt2" labeled
```
* Need to create StatefulSet and replicas: 1 ,yaml is like
```
apiVersion: apps/v1  
kind: StatefulSet  
metadata:  
  name: livesqlsb-db  
  labels:  
    app: livesqlsb-db  
spec:  
  selector:  
        matchLabels:  
           ha: livesqlsb  
  serviceName: livesqlsb-db-service  
  replicas: 1  
  template:  
    metadata:  
        labels:  
           ha: livesqlsb  
    spec:  
      terminationGracePeriodSeconds: 30  
      volumes:  
        - name: livesqlsb-db-pv-storage1  
          persistentVolumeClaim:  
            claimName: livesql-pv-nfs-claim1  
      containers:  
        - image: oracle/database:18.3v2  
          name: livesqldb  
          ports:  
            - containerPort: 1521  
              name: livesqldb  
          volumeMounts:  
            - mountPath: /opt/oracle/oradata  
              name: livesqlsb-db-pv-storage1  
          env:  
            - name: ORACLE_SID  
              value: "LTEST"  
            - name: ORACLE_PDB  
              value: "ltestpdb"
```
* We use kubectl drain   \--ignore-daemonsets  --force to test node eviction. It would shutdown the pod gracefully and wait 30s to start a new pod  in another node
* Or kubectl delete pod  to test  pod eviction. It would shutdown the pod gracefully and wait 30s to start a new pod in the same node 
* To automate kubectl drain the node which is unresponsive (ie host crash, or disk faulty), we can crontab below script to drain the node
```
#!/bin/sh

KUBECTL="/bin/kubectl"

# Get only nodes which are not drained yet
NOT_READY_NODES=$($KUBECTL get nodes | grep  'NotReady' | awk '{print $1}' | xargs echo)
# Get only nodes which are still drained
READY_NODES=$($KUBECTL get nodes | grep '\sReady,SchedulingDisabled' | awk '{print $1}' | xargs echo)

echo "Unready nodes that are be drained: $NOT_READY_NODES"
echo "Ready nodes which need to uncordon: $READY_NODES"


for node in $NOT_READY_NODES; do
  echo "Node $node not drained yet, draining..."
  $KUBECTL drain --ignore-daemonsets --force $node
  echo "Done"
done;

date
for node in $READY_NODES; do
  echo "Node $node are ready, uncordoning..."
  $KUBECTL uncordon $node
  echo "Done"
done;
```

[1]: https://www.oracle.com/database/technologies/rac/racone.html
