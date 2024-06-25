# Martin-Li-96-k8s-NVIDA-gpu-deployment

For Fedora need install iproute-tc
```
dnf install -y iproute-tc
```
Stop swap
```
sudo dnf remove zram-generator-defaults
```
# HP Master 1

1. Pre-install

   ```
   apt update 
   apt upgrade -y
   apt install -y vim net-tools make 
   # Disable swap
   ```

   

2. Edit \etc\hosts

```bash
vi /etc/hosts

192.168.0.100 vip.k8s.io vip
192.168.0.10 k8s-master1.k8s.io k8s-master1
192.168.0.11 k8s-master2.k8s.io k8s-master2
192.168.0.20 k8s-node1.k8s.io k8s-node1
192.168.0.21 k8s-node2.k8s.io k8s-node2
192.168.0.22 k8s-node4.k8s.io k8s-node4
192.168.0.23 k8s-node3.k8s.io k8s-node3
192.168.0.24 k8s-node5.k8s.io k8s-node5
192.168.0.25 k8s-node6.k8s.io k8s-node6
192.168.0.26 k8s-node7.k8s.io k8s-node7

```

3. Install & Configure Keepalived

   ```
   #vi /etc/keepalived/keepalived.conf
   
   
   global_defs {
     notification_email {
     }
     router_id LVS_DEVEL
     vrrp_skip_check_adv_addr
     vrrp_garp_interval 0
     vrrp_gna_interval 0
   }
   
   vrrp_script haproxy_check {
     script "/bin/bash -c 'if [[ $(netstat -nlp | grep 8443) ]]; then exit 0; else exit 1; fi'"
     interval 2
     weight 2
   }
   
   vrrp_instance haproxy-vip {
     state MASTER
     priority 100
     interface eno1                #vip绑定网卡
     virtual_router_id 60
     advert_int 1
     authentication {
       auth_type PASS
       auth_pass 1111
     }
     unicast_src_ip 192.168.0.10   #当前机器地址
     unicast_peer {
       192.168.0.11                #peer中其它地址
     }
   
     virtual_ipaddress {
       192.168.0.100/24           #vip地址
     }
   
     track_script {
       haproxy_check
     }
   }
   
   
   
   
   
   
   # systemctl enable keepalived
   # systemctl restart keepalived
   
   ```

4. Instal & configure HAproxy

   ```
   #vi /etc/haproxy/haproxy.cfg
   
   global
       log /dev/log    local0
       log /dev/log    local1 notice
       chroot /var/lib/haproxy
       stats socket /var/run/haproxy-admin.sock mode 660 level admin
       stats timeout 30s
       user haproxy
       group haproxy
       daemon
       nbthread 2
   
   defaults
       log     global
       timeout connect 5000
       timeout client  10m
       timeout server  10m
   
   listen  admin_stats
       bind 0.0.0.0:10080  #这里需要设置一个不和其他程序冲突的端口,查看端口是否占用:netstat -anp| grep 10080
       mode http
       log 127.0.0.1 local0 err
       stats refresh 30s
       stats uri /status
       stats realm welcome login\ Haproxy
       stats auth admin:123456
       stats hide-version
       stats admin if TRUE
   
   listen kube-master
       bind 0.0.0.0:16443   #这个8443端口也可以修改为其他端口,主要是keepalived中用于端口检测的
       mode tcp
       option tcplog
       balance source
       server master1 192.168.0.10:6443  check inter 2000 fall 2 rise 2 weight 1  #6443端口为k8s集群中apiserver的端口,需要注意
       server master2 192.168.0.11:6443 check inter 2000 fall 2 rise 2 weight 1
   
   
   
   # systemctl enable haproxy
   # systemctl restart haproxy
   ```

5. Install Docker

   ```
   https://docs.docker.com/engine/install/ubuntu/
   ```

6. Install cri-docker (need install golang 1.22.2)

   ```
   git clone https://github.com/Mirantis/cri-dockerd.git
   
   cd cri-dockerd
   make cri-dockerd
   
   install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
   install packaging/systemd/* /etc/systemd/system
   sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
   systemctl daemon-reload
   systemctl enable --now cri-docker.socket
   
   systemctl restart cri-docker
   ```

   

