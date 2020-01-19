# CKA Lab Part 4 - Cluster
## Lab 1 - Upgrading cluster
### Upgrade a Kubernetes cluster running a previous version to the latest version

### Find the latest stable kubeadm version

```
apt update
apt-cache madison kubeadm
# find the latest 1.17 version in the list
# it should look like 1.17.x-00, where x is the latest patch

kubeadm |  1.17.1-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
```

### Upgrade the kubeadm binary

```
# replace x in 1.17.x-00 with the latest patch version
export version=1.17.1-00  && \
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=${version}
```

### Verify kubeadm version

```
kubeadm version

kubeadm version: &version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.1", GitCommit:"d224476cd0730baca2b6e357d144171ed74192d6", GitTreeState:"clean", BuildDate:"2020-01-14T21:02:14Z", GoVersion:"go1.13.5", Compiler:"gc", Platform:"linux/amd64"}

```

### Check upgrade plan that kubeadm has provided
 
```
sudo kubeadm upgrade plan
```

### Applying upgrade

Apply the upgrade by executing the following command:

```
sudo kubeadm upgrade apply v1.17.1
```

### Upgrade kubelet and kubectl

```
export version=1.17.1-00  && \
sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubectl=$version kubelet=$version
```

### Restart kubelet
```
sudo systemctl restart kubelet
```

### Verify version

```
dpkg -l | grep kube

ii  kubeadm                               1.17.1-00                         amd64        Kubernetes Cluster Bootstrapping Tool
ii  kubectl                               1.17.1-00                         amd64        Kubernetes Command Line Tool
ii  kubelet                               1.17.1-00                         amd64        Kubernetes Node Agent
ii  kubernetes-cni                        0.7.5-00                          amd64        Kubernetes CNI
```

## Lab 2 - Cluster Upgrades - OS Upgrades

### Gracefully remove a node from active service

Verify the node has scheduling disabled

```
kubectl get node # get node name (status Ready) and replace <cp-node-name> below
NAME        STATUS   ROLES    AGE   VERSION
temp-test   Ready    master   82m   v1.17.1
```

Drain the control plane node 

```
kubectl drain temp-test --ignore-daemonsets

node/temp-test cordoned
evicting pod "calico-kube-controllers-5c45f5bd9f-627mx"
evicting pod "coredns-6955765f44-6fzlx"
evicting pod "coredns-6955765f44-fgl7t"
pod/calico-kube-controllers-5c45f5bd9f-627mx evicted
pod/coredns-6955765f44-fgl7t evicted
pod/coredns-6955765f44-6fzlx evicted
node/temp-test evicted
```

Verify that node is draied

```
kubectl get node # Status should be Ready,SchedulingDisabled  
NAME        STATUS                     ROLES    AGE   VERSION
temp-test   Ready,SchedulingDisabled   master   87m   v1.17.1
```

### Gracefully return a node into active service

Uncordon the control plane node:

```
kubectl uncordon temp-test
node/temp-test uncordoned
```

Verify the node has scheduling enabled

```
kubectl get node
NAME        STATUS   ROLES    AGE   VERSION
temp-test   Ready    master   82m   v1.17.1
```

## Lab 3 Backing up Single-node etcd cluster

A snapshot may either be taken from a live member with the etcdctl snapshot save command or by copying the member/snap/db file from an etcd data directory that is not currently used by an etcd process. 

> https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/recovery.md

### Take snapshot using native command

#### Checking if etcdtl is installed

```
etcdctl version

etcdctl version: 3.3.18
API version: 3.3
```

#### Troubleshooting etcdctl: command not found

```
etcdctl version

sudo: etcdctl: command not found
```

Install etcd according to the documentation https://github.com/etcd-io/etcd/releases

##### Etcdctl installation

Check the latest release of etcd https://github.com/etcd-io/etcd/releases an install

```
ETCD_VER=v3.3.18 # Check  https://github.com/etcd-io/etcd/releases

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

# remove etcd tmp files if exist
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd && mkdir -p /tmp/etcd

# download archive
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

# unpack
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1

# move binary
sudo mv /tmp/etcd/etcdctl /usr/local/bin

# cleanup
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd
```

Verify if installed correctly

```
etcdctl version

etcdctl version: 3.3.18
API version: 3.3
```

#### Creating snapshot

Run the following command:

```
sudo ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

It should go fast, if it takes longer check with debug option


#### Troubleshooting: connection error

```
sudo ETCDCTL_API=3 etcdctl snapshot save snapshot.db --debug

INFO: 2020/01/19 13:19:34 transport: loopyWriter.run returning. connection error: desc = "transport is closing"
```

Probably certificates are missing, which are stored in */etc/kubernetes/pki/etcd*
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key

```
sudo ls -la /etc/kubernetes/pki/etcd

