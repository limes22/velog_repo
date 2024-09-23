<p>목표</p>
<ol>
<li>Kubernetes 1.29 version으로 설치 후 Kubeflow, airflow 호환되는 버전 체크
환경</li>
<li>Ubuntu 20.04.1</li>
<li>K8s 1.29 </li>
<li>Airflow 2.10.0</li>
<li>Kubeflow 1.9.0</li>
</ol>
<p>K8s 1.27 , Airflow 2.10.0, Kubeflow 1.8.0</p>
<ol>
<li>K8S 1.29 version 설치 (kubeadm)</li>
<li>Docker 설치
$ sudo apt-get install -y <br />ca-certificates <br />curl <br />gnupg <br />lsb-release</li>
</ol>
<p>$curl -fsSL <a href="https://download.docker.com/linux/ubuntu/gpg">https://download.docker.com/linux/ubuntu/gpg</a> | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg</p>
<p>$ echo <br />  &quot;deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] <a href="https://download.docker.com/linux/ubuntu">https://download.docker.com/linux/ubuntu</a> <br />  $(lsb_release -cs) stable&quot; | sudo tee /etc/apt/sources.list.d/docker.list &gt; /dev/null</p>
<p>$ sudo apt-get update -y</p>
<p>$ sudo apt-get install -y docker-ce docker-ce-cli containerd.ioInstall </p>
<ol start="2">
<li>Docker daemon 설정 변경
$ cat &lt;&lt; EOF | sudo tee /etc/docker/daemon.json
{
&quot;exec-opts&quot;: [&quot;native.cgroupdriver=systemd&quot;],
&quot;log-driver&quot;: &quot;json-file&quot;,
&quot;log-opts&quot;: {
&quot;max-size&quot;: &quot;100m&quot;
},
&quot;storage-driver&quot;: &quot;overlay2&quot;
}
EOF</li>
</ol>
<p>$ sudo systemctl enable docker
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker</p>
<p>$ sudo docker info | grep -i cgroup</p>
<ol start="3">
<li><p>ubuntu swap off 설정
$ sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab</p>
</li>
<li><p>ubuntu ufw 방화벽 off
$ sudo systemctl disable --now ufw</p>
</li>
<li><p>containerd 설정 변경
$ cat &lt;&lt; EOF | sudo tee -a /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF</p>
</li>
</ol>
<p>$ sudo modprobe overlay
$ sudo modprobe br_netfilter</p>
<p>$ sudo apt update -y
$ sudo apt install -y containerd apt-transport-https
$ sudo mkdir /etc/containerd
$ sudo containerd config default &gt; /etc/containerd/config.toml
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd</p>
<ol start="6">
<li>kubernetes 네트워크 포워딩 설정 변경
$ cat &lt;&lt; EOF | sudo tee -a /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF</li>
</ol>
<p>$ sudo sysctl –system</p>
<ol start="7">
<li>Kubernetes 설치
$ curl -fsSL <a href="https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key">https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key</a> | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] <a href="https://pkgs.k8s.io/core:/stable:/v1.29/deb/">https://pkgs.k8s.io/core:/stable:/v1.29/deb/</a> /' | sudo tee /etc/apt/sources.list.d/kubernetes.list</li>
</ol>
<p>$ apt-get update
$ apt-get install -y kubelet kubeadm kubectl kubernetes-cni
$ apt-mark hold kubelet kubeadm kubectl kubernetes-cni
$ systemctl restart kubelet 
$ systemctl status kubelet
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16</p>
<p>$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ export KUBECONFIG=/etc/kubernetes/admin.conf</p>
<ol start="8">
<li><p>Calico CNI 설치
$ kubectl create -f <a href="https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml">https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml</a>
$ kubectl create -f <a href="https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml">https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml</a>
$ watch kubectl get pods -n calico-system
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-</p>
</li>
<li><p>Local-path-provisioner 설치
$ kubectl apply -f <a href="https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml">https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml</a>
$ kubectl patch storageclass local-path  -p '{&quot;metadata&quot;: {&quot;annotations&quot;:{&quot;storageclass.kubernetes.io/is-default-class&quot;:&quot;true&quot;}}}'
$ kubectl get sc</p>
</li>
<li><p>K9S 설치
$ curl -sL <a href="https://github.com/derailed/k9s/releases/download/v0.26.3/k9s_Linux_x86_64.tar.gz">https://github.com/derailed/k9s/releases/download/v0.26.3/k9s_Linux_x86_64.tar.gz</a> | tar xfz - -C /usr/local/bin k9s
<img alt="" src="https://velog.velcdn.com/images/limes22/post/00f8963e-6512-40b8-bd32-cbcef95c0236/image.png" /></p>
</li>
</ol>
<ol start="2">
<li><p>Airflow 설치 (helm) (2.10.0 version)</p>
</li>
<li><p>Helm install
$ curl -fsSL -o get_helm.sh <a href="https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3">https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3</a>
$ chmod 700 get_helm.sh
$ ./get_helm.sh</p>
</li>
<li><p>Helm airflow repo를 추가
$ helm repo add apache-airflow <a href="https://airflow.apache.org">https://airflow.apache.org</a></p>
</li>
<li><p>Value file 저장
$ helm show values apache-airflow/airflow &gt; value.yaml</p>
</li>
<li><p>Value file 수정
$ vi value.yaml</p>
</li>
</ol>
<p><img alt="" src="https://velog.velcdn.com/images/limes22/post/9bfd19a6-45d7-48d1-ae24-6ab0e46b0e46/image.png" /></p>
<p> <img alt="" src="https://velog.velcdn.com/images/limes22/post/85fde744-4dc5-4783-90a3-3aca1eb976b2/image.png" /></p>
<ol start="5">
<li>Airflow install
$ helm upgrade --install airflow apache-airflow/airflow --namespace airflow --create-namespace -f value.yaml</li>
</ol>
<p><img alt="" src="https://velog.velcdn.com/images/limes22/post/5309e7e1-aa57-4a02-af19-6b33298e2aaa/image.png" /></p>
<p>후속단계</p>
<ol>
<li><p>Svc 를 node port로 변환
$ k edit svc airflow-webserver -n airflow
<img alt="" src="https://velog.velcdn.com/images/limes22/post/617801ca-81b8-430d-bf72-128087fb8332/image.png" /></p>
</li>
<li><p>Ui 접속 및 확인
<img alt="" src="https://velog.velcdn.com/images/limes22/post/f8c8aff9-5097-4860-929c-46caa6bdaff2/image.png" /></p>
</li>
<li><p>Git sync 활성화
레포지토리 생성
<img alt="" src="https://velog.velcdn.com/images/limes22/post/0297e0ab-2312-4e05-a6f8-6518e716a625/image.png" /></p>
</li>
</ol>
<p>Ssh key 생성
$ ssh-keygen -t rsa -b 4096 -C <a href="mailto:your_email@example.com">your_email@example.com</a>
$ pbcopy &lt; airflow_ssh_key.pub</p>
<p>Deploy keys 설정
repositry setting 페이지로 넘어가면, 왼쪽 사이드 바에 Deploy keys 항목이 있다.
복사한 key를 붙여넣고 Allow write access를 허용해준 후 키를 등록한다.
 <img alt="" src="https://velog.velcdn.com/images/limes22/post/c6520ffd-8ec8-4311-803c-1b0335d12185/image.png" /></p>