7. Install k8s

   ```
   https://kubernetes.io/docs/setup/production-environment/container-runtimes/
   
   # k8s 1.30 go-version 1.22.2
   ```

8. Bootstrp k8s with kubeadm

   ```
   kubeadm init --control-plane-endpoint vip.k8s.io:16443 --upload-certs --apiserver-advertise-address 192.168.0.10 --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock
   ```

# Intel Master 2

1. keepalived

   ```
   #vi /etc/keepalived/keepalived.conf
   
   
   global_defs {
     notification_email {
     }
     router_id LVS_DEVEL
     vrrp_skip_check_adv_addr
     vrrp_garp_interval 0
     vrrp_gna_interval 0
   }
   
   vrrp_script haproxy_check {
     script "/bin/bash -c 'if [[ $(netstat -nlp | grep 8443) ]]; then exit 0; else exit 1; fi'"
     interval 2
     weight 2
   }
   
   vrrp_instance haproxy-vip {
     state BACKUP
     priority 100
     interface enp3s0                #vip绑定网卡
     virtual_router_id 60
     advert_int 1
     authentication {
       auth_type PASS
       auth_pass 1111
     }
     unicast_src_ip 192.168.0.10   #当前机器地址
     unicast_peer {
       192.168.0.11                #peer中其它地址
     }
   
     virtual_ipaddress {
       192.168.0.100/24           #vip地址
     }
   
     track_script {
       haproxy_check
     }
   }
   
   
   ```

   

# Nuc k8s node1

1. GPU

   ```
   #add nomodeset to grub
   
   apt install nvidia-driver-535-server
   # nvidia containte tool kit (https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
   
   
   
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
     && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
       sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
       sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
       
       sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
       sudo apt-get update
       
       sudo apt-get install -y nvidia-container-toolkit nvidia-container-runtime nvidia-docker2
       
       sudo nvidia-ctk runtime configure --runtime=docker
       
       sudo systemctl restart docker
   
       
       
       
   
   
   
   ```

# Jetson Nano

```
apt install -y ip*
```



# All in one Master1

