# Multi master K8S v1.15 on Centos 7
## 3 master nodes
## 2 worker nodes


### Installing Docker as CRI

#### Install required packages.
sudo yum install yum-utils device-mapper-persistent-data lvm2

#### Add Docker repository.
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

#### Install Docker CE.
sudo yum update && sudo yum install docker-ce-18.06.2.ce

#### Create /etc/docker directory.
sudo mkdir /etc/docker

#### Setup daemon. This step may not be required
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

#### Restart Docker
sudo systemctl daemon-reload
sudo systemctl enable docker
sudo systemctl restart docker


### Setting up K8S 

### setup k8s repo
sudo vi /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

### disable selinux
sudo setenforce 0

sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

### As root add sysctl parameters

confirm module is loaded
lsmod | grep br_netfilter

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

### refresh repo and verify versions

sudo yum repolist
sudo yum list kubeadm --showduplicates

### install kubeadm, kubelet and kubectl
sudo yum install kubeadm-1.15.0-0 kubelet-1.15.0-0 kubectl-1.15.0-0 --disableexcludes=kubernetes -y

sudo systemctl enable --now kubelet

### As root Flush iptables and save 
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

### On master

### kubeadm join 
cat ~/kubeadm-config.yml

apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable-1.15
apiServer:
  certSANs:
  - "192.168.3.15"
controlPlaneEndpoint: "192.168.3.15:6443"
networking:
  podSubnet: 172.24.0.0/16

## With Load balancer kubeadm-config

apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"

sudo kubeadm init --upload-certs --config=kubeadm-config.yml

### create kubeconf

   74  mkdir -p $HOME/.kube
   75  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   76  sudo chown $(id -u):$(id -g) $HOME/.kube/config

### Apply calico for cni

wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml

Search cidr and change it to pod network cidr

sudo kubectl apply -f calico.yaml

### On remaining master node 

execute the kubeadm command for control plane from above output

### On worker nodes

execute the kubeadm command for worker nodes from above output



## Deploying an HA cluster
1. Deploy HAProxy
2. Bootstrap kubernetes with HAproxy as control plane endpoint and apiserver

sudo kubeadm init --control-plane-endpoint 192.168.1.255:6443 --apiserver-advertise-address=192.168.1.255 --pod-network-cidr=172.24.0.0/16 --upload-certs 


## Joining any new master after deployment, after initial token is expired
Create token for new master nodes

```sudo kubeadm token create --print-join-command```

create certificates again for new master node

```sudo kubeadm init phase upload-certs --upload-certs```

On new master node

```sudo kubeadm join 192.168.1.255:6443 --token 5gad61.8ijbnt7ohr2hi2wo --discovery-token-ca-cert-hash sha256:99399e60ef3efbd369c17326cc494703a0a5eeef9c13c549e88e40ddc9ad2f3e --control-plane --certificate-key f08b2079b7c13bc5303e0d6dd11c518ff8825fe1c61ca270361ceda1317938a6```



