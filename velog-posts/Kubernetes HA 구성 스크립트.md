<h3 id="master1">master1</h3>
<pre><code>#!/bin/bash

# 클러스터의 VIP 설정 (Virtual IP)
VIP=&quot;192.168.0.100&quot;  # 가상 IP 설정
INTERFACE=&quot;eth0&quot;  # VIP를 사용하는 네트워크 인터페이스

# 각 마스터 노드에서 Keepalived 및 HAProxy 설치
install_keepalived_haproxy() {
  sudo apt-get update &amp;&amp; sudo apt-get install -y keepalived haproxy

  # HAProxy 설정 파일 작성
  cat &lt;&lt;EOF | sudo tee /etc/haproxy/haproxy.cfg
frontend kube-cluster
    bind *:16443
    option tcplog
    mode tcp
    default_backend kube-cluster-be

backend kube-cluster-be
    mode tcp
    balance roundrobin
    option tcp-check
    option tcplog
    server master1 192.168.0.42:6443 check
    server master2 192.168.0.98:6443 check
    server master3 192.168.0.58:6443 check
EOF

  sudo systemctl restart haproxy
  sudo systemctl enable haproxy

  # Keepalived 설정 파일 작성
cat &lt;&lt;EOF | tee /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        $VIP/16 # This is an example VIP
    }
}
EOF

  sudo systemctl restart keepalived
  sudo systemctl enable keepalived
}

# Keepalived 및 HAProxy 설치
install_keepalived_haproxy

# Docker 설치 전 필수 패키지 업데이트 및 설치
sudo apt-get update &amp;&amp; sudo apt-get upgrade -y

# Docker 설치
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo &quot;deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable&quot; | sudo tee /etc/apt/sources.list.d/docker.list &gt; /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Docker daemon 설정
cat &lt;&lt;EOF | sudo tee /etc/docker/daemon.json
{
    &quot;exec-opts&quot;: [&quot;native.cgroupdriver=systemd&quot;],
    &quot;log-driver&quot;: &quot;json-file&quot;,
    &quot;log-opts&quot;: {
        &quot;max-size&quot;: &quot;100m&quot;
    },
    &quot;storage-driver&quot;: &quot;overlay2&quot;
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker info | grep -i cgroup

# Swap off 및 UFW 방화벽 비활성화
sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab
sudo systemctl disable --now ufw

# containerd 설정
cat &lt;&lt;EOF | sudo tee -a /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo apt update -y
sudo apt install -y containerd apt-transport-https
sudo mkdir /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Kubernetes 네트워크 설정
cat &lt;&lt;EOF | sudo tee -a /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Kubernetes 1.29 설치 (kubeadm, kubelet, kubectl)
mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo apt-mark hold kubelet kubeadm kubectl kubernetes-cni


# Master 1에서 클러스터 초기화 (VIP 사용)
sudo kubeadm init --control-plane-endpoint &quot;192.168.0.100:16443&quot; --upload-certs --pod-network-cidr=192.168.0.0/16



# kubectl 설정 파일 복사
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Control-plane 노드에 스케줄링 가능 설정
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Calico CNI 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml

# Calico 시스템이 준비될 때까지 대기
while [[ $(kubectl get pods -n calico-system | grep -c Running) -lt 3 ]]; do
  echo &quot;Waiting for Calico pods to be ready...&quot;
  sleep 5
done

# Join token 및 명령어 출력
kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs)

# kubectx, kubens, k9s install
wget https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx
sudo install kubectx /usr/local/bin
wget https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubens
sudo install kubens /usr/local/bin
curl -sL https://github.com/derailed/k9s/releases/download/v0.26.3/k9s_Linux_x86_64.tar.gz | tar xfz - -C /usr/local/bin k9s

# .bashrc에 alias 추가
if ! grep -q &quot;alias k='kubectl'&quot; ~/.bashrc; then
    echo &quot;alias k='kubectl'&quot; &gt;&gt; ~/.bashrc
    echo &quot;alias 'k=kubectl' 추가 완료&quot;
else
    echo &quot;'alias k=kubectl' 이미 존재함&quot;
fi

# .bashrc 재적용
source ~/.bashrc
echo &quot;.bashrc 재적용 완료&quot;</code></pre><h3 id="master2-3">master2, 3</h3>
<pre><code>#!/bin/bash

# 클러스터 VIP 설정 (Virtual IP)
VIP=&quot;192.168.0.100&quot;  # 가상 IP 설정
INTERFACE=&quot;eth0&quot;  # VIP가 바인딩될 네트워크 인터페이스

# 필수 패키지 업데이트 및 설치
sudo apt-get update &amp;&amp; sudo apt-get upgrade -y