/$ ls -la /etc/kubernetes/pki/etcd/
total 40
drwxr-xr-x 2 root root 4096 Jan 18 14:37 .
drwxr-xr-x 3 root root 4096 Jan 18 14:37 ..
-rw-r--r-- 1 root root 1017 Jan 18 14:37 ca.crt
-rw------- 1 root root 1679 Jan 18 14:37 ca.key
-rw-r--r-- 1 root root 1094 Jan 18 15:56 healthcheck-client.crt
-rw------- 1 root root 1679 Jan 18 15:56 healthcheck-client.key
-rw-r--r-- 1 root root 1135 Jan 18 15:56 peer.crt
-rw------- 1 root root 1679 Jan 18 15:56 peer.key
-rw-r--r-- 1 root root 1135 Jan 18 15:56 server.crt
-rw------- 1 root root 1675 Jan 18 15:56 server.key
```

#### Taking snapshot with proving certificates

```
sudo ETCDCTL_API=3 etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
snapshot save snapshot.db

Snapshot saved at snapshot.db
```

### Taking snapshot using docker image

```
export ETCD_TAG=3.4.3-0 && docker run --rm -it \
--net host \
-v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
export ETCD_TAG=3.4.3-0 && docker run --rm -it --net host -v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt snapshot save /etc/kubernetes/tmp/snapshot.db

{"level":"info","ts":1579444204.521137,"caller":"snapshot/v3_snapshot.go:110","msg":"created temporary db file","path":"/etc/kubernetes/tmp/snapshot.db.part"}
{"level":"warn","ts":"2020-01-19T14:30:04.533Z","caller":"clientv3/retry_interceptor.go:116","msg":"retry stream intercept"}
{"level":"info","ts":1579444204.5342348,"caller":"snapshot/v3_snapshot.go:121","msg":"fetching snapshot","endpoint":"127.0.0.1:2379"}
{"level":"info","ts":1579444204.6110442,"caller":"snapshot/v3_snapshot.go:134","msg":"fetched snapshot","endpoint":"127.0.0.1:2379","took":0.089777322}
{"level":"info","ts":1579444204.6111872,"caller":"snapshot/v3_snapshot.go:143","msg":"saved","path":"/etc/kubernetes/tmp/snapshot.db"}
Snapshot saved at /etc/kubernetes/tmp/snapshot.db
```

Where `${ETCD_TAG}` should be set to the version of your etcd image. To check etcd image and tag used by etcd execute

```
export K8S_VERSION=v1.17.1 && sudo kubeadm config images list --kubernetes-version ${K8S_VERSION}
k8s.gcr.io/kube-apiserver:v1.17.1
k8s.gcr.io/kube-controller-manager:v1.17.1
k8s.gcr.io/kube-scheduler:v1.17.1
k8s.gcr.io/kube-proxy:v1.17.1
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5
```

where `K8S_VERSION` is your kubernetes version which you can check using

```
dpkg -l | grep kubeadm
ii  kubeadm                               1.17.1-00   
```

### Verify the snapshot using docker image

```
export ETCD_TAG=3.4.3-0 && docker run --rm -it \
--net host \
-v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
-- cacert /etc/kubernetes/pki/etcd/ca.crt \
--write-out=table snapshot status /etc/kubernetes/tmp/snapshot.db

+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 674abeeb |   185671 |       1454 |     2.5 MB |
+----------+----------+------------+------------+
```

### Verify the snapshot using native command

```
ETCDCTL_API=3 etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--write-out=table snapshot status snapshot.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 65015083 |   179178 |       1568 |     2.5 MB |
+----------+----------+------------+------------+
```

## Lab 4 Backup kubernetes certificates

```
sudo tar -zcf kubernetes-pki.tar.gz /etc/kubernetes/pki/ # z means gzip
```

Verify archive

```
sudo tar -zxf kubernetes-pki.tar.gz
```

## Lab 5 Check etcd cluster health

> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/#setting-up-the-cluster -> POINT 8

```
sudo ETCDCTL_API=3 etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt endpoint health --cluster

https://10.164.0.2:2379 is healthy: successfully committed proposal: took = 7.967712ms
```

Lab 6. Multinode cluster. Backup all hosts

Get cluster nodes

```
kubectl get nodes - wide

NAME        STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION   CONTAINER-RUNTIME
temp-test   Ready    master   29h   v1.17.1   10.164.0.2    <none>        Debian GNU/Linux 9 (stretch)   4.9.0-11-amd64   docker://19.3.5
temp-demo   Ready    <none>   29h   v1.17.1   10.164.0.3    <none>        Debian GNU/Linux 9 (stretch)   4.9.0-11-amd64   docker://19.3.5

```

Specify --endpoints flag by copying nodes internal IPs.

```
sudo ETCDCTL_API=3 etcdctl --endpoints https://10.164.0.2:2379,https://10.164.0.3:2379 --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt snapshot save snapshot.db
Snapshot saved at snapshot.db
Snapshot saved at snapshot.db
```