<p>K8s Secret 생성
$ kubectl create secret generic airflow-git-ssh-secret <br />  --from-file=gitSshKey=/path/to/.ssh/airflow_ssh_key <br />  --from-file=id_ed25519.pub=/path/to/.ssh/airflow_ssh_key.pub <br />-n airflow
$ kubectl get secret -n airflow</p>
<p>Value file 수정</p>
<p><img alt="" src="https://velog.velcdn.com/images/limes22/post/b6072e1b-d686-4b7e-8406-1b73082145db/image.png" /></p>
<p> <img alt="" src="https://velog.velcdn.com/images/limes22/post/a1a079a9-530d-4da7-958f-708247299f82/image.png" /></p>
<p> <img alt="" src="https://velog.velcdn.com/images/limes22/post/d4f5660a-65d8-440f-a1c6-9ad8b85f3ca7/image.png" /></p>
<p>•    enabled: 해당 기능을 사용할 것인지에 대한 여부
•    repo: sync에 사용할 github repository 주소
•    branch: repository의 어떤 branch를 sync 할 것인지
•    depth: root 디렉토리의 depth. 디렉토리 자체를 dags 디렉토리로 사용할 것이므로 1로 놔둔다.
•    subPath: 추가로 사용할 경로 path. 참조한 자료에서는 테스트 코드를 subPath에 추가해서 사용한다.
•    sshKeySecret: 앞서 작성한 k8s secret의 이름</p>
<p>airflow upgrade
$ helm upgrade --install airflow apache-airflow/airflow -n airflow -f value.yaml</p>
<ol start="3">
<li><p>Kubeflow 설치 (1.9.0 version)</p>
</li>
<li><p>kustomize install
$ curl -s &quot;<a href="https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh&amp;quot">https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh&quot;</a>  | bash
$ sudo mv kustomize /usr/local/bin</p>
</li>
<li><p>Linux kernel subsystem changes to support many pods
$ sudo sysctl fs.inotify.max_user_instances=2280
$ sudo sysctl fs.inotify.max_user_watches=1255360</p>
</li>
<li><p>Kubeflow 1.9.0 version install 
$ git clone <a href="https://github.com/kubeflow/manifests.git">https://github.com/kubeflow/manifests.git</a>
$ cd manifests/
$ while ! kustomize build example | kubectl apply -f -; do echo &quot;Retrying to apply resources&quot;; sleep 20; done</p>
</li>
<li><p>Ui 접속 확인
$ kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
<img alt="업로드중.." src="blob:https://velog.io/128b9f5d-22c5-4422-ab14-21f1129026d4" /></p>
</li>
</ol>