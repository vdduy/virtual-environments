# Steps for Installing Kubernetes on CentOS 7

**Step 1: Configure Kubernetes Repository**
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

**Step 2: Install docker, kubelet, kubeadm, and kubectl**
```
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum update -y && yum install docker-ce -y
systemctl enable --now docker

yum install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
```

**Step 4: Configure Firewall**

On the Master Node enter:
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```

Enter the following commands on each worker node:
```
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --reload
```

**Step 5: Update Iptables Settings**

Set the net.bridge.bridge-nf-call-iptables to ‘1’ in your sysctl config file. This ensures that packets are properly processed by IP tables during filtering and port forwarding.
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

**Step 6: Disable SELinux**
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

**Step 7: Disable SWAP**
```
sed -i '/swap/d' /etc/fstab
swapoff -a
```

**Step 8: Add cluster into hosts**
```
cat >>/etc/hosts<<EOF
192.168.10.201 k8s-master
192.168.10.202 k8s-worker01
192.168.10.203 k8s-worker02
192.168.10.204 k8s-worker03
EOF
```

# Deploy a Kubernetes Cluster

**Step 1: Create Cluster with kubeadm**
```
kubeadm init --apiserver-advertise-address=192.168.10.201 --pod-network-cidr=172.16.0.0/16
```
**Step 2: Manage Cluster as Regular User
To start using the cluster you need to run it as a regular user by typing:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Step 3: Set Up Pod Network**

A Pod Network allows nodes within the cluster to communicate. There are several available Kubernetes networking options. Use the following command to install the calico pod network add-on:
```
kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml
```

**Step 4: Check Status of Cluster**
```
#Thông tin cluster
kubectl cluster-info
#Các node trong cluster
kubectl get nodes
#Các pod đang chạy trong tất cả các namespace
kubectl get pods --all-namespaces
kubectl get pods -A
```

**Step 5: Join Worker Node to Cluster**
Trên master node gõ lệnh sau để lấy token
```
kubeadm token create --print-join-command

kubeadm join 192.168.10.201:6443 --token 7wfnrg.r9r3pnw3i7t7ahl2     --discovery-token-ca-cert-hash sha256:378355124d80d0f1a60f7d3ac7b1e32c949e891af3407275654179bab3c2e8c3 
```
Sau khi lấy được token thì gõ lên các worker nodes để join vào kubernetes cluster.
```
ubeadm join 192.168.10.201:6443 --token 7wfnrg.r9r3pnw3i7t7ahl2     --discovery-token-ca-cert-hash sha256:378355124d80d0f1a60f7d3ac7b1e32c949e891af3407275654179bab3c2e8c3 
```
