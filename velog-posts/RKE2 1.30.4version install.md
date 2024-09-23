<ol>
<li>서버노드 설치(1st node) 
RKE2 설치를 위해 VM 2대를 준비합니다. </li>
</ol>
<p><img alt="" src="https://velog.velcdn.com/images/limes22/post/8490a73d-9b6b-4b26-9c85-c501a939177b/image.png" /></p>
<ol start="2">
<li>스크립트 설치 및 서비스 시작
RKE2 provides an installation script that is a convenient way to install it as a service on systemd based systems.<br />hostname 에 special character (_) 등이 있으면, RKE2 서버 구동/설치에 문제가 있을 수 있음</li>
</ol>
<p>#주의 :  마스터 3대모두 설치 하고 싶으면 아래 명령어 동시에 실행 하세요 !</p>
<pre><code>$ swapoff -a (/etc/fstab swap comment out)
$ apt-get upgrade -y
$ apt-get dist-upgrade -y
$ apt-get update -y
$ systemctl stop ufw &amp;&amp; ufw disable &amp;&amp; iptables -F
RKE2 설치 :
$ curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=&quot;server&quot; sh -
$ systemctl enable rke2-server.service
$ systemctl start rke2-server.service</code></pre><p>Rancher provides the several CNI network providers for Kubernetes clusters:</p>
<p>Canal, Flannel, Calico and Weave. In Rancher, Canal is the default CNI network provider combined with Flannel and VXLAN encapsulation.</p>
<p>Rancher는 Kubernetes 클러스터를 위한 여러 CNI 네트워크 공급자를 제공합니다.</p>
<p>Canal, Flannel, Calico and Weave. Rancher에서 Canal은 Flannel 및 VXLAN 캡슐화와 결합된 
기본 CNI 네트워크 공급자입니다.</p>
<p>$ mkdir -p /etc/rancher/rke2
$ vi /etc/rancher/rke2/config.yaml
cni: &quot;calico&quot;</p>
<h1 id="cni-calico---작성안하게-되면-canal-로-지정됩니다">cni: &quot;calico&quot; &lt;==  작성안하게 되면.. canal 로 지정됩니다.</h1>
<ol start="3">
<li>worker 노드 설치<pre><code>curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=&quot;agent&quot; sh -
</code></pre></li>
</ol>
<p>systemctl enable rke2-agent.service
mkdir -p /etc/rancher/rke2/
vim /etc/rancher/rke2/config.yaml</p>
<p>server: https://:9345
token: </p>
<p>systemctl start rke2-agent.service</p>
<p>journalctl -u rke2-agent -f</p>
<pre><code> master node 에서 /var/lib/rancher/rke2/server/node-token 에서 토큰 확인 후 작성하면 됩니다.

4. 노드 확인
</code></pre><p>export KUBECONFIG=/etc/rancher/rke2/rke2.yaml PATH=$PATH:/var/lib/rancher/rke2/bin</p>
<p>kubectl get nodes</p>
<pre><code>
Rancher 설치 방법!

Cert-Manager 설치
</code></pre><p>kubectl apply -f <a href="https://github.com/jetstack/cert-manager/releases/download/v1.15.3/cert-manager.yaml">https://github.com/jetstack/cert-manager/releases/download/v1.15.3/cert-manager.yaml</a>
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
kubectl get pods --namespace cert-manager</p>
<pre><code>HELM 설치
</code></pre><p>curl -fsSL -o get_helm.sh <a href="https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3">https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3</a>
chmod 700 get_helm.sh
./get_helm.sh</p>
<pre><code>
Rancher UI 배포 ( cattle-system )
</code></pre><p>kubectl create namespace cattle-system
helm repo add rancher-stable <a href="https://releases.rancher.com/server-charts/stable">https://releases.rancher.com/server-charts/stable</a>
helm repo update
helm search repo rancher-stable
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=114.110.160.164.nip.io --set replicas=1</p>
<pre><code>


# Rancher Server 접속해봅시다. 

![](https://velog.velcdn.com/images/limes22/post/da6c954a-52df-42c0-ab5c-cab0fb036230/image.png)

</code></pre>