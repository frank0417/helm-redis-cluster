# helm-redis-cluster

This is Helm Chart to create Redis Cluster on IBM Cloud Kubernetes Service. 
it provides High Availability, Cost Saving, High Performance way.


## Motivation

Redis is basic tool for the [12 Factor Apps](https://12factor.net/). it is a stateless application which need a cache service.

Some web systems rely on “sticky sessions" that is caching user session data in memory of the app’s process and expecting future requests from the same visitor to be routed to the same process.

In Kubernetes, NGINX Ingrss Controller provides Sticky session and Load balancing to Web system. but, Sticky sessions are a violation of twelve-factor and should never be used or relied upon.

According to 12 Factor Apps, Application should save session data to a cache service that should provides automatically delete function by time period expieration. Redis is a good candidate for a datastore that offers time-expiration.

IBM Cloud provides a managed Redis service. its Redis client must connect to service by encryped virtual communiction circuit due to its service open to public network. Original redis protocol does not have encryption due to cache server must located near application server due to it need large bandwidth by frequent communication.

if Redis client must use encypted communication, it need to special library to establise session from client to server. it bring many limitation to customer application.

it is better that Redis caching service and application is in same Kubernetes cluster that communication close in own managed area.

StatefulSet controller is good candidate for Redis server platform. but, it need a persistent volume service on Cloud Platform. in IBM Cloud, Usually StatefulSet dynamically provision Endurance Storage service as Persistent Volume. but, it minimal size is 20GB which is too large for this use case.

I have developed Helm chart to build Redis cluster without Endurance Storage. this Helm Chart use SSD in Kubernetes node as StatefulSet Persistent Volumes. so, it is be able to save cost for IBM Cloud charge.

## Redis Cluster configuration

This Helm Chart configure a Redis Cluster on 3 node of Kubernetes. Those PVs are made from SSD on Node internal storage. If one of Node lost, Redis Service maintain.

![]¥(Redis_Cluster_Config.png)

there are 3 master server for each zone, and 2 replica for 1 master. those Redis nodes distributed on for each Zone bring you High Avalabilty Redis Service.

If one of K8s nodes goes down, one Redis master and 2 Replicas will be lost. but losted master has Replia in other nodes. it will take over Redis master quickly. in this way, Redis service can continue at node  obstacle.

Unfortunattly, Helm does not provide shell executable function in its chart. so, Redis cluster configuration have to do manually.

## Step by Step Configuration

### IKS K8s Cluster

Configuration Point for IKS

- Multi Zone: TOK02,TOK04,TOK05
- Number of Nodes: 1 / Zone, total 3 nodes
- Node spec: vCPU x 2, 4GB RAM, 1st SSD 25GB, 2nd SSD 100GB

### Redis Cluster installation by Helm Chart

Helm chart of Redis Cluster clones GitHub URL to local, refer below.

~~~terminal:
git clone https://github.com/takara9?tab=repositories redis-cluster
~~~

if your environment does not have Helm command, you should install Helm, please refer [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started).

### Configure Helm in your environment

Helm command is initialized and deployed a service account and give permission for the tiller in K8s cluster, in below example.

~~~terminal:
helm init
kubectl apply -f https://raw.githubusercontent.com/IBM-Cloud/kube-samples/master/rbac/serviceaccount-tiller.yaml
helm init --service-account tiller --upgrade
~~~

About 5 mins later applying above manifest, you can execute following helm commands. make sure Helm works right.

~~~terminal:
helm version
helm ls
~~~

After check, install Redis cluster by this helm chart. <Release_Name> should replace your Helm release name. recommended name is "redis" + number, for example redis1, redis2...

~~~terminal:
helm install -n <Release_Name> --namespace redis ./redis-cluster
~~~

This Helm chart work item in below.

- Create new Storage Class for Local Volume
- Create Perssistent Volume by new storage class for each node, those volumes purpose is to create mount path next step.
- Create 3 mount path in SSD local device for each node
- Create 9 paires which are of Persistent Volume and Persistent Volume Claim
- Deploy 9 pods by StatefulSet controller, Those pods are included a Redis latest version container



~~~terminal:
$ helm install -n redis1 --namespace redis ./redis-cluster
NAME:   redis1
LAST DEPLOYED: Wed Aug 14 11:04:16 2019
NAMESPACE: redis
STATUS: DEPLOYED

RESOURCES:
==> v1/StatefulSet
NAME                  DESIRED  CURRENT  AGE
redis1-cluster-tok02  3        1        0s
redis1-cluster-tok04  3        1        0s
redis1-cluster-tok05  3        1        0s

==> v1/Pod(related)
NAME                    READY  STATUS   RESTARTS  AGE
redis1-cluster-tok02-0  0/1    Pending  0         0s
redis1-cluster-tok04-0  0/1    Pending  0         0s
redis1-cluster-tok05-0  0/1    Pending  0         0s

==> v1/ConfigMap
NAME                  DATA  AGE
redis-cluster-redis1  2     0s

==> v1/StorageClass
NAME           PROVISIONER                   AGE
local-storage  kubernetes.io/no-provisioner  0s

==> v1/PersistentVolume
NAME                  CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS     CLAIM                              STORAGECLASS  REASON  AGE
local-redis1-tok02-0  1Gi       RWO           Retain          Available  local-storage                      0s
local-redis1-tok02-1  1Gi       RWO           Retain          Available  local-storage                      0s
local-redis1-tok02-2  1Gi       RWO           Retain          Bound      redis/data-redis1-cluster-tok02-0  local-storage  0s
local-redis1-tok04-1  1Gi       RWO           Retain          Available  local-storage                      0s
local-redis1-tok04-0  1Gi       RWO           Retain          Bound      redis/data-redis1-cluster-tok04-0  local-storage  0s
local-redis1-tok04-2  1Gi       RWO           Retain          Available  local-storage                      0s
local-redis1-tok05-0  1Gi       RWO           Retain          Available  local-storage                      0s
local-redis1-tok05-1  1Gi       RWO           Retain          Available  local-storage                      0s
local-redis1-tok05-2  1Gi       RWO           Retain          Bound      redis/data-redis1-cluster-tok05-0  local-storage  0s
redis1-tok02          1Gi       RWO           Retain          Bound      redis/redis1-tok02                 local-storage  0s
redis1-tok04          1Gi       RWO           Retain          Bound      redis/redis1-tok04                 local-storage  0s
redis1-tok05          1Gi       RWO           Retain          Bound      redis/redis1-tok05                 local-storage  0s

==> v1/PersistentVolumeClaim
NAME          STATUS  VOLUME        CAPACITY  ACCESS MODES  STORAGECLASS   AGE
redis1-tok02  Bound   redis1-tok02  1Gi       RWO           local-storage  0s
redis1-tok04  Bound   redis1-tok04  1Gi       RWO           local-storage  0s
redis1-tok05  Bound   redis1-tok05  1Gi       RWO           local-storage  0s

==> v1/Service
NAME           TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)             AGE
redis-cluster  ClusterIP  None        <none>       6379/TCP,16379/TCP  0s