```
cat >>/etc/hosts<<EOF 
192.168.0.100 vip.k8s.io vip
192.168.0.10 k8s-master1.k8s.io k8s-master1
192.168.0.11 k8s-master2.k8s.io k8s-master2
192.168.0.20 k8s-node1.k8s.io k8s-node1
192.168.0.21 k8s-node2.k8s.io k8s-node2
192.168.0.22 k8s-node4.k8s.io k8s-node4
192.168.0.23 k8s-node3.k8s.io k8s-node3
192.168.0.24 k8s-node5.k8s.io k8s-node5
192.168.0.25 k8s-node6.k8s.io k8s-node6
192.168.0.26 k8s-node7.k8s.io k8s-node7
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system



swapoff -a
systemctl disable firewalld
systemctl stop firewalld



wget https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.2.linux-amd64.tar.gz
rm -rf go1.22.2.linux-amd64.tar.gz
cat >>/etc/profile<< 'EOF'
export PATH=$PATH:/usr/local/go/bin
EOF

cat >>~/.bashrc<<'EOF'
export KUBECONFIG=/etc/kubernetes/admin.conf
source /etc/profile
EOF
source ~/.bashrc


dnf install -y git keepalived haproxy make
############################
rm -rf /etc/keepalived/keepalived.conf

cat >/etc/keepalived/keepalived.conf<<'EOF'
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}

vrrp_script haproxy_check {
  script "/bin/bash -c 'if [[ $(netstat -nlp | grep 8443) ]]; then exit 0; else exit 1; fi'"
  interval 2
  weight 2
}

vrrp_instance haproxy-vip {
  state MASTER
  priority 100
  interface eno1                #vip绑定网卡
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 192.168.0.10   #当前机器地址
  unicast_peer {
    192.168.0.11                #peer中其它地址
  }

  virtual_ipaddress {
    192.168.0.100/24           #vip地址
  }

  track_script {
    haproxy_check
  }
}


EOF
systemctl restart keepalived

systemctl enable keepalived
############################################


rm -rf /etc/haproxy/haproxy.cfg
cat >/etc/haproxy/haproxy.cfg<<'EOF'
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /var/run/haproxy-admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    nbthread 2

defaults
    log     global
    timeout connect 5000
    timeout client  10m
    timeout server  10m

listen  admin_stats
    bind 0.0.0.0:10080  #这里需要设置一个不和其他程序冲突的端口,查看端口是否占用:netstat -anp| grep 10080
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:123456
    stats hide-version
    stats admin if TRUE

listen kube-master
    bind 0.0.0.0:16443   #这个8443端口也可以修改为其他端口,主要是keepalived中用于端口检测的
    mode tcp
    option tcplog
    balance source
    server master1 192.168.0.10:6443  check inter 2000 fall 2 rise 2 weight 1  #6443端口为k8s集群中apiserver的端口,需要注意
    server master2 192.168.0.11:6443  check inter 2000 fall 2 rise 2 weight 1
EOF


####################################################

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl restart docker
systemctl enable docker

git clone https://github.com/Mirantis/cri-dockerd.git

cd cri-dockerd
make cri-dockerd
install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
install packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl restart cri-docker
systemctl enable --now cri-docker.socket


rm -rf cri-dockerd

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

systemctl restart haproxy
systemctl enable haproxy
systemctl restart haproxy

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet

kubeadm init --control-plane-endpoint vip.k8s.io:16443 --upload-certs --apiserver-advertise-address 192.168.0.10 --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock


#when change api server health restart haproxy
####################################

mkdir -p /usr/local/bin
cat >>/etc/profile<<'EOF'
export PATH=$PATH:/usr/local/bin
EOF
source ~/.bashrc

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm -rf ./get_helm.sh


kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml
    
    
    
```



# All in One Master 2

```
cat >>/etc/hosts<<EOF 
192.168.0.100 vip.k8s.io vip
192.168.0.10 k8s-master1.k8s.io k8s-master1
192.168.0.11 k8s-master2.k8s.io k8s-master2
192.168.0.20 k8s-node1.k8s.io k8s-node1
192.168.0.21 k8s-node2.k8s.io k8s-node2
192.168.0.22 k8s-node4.k8s.io k8s-node4
192.168.0.23 k8s-node3.k8s.io k8s-node3
192.168.0.24 k8s-node5.k8s.io k8s-node5
192.168.0.25 k8s-node6.k8s.io k8s-node6
192.168.0.26 k8s-node7.k8s.io k8s-node7
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system



swapoff -a
systemctl disable firewalld
systemctl stop firewalld



wget https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.2.linux-amd64.tar.gz
rm -rf go1.22.2.linux-amd64.tar.gz
cat >>/etc/profile<< 'EOF'
export PATH=$PATH:/usr/local/go/bin
EOF

cat >>~/.bashrc<<'EOF'
export KUBECONFIG=/etc/kubernetes/admin.conf
source /etc/profile
EOF
source ~/.bashrc

###################################################
dnf install -y git keepalived haproxy make

rm -rf /etc/keepalived/keepalived.conf

cat >/etc/keepalived/keepalived.conf<<'EOF'
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}

vrrp_script haproxy_check {
  script "/bin/bash -c 'if [[ $(netstat -nlp | grep 8443) ]]; then exit 0; else exit 1; fi'"
  interval 2
  weight 2
}

vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface enp3s0                #vip绑定网卡
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 192.168.0.11   #当前机器地址
  unicast_peer {
    192.168.0.10                #peer中其它地址
  }

  virtual_ipaddress {
    192.168.0.100/24           #vip地址
  }

  track_script {
    haproxy_check
  }
}

EOF
systemctl restart keepalived
systemctl enable keepalived

####################################################

rm -rf /etc/haproxy/haproxy.cfg
cat >/etc/haproxy/haproxy.cfg<<'EOF'
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /var/run/haproxy-admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    nbthread 2

defaults
    log     global
    timeout connect 5000
    timeout client  10m
    timeout server  10m

listen  admin_stats
    bind 0.0.0.0:10080  #这里需要设置一个不和其他程序冲突的端口,查看端口是否占用:netstat -anp| grep 10080
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:123456
    stats hide-version
    stats admin if TRUE

listen kube-master
    bind 0.0.0.0:16443   #这个8443端口也可以修改为其他端口,主要是keepalived中用于端口检测的
    mode tcp
    option tcplog
    balance source
    server master1 192.168.0.10:6443  check inter 2000 fall 2 rise 2 weight 1  #6443端口为k8s集群中apiserver的端口,需要注意
    server master2 192.168.0.11:6443 check inter 2000 fall 2 rise 2 weight 1
EOF


##############################################################

sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl restart docker
systemctl enable docker

git clone https://github.com/Mirantis/cri-dockerd.git

cd cri-dockerd
make cri-dockerd
install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
install packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl restart cri-docker
systemctl enable --now cri-docker.socket


rm -rf cri-dockerd

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
systemctl restart haproxy
systemctl enable haproxy


cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet




mkdir -p /usr/local/bin
cat >>/etc/profile<<'EOF'
export PATH=$PATH:/usr/local/bin
EOF
source ~/.bashrc

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm -rf ./get_helm.sh
```



