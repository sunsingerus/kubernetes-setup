# Task: Setup Kebernetes

## Change hostname
Name hosts as required:
```bash
/etc/hostname kub1
/etc/hosts kub1
```

## Install k8s on each node to be used for k8s

### Ubuntu
Ensure `curl` is available
```bash
apt-get update
apt-get install -y apt-transport-https curl
```

Add repo key
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

Add k8s repo
```bash
sudo echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

Install k8s
```bash
sudo apt install -y docker.io # install docker container runtine
sudo apt install -y kubelet kubeadm kubernetes-cni # install k8s components
```

### CentOS

```bash
sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```

Set SELinux in permissive mode (effectively disabling it). This is required to allow containers to access the host filesystem.
```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

```bash
systemctl enable --now kubelet
```

## Setup k8s
```bash
sudo echo "swapoff -a" >> /etc/rc.local
```

```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```bash
sudo reboot
```

## Setup k8s

On a Master node:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

COPY AND SAVE 'join command'

COPY k8s config to your local:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


SETUP MASTER NODE and all the other

```text
[mark-control-plane] Marking the node kubeadm-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubeadm-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
```

List taints on nodes
```bash
kubectl get nodes -o json | jq .items[].spec
```

By default, your cluster will not schedule pods on the master

kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default

### Setup Networking
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```


ON NON-MASTER NODES JOIN THEM INTO CLUSTER
sudo kubeadm join 192.168.74.149:6443 --token ugzi1b.e87yd5ha2yr4viil --discovery-token-ca-cert-hash sha256:ba34bc044b87db492896ab9e357cbd8f4280cb185ef0c9b11233b0ccece480a7

FETCH TOKEN IN CASE OLD IS BAD
sudo kubeadm token create


# Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

=====================================
# New node
sudo vim /etc/rc.local
swapoff -a

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo vim /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main  
sudo apt update
sudo apt install -y  docker.io kubelet kubeadm kubernetes-cni ceph-common python

vim ~/.bashrc
source <(kubectl completion bash)

probably reboot

# Join new node to cluster with JOIN command as `sudo kubeadm join `


