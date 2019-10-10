 Task: Setup Kubernetes

Table of contents:
1. [Setup with kubeadm](#kubeadm)
1. [Setup with KOPS](#KOPS)
1. [KOPS Cheat Sheet](#KOPS-cheat-sheet)
1. [kubectl](#kubectl)
1. [EKS](#EKS)

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

In case of S3 stroage, it is important to setup S3 permissions to control access to the S3 bucket.

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

**Sidenote**
If you don't have access keys, you can create them from the AWS Management Console
1. Choose **IAM** in AWS services
1. Choose **Users** in navigation panel
1. Choose the name of the user whose access keys you want to create
1. Choose **Security credentials** tab
1. Choose **Create access key** button
1. Copy or download access keys to be used in profile

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

**Create cluster**:
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
**Apply/update cluster** with:
```bash
kops update cluster dev.altinity.k8s.local --yes
```

The cluster access configuration was written to `~/.kube/config`, so `kubectl` can access new cluster.

```bash
kubectl get nodes
```

Validate cluster with:
```bash
kops validate cluster
```

Install addons as described [here](https://github.com/kubernetes/kops/blob/master/docs/addons.md)

**Delete cluster**:

```bash
kops delete cluster dev.altinity.k8s.local --yes
```

**Create additional instance group**
Need to:
1. Add new subnet
1. Add ig into subnet
```bash
# add subnet as
  - cidr: 172.20.64.0/19
    name: us-east-1b
    type: Public
    zone: us-east-1b
```
```bash
# add ig to subnet
kops create ig --subnet=us-east-1b nodes3
kops update cluster dev.altinity.k8s.local
kops update cluster dev.altinity.k8s.local --yes
```

# KOPS Cheat Sheet
**Create Cluster**:
Simple cluster:
```bash
kops create cluster --zones=us-east-1a --yes dev.altinity.k8s.local
```
Specify Kubernetes version:
```bash
kops create cluster --kubernetes-version 1.12.6 --cloud=aws --zones=us-east-1a --yes dev.altinity.k8s.local
```
Additional options:
```bash
kops create cluster \
    --kubernetes-version 1.12.6 \
    --zones us-east-1a,us-east-1b,us-east-1c \
    --node-count 3 \
    --node-size t2.medium \
    --master-zones us-east-1a,us-east-1b,us-east-1c \
    --master-count 3 \
    --master-size t2.micro \
    --yes \
    dev.altinity.k8s.local
```
even more options:
```bash
kops create cluster \
    --name dev.altinity.k8s.local \
    --cloud aws \
    --master-size m4.large \
    --master-zones=us-east-1b,us-east-1c,us-east-1d \
    --node-size m4.xlarge \
    --zones=us-east-1a,us-east-1b,us-east-1c,us-east-1d,us-east-1e,us-east-1f \
    --node-count=3 \
    --kubernetes-version=1.11.6 \
    --vpc=vpc-1234567\
    --network-cidr=10.0.0.0/16 \
    --networking=flannel \
    --authorization=RBAC \
    --ssh-public-key="~/.ssh/kube_aws_rsa.pub" \
    --yes
```

**Delete Cluster**:
```bash
kops delete cluster dev.altinity.k8s.local --yes
```

**Delete instance group**
```bash
kops delete ig --name=dev.altinity.k8s.local nodes --yes
```

**SSH into node**
```bash
kubectl get node -o wide
```
Take a look into **EXTERNAL IP** column. This IP is accessible with SSH and RSA key. We can directly specify RSA key in case there are many of them or just `ssh admin@1.2.3.4` in case there is only one
```bash
ssh -i ~/.ssh/id_rsa admin@1.2.3.4
```
Username - `admin@` in this case - depends on guest OS and would be `admin` for debian and `centos` for centos.

# kubectl
In case you'd like to remove a node from service use `kubectl drain`. This will safely evict all pods from a node. After drain completed, node can be safely maintained.
All pods will be gracefully terminated with respect to PodDistributionBudgets.
Choose node to drain:
```text
kubectl get nodes
NAME                            STATUS   ROLES    AGE   VERSION
ip-172-20-34-156.ec2.internal   Ready    master   46d   v1.10.12
ip-172-20-41-19.ec2.internal    Ready    node     46d   v1.10.12
ip-172-20-59-111.ec2.internal   Ready    node     46d   v1.10.12
```
Let it be `ip-172-20-41-19.ec2.internal`
```bash
kubectl drain ip-172-20-41-19.ec2.internal
```
Drain command is clever enough not to delete local data. So you can see the following report in case some pods have loca data on the node:
```text
node/ip-172-20-41-19.ec2.internal cordoned
error: unable to drain node "ip-172-20-41-19.ec2.internal", aborting command...

There are pending nodes to be drained:
 ip-172-20-41-19.ec2.internal
error: pods with local storage (use --delete-local-data to override): grafana-68cd9ff6c5-jxsxz, prometheus-prometheus-0
```
This is normal, Pods were not terminated and Node is just excluded from scheduling new Pods on it:
```text
kubectl get node
NAME                            STATUS                     ROLES    AGE   VERSION
ip-172-20-41-19.ec2.internal    Ready,SchedulingDisabled   node     46d   v1.10.12
```
and no any Pods are evicted from this Node:
```text
kubectl get all -o wide -n replminpv 
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE
pod/chi-83898a64ab-1-0   1/1     Running   0          46d   100.96.2.11   ip-172-20-59-111.ec2.internal
pod/chi-83898a64ab-2-0   1/1     Running   0          46d   100.96.1.11   ip-172-20-41-19.ec2.internal
```
After node maintenace is completed, run
```bash
kubectl uncordon ip-172-20-41-19.ec2.internal
```
to return node back to service

# EKS

# eksctl
Get `eksctl` binary
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv -v /tmp/eksctl ~/bin
```
Check `eksctl` is available
```bash
eksctl version
```
```console
[ℹ]  version.Info{BuiltAt:"", GitCommit:"", GitTag:"0.1.33"}
```
Ok, `eksctl` is available! Now time to get `aws-iam-authenticator`
```bash
curl --silent --location "https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.4.0/aws-iam-authenticator_0.4.0_$(uname -s)_amd64" -o /tmp/aws-iam-authenticator
chmod a+x /tmp/aws-iam-authenticator
mv /tmp/aws-iam-authenticator ~/bin
```

Ensure `command` in `~/.kube/config` is specified as:
```yaml
      command: aws-iam-authenticator
```

Simple cluster management
```bash
eksctl create cluster --name=ekscluster1 --nodes=3 --node-ami=auto --region=us-east-1
```
```bash
eksctl get cluster --name=ekscluster1 --region=us-east-1
```
```bash
eksctl delete cluster --name=ekscluster1 --region=us-east-1
```

Advanced cluster management options:

```bash
eksctl create cluster \
    --name testcluster \
    --version 1.12 \
    --region us-west-2 \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 1 \
    --nodes-max 4 \
    --node-ami auto
```
```bash
eksctl get cluster --name=testcluster --region=us-west-2
```
```bash
eksctl delete cluster --name=testcluster --region=us-west-2
```

For all available options see:
```bash
eksctl create cluster --help
```
```bash
kubectl get node
```
```console
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-17-77.ec2.internal    Ready    <none>   19m   v1.12.7
ip-192-168-25-67.ec2.internal    Ready    <none>   19m   v1.12.7
ip-192-168-40-148.ec2.internal   Ready    <none>   18m   v1.12.7
```

Amazon EKS worker nodes run in your AWS account and connect to your cluster's control plane via the API server endpoint and a certificate file that is created for your cluster. 
Instance type `m5.large` is used by default for worker nodes. Instance type can be specified as `--node-type t3.medium` param for `eksctl create cluster` command.
A special [Worker group](https://docs.aws.amazon.com/eks/latest/userguide/launch-workers.html) used to launch worker nodes can be created with:
```bash
eksctl create nodegroup \
    --cluster testcluster \
    --version auto \
    --name testcluster-workers \
    --node-type t3.medium \
    --node-ami auto \
    --nodes 3 \
    --nodes-min 1 \
    --nodes-max 4
```



```bash
source <(eksctl completion bash)
```
EKS cluster is specified by `.yaml` file. Let's download template
```bash
wget https://eksworkshop.com/eksctl/launcheks.files/eksworkshop.yml.template
```

