# CKA Lab Part 10 - Installation, Configuration & Validation

## Lab 1 - Master bringup using kubeadm

- Create master node VM

- Install a container runtime f.ex. docker.

### Installing Docker

> Source: https://docs.docker.com/install/linux/docker-ce/debian/

Installing the latest version of docker using installation script

```
curl https://get.docker.com | sh
sudo usermod -aG docker $USER
```

### Installing kubeadm, kubelet and kubectl

> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Packages:

- `kubeadm`: the command to bootstrap the cluster.
- `kubelet`: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- `kubectl`: the command line util to talk to your cluster.

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Master node initialization

- Initialise the master assmuming you will leverage the flannel CNI
For flannel to work correctly, you must pass `--pod-network-cidr=10.244.0.0/16` to `kubeadm init`.

```
sudo sudo kubeadm init --pod-network-cidr=10.244.0.0/16 

...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
  
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.164.0.2:6443 --token xm8jmj.3hk5vzhk748am4z6 \
    --discovery-token-ca-cert-hash sha256:a01be46cd44c179cb45b29bc55bce71d33b3bf04cc343d3d96bdf1394ddb159a 
```

- Run

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Deploy a pod network to the cluster (flannel CNI)

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flan
nel.yml

podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```

## Lab 2 - Worker node bringup using kubeadm

- Create worker node VM

- Install a container runtime f.ex. docker.

### Installing Docker

> Source: https://docs.docker.com/install/linux/docker-ce/debian/

Installing the latest version of docker using installation script

```
curl https://get.docker.com | sh
sudo usermod -aG docker $USER
```

### Installing kubeadm, kubelet and kubectl

> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Packages:

- `kubeadm`: the command to bootstrap the cluster.
- `kubelet`: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- `kubectl`: the command line util to talk to your cluster.

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Join the worker to the cluster

```
kubeadm join 10.164.0.2:6443 --token xm8jmj.3hk5vzhk748am4z6 \
    --discovery-token-ca-cert-hash sha256:a01be46cd44c179cb45b29bc55bce71d33b3bf04cc343d3d96bdf1394ddb159a 
```

Verify (from master node)
```
kubectl get nodes

NAME          STATUS   ROLES    AGE   VERSION
temp-test     Ready    master   18m   v1.17.2
temp-test-w   Ready    <none>   98s   v1.17.2
```

## Lab 3 - Manage cluster with Kubeadm

### Generate a new token to add additional worker nodes the cluster

- get current token list

```
kubeadm token list

TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION      EXTRA GROUPS
xm8jmj.3hk5vzhk748am4z6   23h         2020-02-01T09:28:21Z   authentication,signing   The default...   system:bootstrappers:kubeadm:default-node-token
```

- Generate new token

```
kubeadm token create --print-join-command

W0131 09:53:13.604940    7436 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0131 09:53:13.604986    7436 validation.go:28] Cannot validate kubelet config - no validator is available
kubeadm join 10.164.0.2:6443 --token gctk6m.wbs8obm6ccs26exx     --discovery-token-ca-cert-hash sha256:a01be46cd44c179cb45b29bc55bce71d33b3bf04cc343d3d96bdf1394ddb159a 
```