==> v1/Pod
NAME                READY  STATUS   RESTARTS  AGE
setup-redis1-tok02  0/1    Pending  0         0s
setup-redis1-tok04  0/1    Pending  0         0s
setup-redis1-tok05  0/1    Pending  0         0s
~~~

### Change Namespace of Redis

helm command creates new namespace which for Redis Cluster. Redis configuration work should take place in redis namespace. next step is a creating new context for redis namespace.

you should pick up necessary item to make new context of Redis

~~~terminal:
kubectl config view
~~~

Necessary items to create new context

- cluster.name: cluster name which you gave name when you created a cluster.
- users.name: email address if you use IKS

Example command line to create a context that use namespace redis.

~~~terminal:
kubectl config set-context redis --cluster=cluster_nickname --user=user_name
--namespace=redis
~~~

you can confirm new context successfully.

~~~terminal:
kubect config get contexts
~~~

change context redis, you are in redis namespace allways after this.

~~~terminal:
kubectl config use-context redis
~~~

### Redis Cluster configuration

list Redis cluster menber of Pod in following.

~~~terminal:
$ kubectl get po -o wide
NAME                     READY   STATUS      RESTARTS   AGE   IP               NODE
redis1-cluster-tok02-0   1/1     Running     0          86s   172.30.198.81    10.212.5.220
redis1-cluster-tok02-1   1/1     Running     0          60s   172.30.198.82    10.212.5.220
redis1-cluster-tok02-2   1/1     Running     0          37s   172.30.198.83    10.212.5.220
redis1-cluster-tok04-0   1/1     Running     0          86s   172.30.23.208    10.192.21.111
redis1-cluster-tok04-1   1/1     Running     0          63s   172.30.23.209    10.192.21.111
redis1-cluster-tok04-2   1/1     Running     0          43s   172.30.23.210    10.192.21.111
redis1-cluster-tok05-0   1/1     Running     0          86s   172.30.201.151   10.193.10.50
redis1-cluster-tok05-1   1/1     Running     0          66s   172.30.201.152   10.193.10.50
redis1-cluster-tok05-2   1/1     Running     0          46s   172.30.201.153   10.193.10.50
setup-redis1-tok02       0/1     Completed   0          86s   172.30.198.80    10.212.5.220
setup-redis1-tok04       0/1     Completed   0          86s   172.30.23.207    10.192.21.111
setup-redis1-tok05       0/1     Completed   0          86s   172.30.201.150   10.193.10.50
~~~