# NUC

```
#subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
#dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
#dnf install -y kernel-* dkms

#wget #https://developer.download.nvidia.com/compute/cuda/12.4.1/local_installers/cuda_12.4.1_#550.54.15_linux.run

#sudo sh cuda_12.4.1_550.54.15_linux.run

#rm -rf cuda_12.4.1_550.54.15_linux.run 
###########################################################################
subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
dnf install -y kernel-* dkms

sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
sudo dnf -y module install nvidia-driver:latest-dkms
#sudo dnf -y install cuda-toolkit-12-5









#################################
cat >>/etc/hosts<<EOF 
192.168.0.100 vip.k8s.io vip
192.168.0.10 k8s-master1.k8s.io k8s-master1
192.168.0.11 k8s-master2.k8s.io k8s-master2
192.168.0.20 k8s-node1.k8s.io k8s-node1
192.168.0.21 k8s-node2.k8s.io k8s-node2
192.168.0.22 k8s-node4.k8s.io k8s-node4
192.168.0.23 k8s-node3.k8s.io k8s-node3
192.168.0.24 k8s-node5.k8s.io k8s-node5
192.168.0.25 k8s-node6.k8s.io k8s-node6
192.168.0.26 k8s-node7.k8s.io k8s-node7
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system



swapoff -a
systemctl disable firewalld
systemctl stop firewalld



wget https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.2.linux-amd64.tar.gz
rm -rf go1.22.2.linux-amd64.tar.gz
cat >>/etc/profile<< 'EOF'
export PATH=$PATH:/usr/local/go/bin
EOF

cat >>~/.bashrc<<'EOF'
source /etc/profile
EOF
source ~/.bashrc


dnf install -y git make
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl restart docker
systemctl enable docker

git clone https://github.com/Mirantis/cri-dockerd.git

cd cri-dockerd
make cri-dockerd
install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
install packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl restart cri-docker
systemctl enable --now cri-docker.socket


rm -rf cri-dockerd


curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo yum-config-manager --enable nvidia-container-toolkit-experimental


sudo dnf install -y nvidia-container-toolkit nvidia-container-runtime nvidia-docker2  

rm -rf /etc/docker/daemon.json

cat >/etc/docker/daemon.json<<'EOF'
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
EOF

    
sudo systemctl restart docker


sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet


```

# Jetson Nano All in One

