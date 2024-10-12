<h4 id="1-문제-상황">1. 문제 상황</h4>
<p>Kubernetes 마스터 노드(1, 2, 3)에서 VIP(예: 16443 포트)로 telnet 연결이 되지 않는 문제가 발생함.</p>
<p>keepalived, haproxy, containerd, kubelet 서비스를 재시작한 후 VIP로의 연결이 다시 가능해짐.</p>
<h4 id="2-원인-분석">2. 원인 분석</h4>
<h4 id="21-keepalived의-장애-복구">2.1 Keepalived의 장애 복구</h4>
<p>Keepalived는 VIP를 관리하고 마스터 노드 간에 고가용성을 제공하는 역할을 합니다.</p>
<p>하나 이상의 마스터 노드에서 Keepalived가 비정상적인 상태가 되었거나 VIP를 유지하지 못하는 경우 VIP로의 연결이 실패할 수 있습니다.</p>
<p>모든 노드에서 Keepalived를 재시작함으로써 VIP가 정상적으로 다시 활성화되고 해당 노드로 트래픽이 라우팅되었을 가능성이 있습니다.</p>
<h4 id="22-haproxy-설정-및-동기화-문제">2.2 HAProxy 설정 및 동기화 문제</h4>
<p>HAProxy는 VIP를 통해 들어오는 트래픽을 여러 노드에 분산시키는 로드 밸런서 역할을 합니다.</p>
<p>HAProxy 설정에 문제가 있거나, 설정 변경 후 적용되지 않았다면 트래픽이 올바르게 분산되지 않을 수 있습니다.</p>
<p>HAProxy를 재시작하면서 설정이 다시 적용되고 VIP에 대한 로드 밸런싱이 정상화되었을 가능성이 있습니다.</p>
<h4 id="23-containerd-및-kubelet의-상태-문제">2.3 Containerd 및 Kubelet의 상태 문제</h4>
<p>Containerd와 Kubelet은 컨테이너 관리와 Kubernetes 워커 노드의 핵심 서비스입니다.</p>
<p>이들이 비정상적인 상태라면 Kubernetes 마스터 노드에서 필요한 컨테이너들이 실행되지 않아 VIP 접근이 제한될 수 있습니다.</p>
<p>Kubelet이 제대로 작동하지 않으면 마스터 노드의 주요 서비스들이 정상적으로 동작하지 않게 되어 API 서버 접근이 차단될 수 있습니다.</p>
<p>모든 노드에서 Containerd와 Kubelet을 재시작함으로써 서비스가 정상적으로 복구되었습니다.</p>
<h4 id="24-마스터-노드-간의-상태-불일치">2.4 마스터 노드 간의 상태 불일치</h4>
<p>세 개의 마스터 노드에서 Keepalived, HAProxy, Kubelet 등의 상태가 일치하지 않거나 일부 노드가 장애 상태였을 가능성이 있습니다.</p>
<p>모든 마스터에서 서비스를 재시작하면서 클러스터의 상태가 다시 동기화되어 VIP 접근이 가능해졌습니다.</p>
<h4 id="25-네트워크-문제-또는-세션-상태">2.5 네트워크 문제 또는 세션 상태</h4>
<p>네트워크 경로에서 문제가 있었거나, Keepalived 및 HAProxy 간의 세션이 꼬였을 가능성도 있습니다.</p>
<p>모든 관련 서비스를 재시작하면서 네트워크 상태나 세션이 초기화되어 문제가 해결되었을 수 있습니다.</p>
<h4 id="3-해결-방법-매뉴얼">3. 해결 방법 매뉴얼</h4>
<p>다음 절차를 따라 VIP 연결 문제를 해결할 수 있습니다:</p>
<p>서비스 재시작</p>
<pre><code>systemctl restart keepalived
systemctl restart haproxy
systemctl restart containerd
systemctl restart kubelet</code></pre><p>모든 마스터 노드에서 위 명령어를 실행하여 서비스들을 재시작합니다.</p>
<p>VIP 상태 확인</p>
<p>각 마스터 노드에서 VIP가 올바르게 활성화되었는지 확인합니다.</p>
<p>ip addr show
VIP가 각 마스터 노드 중 하나에 올바르게 할당되어 있는지 확인합니다.</p>
<p>HAProxy 설정 확인</p>
<p>/etc/haproxy/haproxy.cfg 파일을 열어 VIP 트래픽이 올바르게 백엔드 노드로 분산되고 있는지 확인합니다.</p>
<p>설정을 변경한 경우, 설정 파일을 다시 로드하거나 HAProxy를 재시작합니다.</p>
<pre><code>systemctl restart haproxy</code></pre><p>네트워크 및 방화벽 확인</p>
<p>네트워크 방화벽 규칙이 VIP와 관련된 포트를 차단하고 있지 않은지 확인합니다.</p>
<p>각 노드에서 iptables나 Proxmox 방화벽 설정을 점검하여 VIP 접근이 허용되어 있는지 확인합니다.</p>
<p>서비스 상태 점검</p>
<p>각 노드에서 Containerd와 Kubelet 서비스가 정상적으로 실행 중인지 확인합니다.</p>
<pre><code>systemctl status containerd
systemctl status kubelet</code></pre>