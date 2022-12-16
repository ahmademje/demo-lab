# Server Used (and technology used)
- Master (kubernetes, docker, Helm)
- Worker (kubernetes, docker)
- nfs (nfs-server)
- ahmad (gitlab ce, jenkins master, sonarqube)

# Install Gitlab CE
> Server : Ahmad

## Update Package
```
sudo apt update
```

## Install Dependencies
```
sudo apt install -y curl openssh-server ca-certificates
sudo apt install -y postfix
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

## Install Gitlab CE
```
sudo apt-get install gitlab-ce
gitlab-ctl start
```
> login credential (user = root, password = $ sudo cat /etc/gitlab/initial_root_password)

# Setup Kubernetes Cluster
 
## Install Docker
> Server : Master, Worker
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
## Install Kubernetes
```
sudo apt update
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

kubectl version --client && kubeadm version
```
## Disable Swap
```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo vim /etc/fstab
sudo swapoff -a
sudo mount -a
free -h
```

## Configure ContainerD
```
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status  containerd

lsmod | grep br_netfilter
```

## Check Kubelet Status
```
sudo systemctl enable kubelet
```

## Init Cluster
> Server : Master
```
sudo kubeadm config images pull

sudo kubeadm init --pod-network-cidr=172.16.0.0/16 --control-plane-endpoint=192.168.56.106 --apiserver-advertise-address=192.168.56.106

mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info
```

## Install Calico CNI to the cluster
> Server: Master
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/custom-resources.yaml -O

vi custom-resources.yaml
-> edit cidr (depends on your pod network)

kubectl create -f custom-resources.yaml

sudo ufw allow 179/tcp
sudo ufw enable
curl https://docs.projectcalico.org/manifests/calico.yaml -O
vi calico.yaml
-> edit CALICO_IPV4POOL_CIDR

kubeadm token create --print-join-command
```
## Join Worker Node to Kubernetes Master
> Server : Worker
```
udo kubeadm join 192.168.56.106:6443 --token tb8rdg.e7o3l3x8rh3fsnfa --discovery-token-ca-cert-hash sha256:8b4f531a02735db80e0dfc291e2b15ce44f70808afd0a6089b9ee81f56b07624 --control-plane 192.168.56.106
```

# Instal NFS Server
> Server : nfs
```
sudo systemctl status nfs-server
sudo apt install nfs-kernel-server nfs-common portmap
sudo start nfs-server
mkdir -p /srv/nfs/mydata 
chmod -R 777 /srv/nfs/

sudo echo "/srv/nfs/mydata  *(rw,sync,no_subtree_check,no_root_squash,insecure)" >> /etc/exports
sudo exportfs -rv
```
## Install nfs-common to every nodes
> Server : master, worker
```
sudo apt install nfs-common
```

## Setup NFS Server on Master Nodes
> Server : master
```
sudo apt install apt-transport-https --yes
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt install -y helm

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=10.8.60.210 --set nfs.path=/srv/nfs/mydata --set storageClass.name=nfs --set storageClass.defaultClass=true -n nfs --create-namespace
```

# Install MetalLB
> Server : master
```
kubectl edit configmap -n kube-system kube-proxy 

...
	strictARP: true
...

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

cat <<EOF | kubectl create -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.120-192.168.56.125
EOF

apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
  interfaces:
  - eth0
```
> source : https://metallb.universe.tf/configuration/_advanced_l2_configuration/

### Test Load Balancer
```
kubectl create deploy nginx-test --image nginx && kubectl expose deploy nginx-test --port 80 --type LoadBalancer
kubectl delete deploy nginx-test && kubectl delete svc nginx-test
```

# Instal Jenkins Server
> Server : ahmad
## Instal Jenkins Server
```
sudo apt install openjdk-11-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

### Configure Jenkins Server

- login using initial password $ sudo cat var/jenkins_home/secrets/initialAdminPassword
- Create User
- Install Sugessted Plugin
- Start Using Jenkins

### Install Plugin (Docker, Gitlab, SonarQube, NodeJS)
- Go to Dashobrd > Manage Jenkins > Manage Plugin > Available Plugin
- Install Plugin Docker, Gitlab, SonarQube, NodeJS, Kubernetes
- Download Now and Install After Restart
- Configure each of plugin

### Configure Jenkins Slave as Kubernetes Worker

#### Configure Cloud
- Go To Dashborad > Manage Jenkins > Configure Clouds > Add New Cloud > select Kubernetes
- Fill Kubernetes Env (Kubernetes and Pod Template)
- Test Connection

#### Test Slave
- Create Pipeline
- 