To make Redis cluster, redis-cli comannd execute one of cluster member.
In following command, 'redis1-cluster-tok02-0' is one of redis cluster member.
'app=redis-cluster-redis1' is the label of Helm release, is a identifier for Helm release. you can make sure this label by 'kubectl get po --show-labels' command.

~~~terminal:
$ kubectl exec -it redis1-cluster-tok02-0 -- redis-cli --cluster create --cluster-replicas 2 $(kubectl get pods -l 'app=redis-cluster-redis1' -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
>>> Performing hash slots allocation on 9 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.30.23.209:6379 to 172.30.198.81:6379
Adding replica 172.30.23.210:6379 to 172.30.198.81:6379
Adding replica 172.30.201.151:6379 to 172.30.198.82:6379
Adding replica 172.30.201.152:6379 to 172.30.198.82:6379
Adding replica 172.30.201.153:6379 to 172.30.198.83:6379
Adding replica 172.30.23.208:6379 to 172.30.198.83:6379
M: 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 172.30.198.81:6379
   slots:[0-5460] (5461 slots) master
M: 82f41aee313dc3cd9ef2e9ecefc6432bf9428375 172.30.198.82:6379
   slots:[5461-10922] (5462 slots) master
M: b295c7aeede1988e1f97e58c32550537a8436c21 172.30.198.83:6379
   slots:[10923-16383] (5461 slots) master
S: 3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 172.30.23.208:6379
   replicates b295c7aeede1988e1f97e58c32550537a8436c21
S: 8bcc0a07889be0369902e37e56ba3c4a93df8644 172.30.23.209:6379
   replicates 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f
S: a305234d01f80d6bb63471ad9d394945275cca7a 172.30.23.210:6379
   replicates 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f
S: 0c2453357a09efc64940e44a8147f614f5911a9b 172.30.201.151:6379
   replicates 82f41aee313dc3cd9ef2e9ecefc6432bf9428375
S: 09fb2428bcf52dd233c5d5b9729820406c812f7e 172.30.201.152:6379
   replicates 82f41aee313dc3cd9ef2e9ecefc6432bf9428375
S: 44697db50d047ae4c5a9fbdd7927a9ccbb15eaa0 172.30.201.153:6379
   replicates b295c7aeede1988e1f97e58c32550537a8436c21
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
........
>>> Performing Cluster Check (using node 172.30.198.81:6379)
M: 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 172.30.198.81:6379
   slots:[0-5460] (5461 slots) master
   2 additional replica(s)
S: 3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 172.30.23.208:6379
   slots: (0 slots) slave
   replicates b295c7aeede1988e1f97e58c32550537a8436c21
S: 44697db50d047ae4c5a9fbdd7927a9ccbb15eaa0 172.30.201.153:6379
   slots: (0 slots) slave
   replicates b295c7aeede1988e1f97e58c32550537a8436c21
S: a305234d01f80d6bb63471ad9d394945275cca7a 172.30.23.210:6379
   slots: (0 slots) slave
   replicates 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f
S: 0c2453357a09efc64940e44a8147f614f5911a9b 172.30.201.151:6379
   slots: (0 slots) slave
   replicates 82f41aee313dc3cd9ef2e9ecefc6432bf9428375
M: 82f41aee313dc3cd9ef2e9ecefc6432bf9428375 172.30.198.82:6379
   slots:[5461-10922] (5462 slots) master
   2 additional replica(s)
M: b295c7aeede1988e1f97e58c32550537a8436c21 172.30.198.83:6379
   slots:[10923-16383] (5461 slots) master
   2 additional replica(s)
S: 09fb2428bcf52dd233c5d5b9729820406c812f7e 172.30.201.152:6379
   slots: (0 slots) slave
   replicates 82f41aee313dc3cd9ef2e9ecefc6432bf9428375
S: 8bcc0a07889be0369902e37e56ba3c4a93df8644 172.30.23.209:6379
   slots: (0 slots) slave
   replicates 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
~~~

### Master & Slave adjustment to maximaize HA

pleaase pay attention to following teminal output.
it displayed Redis cluster node. first 3 lines are in Zone-A, 2nd 3 lines are in Zone-C TOK05, last 3 are in Zone-B TOK04.
for each Zone has to have all Hash range. only Zone-A has satisfied this condition. but, Zone-B and Zone-C have same issue.
Zone-C has Redis slaves. 172.30.201.152 and 172.30.201.153 refer same master.Zone-B has Redis slaves. 172.30.23.209 and 172.30.23.210 refer same master. it is bad configuration.

~~~terminal:
$ kubectl exec -it redis1-cluster-tok02-0 -- redis-cli -c cluster nodes |sort +1
8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 172.30.198.81:6379@16379 myself,master - 0 1565748879000 2 connected 0-5460
82f41aee313dc3cd9ef2e9ecefc6432bf9428375 172.30.198.82:6379@16379 master - 0 1565748881567 1 connected 5461-10922
b295c7aeede1988e1f97e58c32550537a8436c21 172.30.198.83:6379@16379 master - 0 1565748882569 3 connected 10923-16383
0c2453357a09efc64940e44a8147f614f5911a9b 172.30.201.151:6379@16379 slave 82f41aee313dc3cd9ef2e9ecefc6432bf9428375 0 1565748882000 7 connected
09fb2428bcf52dd233c5d5b9729820406c812f7e 172.30.201.152:6379@16379 slave 82f41aee313dc3cd9ef2e9ecefc6432bf9428375 0 1565748880000 8 connected
44697db50d047ae4c5a9fbdd7927a9ccbb15eaa0 172.30.201.153:6379@16379 slave b295c7aeede1988e1f97e58c32550537a8436c21 0 1565748880570 9 connected
3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 172.30.23.208:6379@16379 slave b295c7aeede1988e1f97e58c32550537a8436c21 0 1565748882000 6 connected
8bcc0a07889be0369902e37e56ba3c4a93df8644 172.30.23.209:6379@16379 slave 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 0 1565748883572 4 connected
a305234d01f80d6bb63471ad9d394945275cca7a 172.30.23.210:6379@16379 slave 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 0 1565748880000 5 connected
~~~

Zone-C 172.30.201.151 should be delete and add as slabe of 172.30.198.81
Zone-B 172.30.23.210 should be delete and add as slave of 172.30.198.82

following are summary to change from Redis cluster. it is useful information that IP, Pod name, Redis cluster ID.

Zone-C 172.30.201.151 redis1-cluster-tok05-0  0c2453357a09efc64940e44a8147f614f5911a9b
Zone-B 172.30.23.210  redis1-cluster-tok04-2  a305234d01f80d6bb63471ad9d394945275cca7a

Operation of delete is in following terminal screen. after deleting, deleted redis node have to do reset to add as new member at next.

~~~terminal:
## DELETE REDIS NODE 0c2453357a09efc64940e44a8147f614f5911a9b
$ kubectl exec redis1-cluster-tok02-0 -- redis-cli --cluster del-node $(kubectl get pod redis1-cluster-tok02-0 -o jsonpath='{.status.podIP}'):6379 0c2453357a09efc64940e44a8147f614f5911a9b
>>> Removing node 0c2453357a09efc64940e44a8147f614f5911a9b from cluster 172.30.23.208:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.

## RESET REDIS NODE 0c2453357a09efc64940e44a8147f614f5911a9b
$ kubectl exec -it redis1-cluster-tok05-0 -- redis-cli -c cluster reset
OK
~~~

~~~terminal:
$ kubectl exec redis1-cluster-tok02-0 -- redis-cli --cluster del-node $(kubectl get pod redis1-cluster-tok02-0 -o jsonpath='{.status.podIP}'):6379 a305234d01f80d6bb63471ad9d394945275cca7a
>>> Removing node a305234d01f80d6bb63471ad9d394945275cca7a from cluster 172.30.198.81:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
command terminated with exit code 1

$ kubectl exec -it redis1-cluster-tok04-2 -- redis-cli -c cluster reset
OK
~~~

"Zone-B redis1-cluster-tok04-2 172.30.23.210" is going to add slave as Master redis1-cluster-tok02-2.

~~~terminal:
$ kubectl exec redis1-cluster-tok02-0 -- redis-cli --cluster add-node 172.30.201.151:6379 172.30.198.81:6379 --cluster-slave --cluster-master-id 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f
~~~

"Zone-C redis1-cluster-tok05-0 172.30.201.151" is going to add slave as Master redis1-cluster-tok02-1.

~~~terminal:
$ kubectl exec redis1-cluster-tok02-0 -- redis-cli --cluster add-node 172.30.23.210:6379 172.30.198.81:6379 --cluster-slave --cluster-master-id 82f41aee313dc3cd9ef2e9ecefc6432bf9428375
~~~

Outcome is in next terminal.Redis cluster have all hash range by for each zone. by this countermeasure even if redis cluster have zone obstacle, it never lose hash range.

~~~terminal:
kubectl exec redis1-cluster-tok02-0 -- redis-cli -c cluster nodes |sort +1
8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 172.30.198.81:6379@16379 myself,master - 0 1565750874000 2 connected 0-5460
82f41aee313dc3cd9ef2e9ecefc6432bf9428375 172.30.198.82:6379@16379 master - 0 1565750874246 1 connected 5461-10922
b295c7aeede1988e1f97e58c32550537a8436c21 172.30.198.83:6379@16379 master - 0 1565750872236 3 connected 10923-16383
0c2453357a09efc64940e44a8147f614f5911a9b 172.30.201.151:6379@16379 slave 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 0 1565750873000 2 connected
09fb2428bcf52dd233c5d5b9729820406c812f7e 172.30.201.152:6379@16379 slave 82f41aee313dc3cd9ef2e9ecefc6432bf9428375 0 1565750872000 8 connected
44697db50d047ae4c5a9fbdd7927a9ccbb15eaa0 172.30.201.153:6379@16379 slave b295c7aeede1988e1f97e58c32550537a8436c21 0 1565750872000 9 connected
3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 172.30.23.208:6379@16379 slave b295c7aeede1988e1f97e58c32550537a8436c21 0 1565750873000 6 connected
8bcc0a07889be0369902e37e56ba3c4a93df8644 172.30.23.209:6379@16379 slave 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 0 1565750873246 4 connected
a305234d01f80d6bb63471ad9d394945275cca7a 172.30.23.210:6379@16379 slave 82f41aee313dc3cd9ef2e9ecefc6432bf9428375 0 1565750875250 5 connected
~~~

as you can see at above terminal, Redis masters exist only Zone-A. if Zone-A failed down due to some issue, Redis data is never lost, but Redis cluster service is stopped due to over two redis master are lost.

To reinforce high availability, master node should be migrate other zone. by next command, slave takes over master, it becomes master.

~~~terminal:
$ kubectl exec -it redis1-cluster-tok04-0 -- redis-cli -c cluster failover
$ kubectl exec -it redis1-cluster-tok05-1 -- redis-cli -c cluster failover
~~~

As you can see in next terminal, Redis masters are in each zone, All Hash-Range in for each Zone.
Redis cluster configuration is complete at this step.

~~~terminal:
$ kubectl exec redis1-cluster-tok02-0 -- redis-cli -c cluster nodes |sort +1
8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 172.30.198.81:6379@16379 myself,master - 0 1565751557000 2 connected 0-5460
82f41aee313dc3cd9ef2e9ecefc6432bf9428375 172.30.198.82:6379@16379 slave 09fb2428bcf52dd233c5d5b9729820406c812f7e 0 1565751556000 11 connected
b295c7aeede1988e1f97e58c32550537a8436c21 172.30.198.83:6379@16379 slave 3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 0 1565751558000 10 connected
0c2453357a09efc64940e44a8147f614f5911a9b 172.30.201.151:6379@16379 slave 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 0 1565751558590 2 connected
09fb2428bcf52dd233c5d5b9729820406c812f7e 172.30.201.152:6379@16379 master - 0 1565751559000 11 connected 5461-10922
44697db50d047ae4c5a9fbdd7927a9ccbb15eaa0 172.30.201.153:6379@16379 slave 3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 0 1565751559196 10 connected
3b37f9681ef2c8e34324e1e9b0686f5eb8d3ae9d 172.30.23.208:6379@16379 master - 0 1565751560194 10 connected 10923-16383
8bcc0a07889be0369902e37e56ba3c4a93df8644 172.30.23.209:6379@16379 slave 8aa805d691d3e5bb4ec06f8f597a7ee875297d1f 0 1565751557184 4 connected
a305234d01f80d6bb63471ad9d394945275cca7a 172.30.23.210:6379@16379 slave 09fb2428bcf52dd233c5d5b9729820406c812f7e 0 1565751558189 11 connected
~~~


## References

1. https://redis.io/topics/cluster-tutorial
2. https://kubernetes.io/docs/reference/kubectl/cheatsheet/
3. https://redis.io/clients
4. https://github.com/antirez/redis
5. https://redis.io/topics/rediscli

