# Task: Setup Kebernetes

## Change hostname
Name hosts as required:
```bash
/etc/hostname kub1
/etc/hosts kub1
```

## Install k8s

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
sudo apt install -y docker.io kubelet kubeadm kubernetes-cni
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

==================
Setup k8s

RUN ON MASTER NODE

sudo kubeadm init --pod-network-cidr=10.244.0.0/16

COPY AND SAVE 'join command'

COPY k8s config to your local:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


SETUP MASTER NODE and all the other
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
SETUP NETWORKING
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml


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


