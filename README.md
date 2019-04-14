# Task: Setup Kubernetes

Table of contents:
 1. [Setup with kubeadm](#kubeadm)
 1. [Setup with KOPS](#KOPS)

# Prerequisites
Set of nodes to install Kubernetes on. Virtual Machives will do.

# kubeadm

## Change hostname OPTIONAL
We may want to name kubernetes hosts in order to tell them apart from other nodes. In this case, on each k8s node we may need to setup hostname and DNS name
 - Edit `/etc/hostname` and set preferred `hostname`, ex: `kub1`:
```bash
sudo vim /etc/hostname
```
 - Edit `/etc/hosts` and set preferred DNS name, ex: `kub1`:
```bash
sudo vim /etc/hosts
```

## Install k8s on each node to be used for k8s

### Ubuntu
Ensure `curl` is available
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https curl
```

Add repo key
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

Add k8s repo
```bash
sudo bash -c "echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' >> /etc/apt/sources.list.d/kubernetes.list"
sudo apt update
```

**Install k8s:**
Install `Docker` to run containers:
```bash
sudo apt install -y docker.io # install docker container runtime

```
Install k8s components. This step may be different in case of k8s version you'd like to have
 - Latest version
```bash
sudo apt install -y kubernetes-cni kubectl kubelet kubeadm # install k8s components
```
 - Specific version, say `1.10.x`
```bash
sudo apt install -y kubernetes-cni=0.6.0-00
sudo apt install -y kubectl=1.10.0-00
sudo apt install -y kubelet=1.10.0-00
sudo apt install -y kubeadm=1.10.0-00
```
 - Specific version, say `1.11.x`
```bash
sudo apt install -y kubernetes-cni=0.6.0-00
sudo apt install -y kubectl=1.11.0-00
sudo apt install -y kubelet=1.11.0-00
sudo apt install -y kubeadm=1.11.0-00
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

END OF CentOS setup, continue.

## Setup k8s
Kubernetes requires to have swap turned off, so let's specify this
```bash
sudo vim /etc/rc.local
```
Insert into `rc.local`
```bash
swapoff -a
```

Insert `kubectl` autocomplete script - very useful feature, just press `tab` while editing `kubectl` command and see what autocomplete can do.
```bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```bash
sudo reboot
```

## Setup k8s

On a Master node init kubernetes cluster with default settings as:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Or, in case you'd like to setup specific Kubernetes version or have issues with preflight system versification:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.10.0 
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.10.0 --ignore-preflight-errors=SystemVerification
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=SystemVerification
```

This command may take a while to complete. In case of success, we'll see something like:

```text
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.74.149:6443 --token gonf7n.0w7x9huih0gbqcsj --discovery-token-ca-cert-hash sha256:3871b941076a8b6524bfb5407a4e6315a2663bbdb908986a8e982805b7b82f37

```

**Copy and save** 'join command'
```bash
sudo kubeadm join 192.168.74.149:6443 --token gonf7n.0w7x9huih0gbqcsj --discovery-token-ca-cert-hash sha256:3871b941076a8b6524bfb5407a4e6315a2663bbdb908986a8e982805b7b82f37
```

And we would like to fetch `config` file from `/etc/kubernetes/admin.conf` to your local `$HOME/.kube` folder on master node, so it can easily be propagated further to users.
Copy k8s config file  This can be done with SSH in case you'd like to access k8s cluster from other machine:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After that, you can easily get `config` from master node with `scp` as:
```bash
scp <MASTER_NODE>:/home/<USER>/.kube/config ~/.kube/config
```

Note the following lines reported by `kubeadm init` command:
```text
[mark-control-plane] Marking the node kubeadm-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kubeadm-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
```
This means that `master` node has label `node-role.kubernetes.io/master=''` and taint `node-role.kubernetes.io/master:NoSchedule`.
With such a taint master node will not run any pods. So, in case we'd like to have pods running on all nodes of k8s cluster - including master - we need to clear this taint.

List taints on nodes in JSON form
```bash
kubectl get nodes -o json | jq .items[].spec
```
```json
{
  "podCIDR": "10.244.0.0/24",
  "taints": [
    {
      "effect": "NoSchedule",
      "key": "node-role.kubernetes.io/master"
    },
    {
      "effect": "NoSchedule",
      "key": "node.kubernetes.io/not-ready"
    }
  ]
}

```
In yaml form
```bash
kubectl get nodes -o yaml
```
and see
```yaml
    labels:
      beta.kubernetes.io/arch: amd64
      beta.kubernetes.io/os: linux
      kubernetes.io/hostname: kub1
      node-role.kubernetes.io/master: ""

```
```yaml
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
```


By default, your cluster will not schedule pods on the master.
Clear taint with:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Assign `cluster-admin` to `kube-system:default` account
```bash
kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
```

### Setup Networking
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
It'll take some time so setup networking and after several minutes we'll have k8s cluster of one node ready:

```bash
kubectl get node
```
```text
NAME   STATUS   ROLES    AGE   VERSION
kub1   Ready    master   16m   v1.13.4
```

## Join node into existing k8s cluster
Now we can setup k8s on other nodes.

Join node into k8s cluster with `join` command, saved earlier
```bash
sudo kubeadm join 192.168.74.149:6443 --token gonf7n.0w7x9huih0gbqcsj --discovery-token-ca-cert-hash sha256:3871b941076a8b6524bfb5407a4e6315a2663bbdb908986a8e982805b7b82f37
```
We should see something like:
```text
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```
FETCH TOKEN IN CASE OLD IS BAD
sudo kubeadm token create

```bash
kubectl get node
```
```text
NAME   STATUS     ROLES    AGE   VERSION
kub1   Ready      master   56m   v1.13.4
kub2   NotReady   <none>   31s   v1.13.4
```
Give it some time to setup
```text
NAME   STATUS   ROLES    AGE   VERSION
kub1   Ready    master   56m   v1.13.4
kub2   Ready    <none>   70s   v1.13.4
```

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

# KOPS

Download `kops`
```bash
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
mv kops ~/bin/kops
```

`kops` store state of the cluster it manages in its own storage. We have two options where to keep cluster state:
 - either locally (in files) or 
 - in S3 storage.

In case of S3 stroage, it is importnant to setup S3 permissions to control access to the S3 bucket.

Let’s use `altinity-kops-state-store` as the S3 bucket name. Create this bucket either in GUI or with CLI tool:
```bash
aws s3api create-bucket --bucket altinity-kops-state-store --region us-east-1
```
Specify bucket to be used as storage to `kops`:
```bash
export KOPS_STATE_STORE=s3://altinity-kops-state-store
```
and then `kops` will use this S3 bucket as default storage.

Specify and export `AWS_PROFILE` env var which references section in `~/.aws/credentials` file with `aws_access_key_id` and `aws_secret_access_key`. Example of the section:
```ini
[altinity]
aws_access_key_id = XXX
aws_secret_access_key = YYY
```
So in our case `AWS_PROFILE` env var would point to `altinity` section and we have the whole statement as:
```bash
export AWS_PROFILE=altinity
```
Now, we are ready to manage clusters!

More docs on how to setup profile and users: [https://github.com/kubernetes/kops/blob/master/docs/aws.md](https://github.com/kubernetes/kops/blob/master/docs/aws.md)

## Configure DNS
If you are using Kops 1.6.2 or later, then DNS configuration is optional.
Instead, a gossip-based cluster can be easily created. 
The only requirement to trigger this is to have the cluster name end with `.k8s.local`.

For test run it is sufficient to have `.k8s.local` domain.

Otherwise [setup DNS configuration](https://github.com/kubernetes/kops/blob/master/docs/aws.md#configure-dns)

## Manage clusters

For a gossip-based cluster, make sure the name ends with `.k8s.local`

Create cluster:
```bash
kops create cluster --zones=us-east-1a dev.altinity.k8s.local
```
List clusters:
```bash
kops get cluster
```
Edit cluster:
```bash
kops edit cluster dev.altinity.k8s.local
```
Edit node instance group - specify node parameters (RAM, CPU, etc):
```bash
kops edit ig --name=dev.altinity.k8s.local nodes
```
Edit your master instance group - specify master node parameters (RAM, CPU, etc):
```bash
kops edit ig --name=dev.altinity.k8s.local master-us-east-1a
```
Apply/update cluster with:
```bash
kops update cluster dev.altinity.k8s.local --yes
```

The cluster access configuration was written to `~/.kube/config`, so `kubectl` can access new cluster.

```bash
kubectl get nodes
```

## Delete cluster

```bash
kops delete cluster dev.altinity.k8s.local --yes
```

