
# CKA Lab Part 3 - Kubernetes installation via kubeadm

Installing Kubernetes ver 1.16.0

## Requirements

- min 2CPU
- docker
- kubeadm 1.16.0

## Install docker

> Source: https://docs.docker.com/install/linux/docker-ce/debian/

Installing the latest version of docker using installation script

```
curl https://get.docker.com | sh
sudo usermod -aG docker $USER
```

## Install Kubelet, kubectl kubeadm

> Source: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/


Adding and updating repositories

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
```

Checking available kibernetes versions

```
curl -s https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages | grep Version | awk '{print $2}'
...
1.16.0-00
1.16.1-00
1.16.2-00
1.16.3-00
1.16.4-00
1.16.5-00
...
```

Installing kubeadm

```
export version=1.16.5-00 && sudo apt-get install -qy kubelet=${version} kubectl=$version kubeadm=${version} --allow-downgrades
sudo apt-mark hold kubelet kubeadm kubectl # prevents from upgrading when apt-get upgrade is perfomed
```

Verifying installed kubeadmin version

```
kubeadm version
```

## Install k8s

> Requirement: min 2CPU on the node

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 # pod network is needed for next step

# If you need to specify a version and skip checks
# sudo kubeadm init --kubernetes-version stable-1.16 --skip-preflight-checks

# Create config (copy from output after installation)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify if node is ready and installed version

```
kubectl get nodes

NAME        STATUS   ROLES    AGE   VERSION
temp-test   Ready    master   29h   v1.16.5
```

If note is node is NOT ready, check kubelet logs 

```
journalctl -u kubelet -n 100
```

## Choose POD network addon

> Source https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network

Install Calico

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
# During kubeadmin init in the previous step, the following option should be passed: --pod-network-cidr=192.168.0.0/16 to kubeadmin init
```

## Verify installed packages

```
dpkg -l | grep kube

ii  kubeadm                               1.16.0-00                         amd64        Kubernetes Cluster Bootstrapping Tool
ii  kubectl                               1.16.0-00                         amd64        Kubernetes Command Line Tool
ii  kubelet                               1.16.0-00                         amd64        Kubernetes Node Agent
```

## Cleanup


If something went wrong, cleanup node first and try again

```
sudo kubeadm reset cleanup-node
```

## Useful commands

```
lsb_release -a # Checking linux distribution and version
journalctl -u kubelet -n 100 # Viewing kubelet logs
dpkg -l | grep kube # Checking kube packages
kubeclt cluster-info
kubeclt cluster-info --debug # Cluster logs
```

