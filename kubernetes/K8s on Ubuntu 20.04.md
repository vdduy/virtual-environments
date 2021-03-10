Install Kubernetes Cluster on Ubuntu 20.04 with kubeadm
My cluster:
```
sudo vi /etc/hosts
172.16.14.101 k8s-master01
172.16.14.102 k8s-master02
172.16.14.103 k8s-master03
172.16.14.104 k8s-worker01
172.16.14.105 k8s-worker02
172.16.14.106 k8s-worker03
```
Virtual IP for Kubernetes API: 172.16.14.100:8443

**Pre-flight**

Turn off swap for all servers
```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```
Update all servers and Reboot
```
sudo apt update
sudo apt -y upgrade && sudo systemctl reboot
```
Once the servers are rebooted, add Kubernetes repository for Ubuntu 20.04 to all the servers.
```
sudo apt update
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
```
prevent updating packages
```
sudo apt-mark hold kubelet kubeadm kubectl
```
confirm versions
```
kubectl version --client && kubeadm version
```
Installing Containerd:
Configure persistent loading of modules
```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
Load at runtime
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
Ensure sysctl params are set
```
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
Reload configs
```
sudo sysctl --system
```
make sure that the br_netfilter module is loaded:
```
lsmod | grep br_netfilter
br_netfilter           22256  0 
bridge                151336  2 br_netfilter,ebtable_broute
```
Enable kubelet service.
```
sudo systemctl enable kubelet
```

Install containerd
```
sudo apt update
sudo apt install -y containerd
```
Configure containerd and start service
```
sudo mkdir -p /etc/containerd
sudo su -
containerd config default  > /etc/containerd/config.toml
```
restart containerd
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Config Keepalived and HAProxy for Kubernetes API**
We need to deploy keepalived and HAProxy on k8s-master01, k8s-master02 and k8s-master03
```
sudo apt install -y keepalived haproxy
```
Keepalived Config
```
sudo vi /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state [MASTER/BACKUP]
    interface ens160
    virtual_router_id 51
    priority [101/100] #101 for master and 100 for slave
    authentication {
        auth_type PASS
        auth_pass 1234567890 #your pass
    }
    virtual_ipaddress {
        172.16.14.100
    }
    track_script {
        check_apiserver
    }
}
```
```
sudo vi /etc/keepalived/check_apiserver.sh
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 172.16.14.100; then
    curl --silent --max-time 2 --insecure https://172.16.14.100:6443/ -o /dev/null || errorExit "Error GET https://172.16.14.100:6443/"
fi
```
Add the following config in haproxy.cfg
```
sudo vi /etc/haproxy/haproxy.cfg 
...

#---------------------------------------------------------------------
# Configure HAProxy for Kubernetes API Server
#---------------------------------------------------------------------
listen stats
  bind    *:9000
  mode    http
  stats   enable
  stats   hide-version
  stats   uri       /stats
  stats   refresh   30s
  stats   realm     Haproxy\ Statistics
  stats   auth      Admin:Password


############## Configure HAProxy Secure Frontend #############
frontend k8s-api-https-proxy
    bind *:8443
    mode tcp
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }
    default_backend k8s-api-https

############## Configure HAProxy SecureBackend #############
backend k8s-api-https
    balance roundrobin
    mode tcp
    option tcp-check
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server k8s-master01 172.16.14.101:6443 check
    server k8s-master02 172.16.14.102:6443 check
    server k8s-master03 172.16.14.103:6443 check
```                                                               

```
sudo systemctl enable haproxy --now
sudo systemctl enable keepalived --now
```

**Initialize master node**
First, we need to deploy k8s-master01 then we will join other servers later.
```
sudo kubeadm config images pull
```

```
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=172.16.14.100:8443
```
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

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 172.16.14.100:8443 --token 91eeaz.fghvf902m0bkmiyf \
    --discovery-token-ca-cert-hash sha256:5c899becd839ba8bb7e98e013e0f928672b53605867031a1ee7661a1d2abe6bd \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.14.100:8443 --token 91eeaz.fghvf902m0bkmiyf \
    --discovery-token-ca-cert-hash sha256:5c899becd839ba8bb7e98e013e0f928672b53605867031a1ee7661a1d2abe6bd 
```
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install network plugin on Master
In this guide we’ll use Calico. You can choose any other supported network plugins.
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

#Lấy certificate key, các master node khác sẽ cần phải add thêm thông số --certificate-key vào
```
sudo kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
9a29a9fbc0a893bc5348139148e811efc7baae09f7b7462c8ea5debcd32e3d7a
```

Then join other master nodes into the cluster
```
sudo kubeadm join 172.16.14.100:8443 --token 91eeaz.fghvf902m0bkmiyf \
    --discovery-token-ca-cert-hash sha256:5c899becd839ba8bb7e98e013e0f928672b53605867031a1ee7661a1d2abe6bd \
    --control-plane \
    --certificate-key 9a29a9fbc0a893bc5348139148e811efc7baae09f7b7462c8ea5debcd32e3d7a
```

Then join all worker nodes into the cluster
```
sudo kubeadm join 172.16.14.100:8443 --token 91eeaz.fghvf902m0bkmiyf \
    --discovery-token-ca-cert-hash sha256:5c899becd839ba8bb7e98e013e0f928672b53605867031a1ee7661a1d2abe6bd
```
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
