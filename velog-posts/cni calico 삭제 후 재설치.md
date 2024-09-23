<p>calico pod 비정상으로 인해 node NotReady 상태 확인</p>
<h4 id="전제-조건">전제 조건</h4>
<ul>
<li>Kubernetes 클러스터가 정상적으로 구성되어 있고 kubectl 명령어를 사용할 수 있어야 함.</li>
<li>Calico가 현재 CNI 플러그인으로 설치되어 있어야 함.</li>
<li>클러스터가 작동 중이고, 노드와 파드들이 Calico를 통해 네트워크 연결을 하고 있음.</li>
</ul>
<h4 id="1-calico-설정-상태-확인">1. Calico 설정 상태 확인</h4>
<pre><code>kubectl get pods -n kube-system -o wide</code></pre><h4 id="2-calico-cni-삭제">2. Calico CNI 삭제</h4>
<h4 id="2-1-calico-리소스-삭제">2-1. Calico 리소스 삭제</h4>
<pre><code>kubectl delete -f &lt;calico.yaml 파일 경로&gt;</code></pre><p>만약 설치한 calico.yaml 파일이 없다면, kubectl get 명령어로 설치된 리소스를 확인하고 삭제해야 함.</p>
<pre><code>kubectl delete daemonset calico-node -n kube-system
kubectl delete deployment calico-kube-controllers -n kube-system
kubectl delete daemonset calico-typha -n kube-system
kubectl delete podsecuritypolicy calico-node</code></pre><h4 id="2-2cni-설정-파일-제거">2-2.CNI 설정 파일 제거</h4>
<ol>
<li><p>Calico의 네트워크 구성 파일 삭제:</p>
<pre><code>rm -rf /var/run/calico/
rm -rf /var/lib/calico/
rm -rf /etc/cni/net.d/
rm -rf /var/lib/cni/</code></pre></li>
<li><p>kubelet 재시작 &amp; 재부팅</p>
<pre><code>sudo systemctl restart kubelet
reboot</code></pre></li>
</ol>
<h4 id="3-calico-재설치">3. calico 재설치</h4>
<pre><code>curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml</code></pre><h4 id="4-설치-확인">4. 설치 확인</h4>
<pre><code>kubectl get pods -n kube-system
kubectl get nodes</code></pre><h4 id="5-네트워크-테스트">5. 네트워크 테스트</h4>
<pre><code>kubectl run test-pod --image=busybox --restart=Never -- sleep 3600
kubectl exec -it test-pod -- ping &lt;다른 파드 IP&gt;</code></pre><pre><code>kubectl describe pod &lt;test-pod 이름&gt;</code></pre>