```
apt update  -y
apt upgrade -y
apt install -y vim net-tools make iptables ufw

systemctl disable ufw
systemctl stop ufw
swapoff -a


cat >>/etc/hosts<<EOF 
192.168.0.100 vip.k8s.io vip
192.168.0.10 k8s-master1.k8s.io k8s-master1
192.168.0.11 k8s-master2.k8s.io k8s-master2
192.168.0.20 k8s-node1.k8s.io k8s-node1
192.168.0.21 k8s-node2.k8s.io k8s-node2
192.168.0.22 k8s-node4.k8s.io k8s-node4
192.168.0.23 k8s-node3.k8s.io k8s-node3
192.168.0.24 k8s-node5.k8s.io k8s-node5
192.168.0.25 k8s-node6.k8s.io k8s-node6
192.168.0.26 k8s-node7.k8s.io k8s-node7
EOF


cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system



wget https://go.dev/dl/go1.22.2.linux-arm64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.2.linux-arm64.tar.gz
rm -rf go1.22.2.linux-arm64.tar.gz
cat >>/etc/profile<< 'EOF'
export PATH=$PATH:/usr/local/go/bin
EOF

cat >>~/.bashrc<<'EOF'
source /etc/profile
EOF

source /etc/profile
source ~/.bashrc



for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get -y remove $pkg; done


# Add Docker's official GPG key:
sudo apt-get update 
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y


sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl restart docker
systemctl enable docker



git clone https://github.com/Mirantis/cri-dockerd.git

cd cri-dockerd
make cri-dockerd
install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
install packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl restart cri-docker
systemctl enable --now cri-docker.socket

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    
    
    
    sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
    
    sudo apt-get update
    
    
    sudo apt-get install -y nvidia-container-toolkit nvidia-container-runtime nvidia-docker2
    
    
rm -rf /etc/docker/daemon.json

cat >/etc/docker/daemon.json<<'EOF'
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
EOF

systemctl restart docker
systemctl restart cri-docker


sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet

#mkdir -p /etc/kubernetes/node-feature-discovery/source.d
#mkdir -p /etc/kubernetes/node-feature-discovery/features.d




```

# k8s-Node2 AMD

```
cat >>/etc/hosts<<EOF 
192.168.0.100 vip.k8s.io vip
192.168.0.10 k8s-master1.k8s.io k8s-master1
192.168.0.11 k8s-master2.k8s.io k8s-master2
192.168.0.20 k8s-node1.k8s.io k8s-node1
192.168.0.21 k8s-node2.k8s.io k8s-node2
192.168.0.22 k8s-node4.k8s.io k8s-node4
192.168.0.23 k8s-node3.k8s.io k8s-node3
192.168.0.24 k8s-node5.k8s.io k8s-node5
192.168.0.25 k8s-node6.k8s.io k8s-node6
192.168.0.26 k8s-node7.k8s.io k8s-node7
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system



swapoff -a
systemctl disable firewalld
systemctl stop firewalld



wget https://go.dev/dl/go1.22.2.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.2.linux-amd64.tar.gz
rm -rf go1.22.2.linux-amd64.tar.gz
cat >>/etc/profile<< 'EOF'
export PATH=$PATH:/usr/local/go/bin
EOF

cat >>~/.bashrc<<'EOF'
source /etc/profile
EOF
source ~/.bashrc


dnf install -y git make
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl restart docker
systemctl enable docker

git clone https://github.com/Mirantis/cri-dockerd.git

cd cri-dockerd
make cri-dockerd
install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
install packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl restart cri-docker
systemctl enable --now cri-docker.socket


rm -rf cri-dockerd



sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

# Generate Token

```
kubeadm init phase upload-certs --upload-certs
```

# CUDNN

```
$ tar -xvf cudnn-linux-x86_64-8.x.x.x_cudaX.Y-archive.tar.xz
$ sudo cp cudnn-*-archive/include/cudnn*.h /usr/local/cuda/include 
$ sudo cp -P cudnn-*-archive/lib/libcudnn* /usr/local/cuda/lib64 
$ sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*

```


# RaspberryPi
Add following command to grub ('linux-default')
```
cgroup_enable=cpuset
cgroup_enable=memory
cgroup_memory=1
```

```
sudo apt install linux-modules-extra-raspi && reboot

```