# Keepalived 및 HAProxy 설치
sudo apt-get install -y keepalived haproxy

# HAProxy 설정 파일 작성
cat &lt;&lt;EOF | sudo tee /etc/haproxy/haproxy.cfg
frontend kube-cluster
    bind *:16443
    option tcplog
    mode tcp
    default_backend kube-cluster-be

backend kube-cluster-be
    mode tcp
    balance roundrobin
    option tcp-check
    option tcplog
    server master1 192.168.0.42:6443 check
    server master2 192.168.0.98:6443 check
    server master3 192.168.0.58:6443 check
EOF

sudo systemctl restart haproxy
sudo systemctl enable haproxy

# Keepalived 설정 파일 작성
cat &lt;&lt;EOF | tee /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 50
    priority 99 #master3 98
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        $VIP/16 # This is an example VIP
    }
}
EOF

sudo systemctl restart keepalived
sudo systemctl enable keepalived

# Docker 설치
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo &quot;deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable&quot; | sudo tee /etc/apt/sources.list.d/docker.list &gt; /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Docker daemon 설정
cat &lt;&lt;EOF | sudo tee /etc/docker/daemon.json
{
    &quot;exec-opts&quot;: [&quot;native.cgroupdriver=systemd&quot;],
    &quot;log-driver&quot;: &quot;json-file&quot;,
    &quot;log-opts&quot;: {
        &quot;max-size&quot;: &quot;100m&quot;
    },
    &quot;storage-driver&quot;: &quot;overlay2&quot;
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker info | grep -i cgroup

# Swap off 및 UFW 방화벽 비활성화
sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab
sudo systemctl disable --now ufw

# containerd 설정
cat &lt;&lt;EOF | sudo tee -a /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo apt update -y
sudo apt install -y containerd apt-transport-https
sudo mkdir /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Kubernetes 네트워크 설정
cat &lt;&lt;EOF | sudo tee -a /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Kubernetes 1.29 설치 (kubeadm, kubelet, kubectl)
mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo apt-mark hold kubelet kubeadm kubectl kubernetes-cni

# Master 2 및 Master 3에서 클러스터 조인 (Master 1에서 받은 명령어를 붙여넣기)
# 클러스터에 참여하는 명령은 Master 1에서 생성한 token 및 cert 키에 따라 변경


kubeadm join 192.168.0.100:16443 --token q7s9rd.90xc8sj6v3tg9e20 \
        --discovery-token-ca-cert-hash sha256:48e7fd3469dc5f7f2c548e6436352c338b1e207303e26c6c4e96c581ade3eb83 \
        --control-plane --certificate-key 5a1d11a1f16fe61f528823b36a71361d7e1148576ea56cb24734c52838a5113b</code></pre><h3 id="worker1-2">worker1, 2</h3>
<pre><code>#!/bin/bash

# Update and install necessary packages
sudo apt-get update &amp;&amp; sudo apt-get upgrade -y

# Docker 설치
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo &quot;deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable&quot; | sudo tee /etc/apt/sources.list.d/docker.list &gt; /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Docker daemon 설정
cat &lt;&lt;EOF | sudo tee /etc/docker/daemon.json
{
    &quot;exec-opts&quot;: [&quot;native.cgroupdriver=systemd&quot;],
    &quot;log-driver&quot;: &quot;json-file&quot;,
    &quot;log-opts&quot;: {
        &quot;max-size&quot;: &quot;100m&quot;
    },
    &quot;storage-driver&quot;: &quot;overlay2&quot;
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker info | grep -i cgroup

# Swap off 및 UFW 방화벽 비활성화
sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab
sudo systemctl disable --now ufw

# containerd 설정
cat &lt;&lt;EOF | sudo tee -a /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo apt update -y
sudo apt install -y containerd apt-transport-https
sudo mkdir /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Kubernetes 네트워크 설정
cat &lt;&lt;EOF | sudo tee -a /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Kubernetes 1.29 설치 (kubeadm, kubelet, kubectl)
mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sudo apt-mark hold kubelet kubeadm kubectl kubernetes-cni

# Worker 노드에서 클러스터 조인 (Master 1에서 받은 명령어를 붙여넣기)
#sudo kubeadm join 192.168.0.57:6443 --token &lt;token&gt; \
#    --discovery-token-ca-cert-hash sha256:&lt;hash&gt;

kubeadm join 192.168.0.100:16443 --token tt98j7.lsxbnmbeivx30fsk \
        --discovery-token-ca-cert-hash sha256:0f3b98b9de523c6ba11c877f2ef1b343342c79374e84516f001526ab7d36b26c</code></pre>