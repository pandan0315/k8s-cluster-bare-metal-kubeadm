# Install k8s cluster in vms using kubeadm

## Step 1 Setup (VMs)
|Role|IP|OS|RAM|CPU|
|----|----|----|----|----|
|master|192.168.186.130|Ubuntu 20.04.2|2G|2|
|worker1|192.168.186.131|Ubuntu 20.04.2|8G|2|
|worker1|192.168.186.132|Ubuntu 20.04.2|8G|2|

#### Networking
I will use static networks in this guide:

* node network(virtual networks):               192.168.186.0/24 
* Cluster CIDR:        10.200.0.0/16 

## Step 2  On all nodes (both master and workers)

#### disable firewall
```
sudo ufw disable
```
#### disable swap
```
sudo swapoff -a; 
sudo sed -i '/swap/d' /etc/fstab
```
#### config ssh access
```
sudo apt install openssl-server
```

#### Letting iptables see bridged traffic
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### install runtime(docker)
```
 sudo apt-get update
 sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
Add Docker’s official GPG key:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Set up the stable repository
```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install docker engine:
```
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups:
```
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Restart Docker and enable on boot:
```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### Installing kubeadm, kubelet and kubectl

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


## Step 3 On master

#### Initialize Kubernetes Cluster

```
kubeadm init --apiserver-advertise-address=192.168.186.130 --pod-network-cidr=10.200.0.0/16  --ignore-preflight-errors=all

```
>output

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.186.130:6443 --token 4pzarm.y464hahz4t9179m9 \
	--discovery-token-ca-cert-hash sha256:93c83b3a5c83376c4b7a6fe829184d92d9f26b9b61eac5e52398580fb2197a22 
```

#### Install calico

download calico manifest:
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```
deploy calico:
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf apply -f calico.yaml
```

check master node status:
```
kubectl get nodes 
```
>output:
```
NAME     STATUS     ROLES                  AGE   VERSION
master   Ready   control-plane,master   12h   v1.22.0
```

To be able to run kubectl commands as non-root user:
```
mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Step 4 on workers

Join the cluser(use output of kubeadm init on master node): 

```
kubeadm join 192.168.186.130:6443 --token 4pzarm.y464hahz4t9179m9 \
	--discovery-token-ca-cert-hash sha256:93c83b3a5c83376c4b7a6fe829184d92d9f26b9b61eac5e52398580fb2197a22
```

Create a new join command if forget:

```
kubeadm token create --print-join-command

```


## Verify the cluster

```
kubectl get nodes -o wide
```

>output:
```
NAME      STATUS   ROLES                  AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master    Ready    control-plane,master   12h   v1.22.0   192.168.186.130   <none>        Ubuntu 20.04.2 LTS   5.11.0-27-generic   docker://20.10.8
worker1   Ready    <none>                 71s   v1.22.0   192.168.186.131   <none>        Ubuntu 20.04.2 LTS   5.11.0-27-generic   docker://20.10.8
worker2   Ready    <none>                 33s   v1.22.0   192.168.186.132   <none>        Ubuntu 20.04.2 LTS   5.11.0-27-generic   docker://20.10.8
```






