<p>ETCD 클러스터가 3개의 마스터 노드로 구성되어 있고, static pod로 구성된 환경에서 etcdctl restore 명령어를 사용하여 백업한 데이터로 복구하는 방법입니다.</p>
<h4 id="전제-조건">전제 조건</h4>
<p>etcdctl 명령어를 사용할 수 있어야 함.
각 마스터 노드에서 etcd가 static pod로 실행되고 있어야 함.</p>
<p>모든 명령어는 master 1, 2, 3 에서 모두 실행해야함.</p>
<pre><code>#etcdctl 설치
export RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '&quot;' -f 4)
wget https://github.com/etcd-io/etcd/releases/download/${RELEASE}/etcd-${RELEASE}-linux-amd64.tar.gz
tar xvf etcd-${RELEASE}-linux-amd64.tar.gz
cd etcd-${RELEASE}-linux-amd64
sudo mv etcd etcdctl etcdutl /usr/local/bin 
etcd --version</code></pre><h4 id="1etcd-스냅샷-백업">1.ETCD 스냅샷 백업</h4>
<pre><code>export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        snapshot save /path/to/backup/etcd-snapshot.db</code></pre><h4 id="2-etcd-정지">2. ETCD 정지</h4>
<pre><code>mv /etc/kubernetes/manifests/etcd.yaml /tmp/</code></pre><p>kubelet은 /etc/kubernetes/manifests/ 디렉토리에서 YAML 파일을 기반으로 etcd 컨테이너를 관리하기 때문에, 파일을 이동하면 etcd가 자동으로 중지됨</p>
<h4 id="3-etcd-데이터-디렉토리-정리">3. ETCD 데이터 디렉토리 정리</h4>
<pre><code>rm -rf /var/lib/etcd</code></pre><h4 id="4-etcd-스냅샷-복원">4. ETCD 스냅샷 복원</h4>
<p>master1</p>
<pre><code>etcdctl snapshot restore /path/to/backup/etcd-snapshot.db \
        --data-dir /var/lib/etcd \
        --name master1 \
        --initial-cluster master1=https://${master1_ip}:2380,master2=https://${master2_ip}:2380,master3=https://${master3_ip}:2380 \
        --initial-cluster-token etcd-cluster \
        --initial-advertise-peer-urls https://${master1_ip}:2380</code></pre><p>master2</p>
<pre><code>etcdctl snapshot restore /path/to/backup/etcd-snapshot.db \
        --data-dir /var/lib/etcd \
        --name master2 \
        --initial-cluster master1=https://${master1_ip}:2380,master2=https://${master2_ip}:2380,master3=https://${master3_ip}:2380 \
        --initial-cluster-token etcd-cluster \
        --initial-advertise-peer-urls https://${master2_ip}:2380</code></pre><p>master3</p>
<pre><code>etcdctl snapshot restore /path/to/backup/etcd-snapshot.db \
        --data-dir /var/lib/etcd \
        --name master3 \
        --initial-cluster master1=https://${master1_ip}:2380,master2=https://${master2_ip}:2380,master3=https://${master3_ip}:2380 \
        --initial-cluster-token etcd-cluster \
        --initial-advertise-peer-urls https://${master3_ip}:2380</code></pre><p>--data-dir: 복원할 디렉터리 (/var/lib/etcd)
--name: 현재 노드의 이름 (master1)
--initial-cluster: 모든 etcd 멤버를 지정 (다른 마스터 노드의 IP 포함)
--initial-advertise-peer-urls: 복원하려는 노드의 peer URL</p>
<h4 id="5-etcd-다시-시작">5. ETCD 다시 시작</h4>
<pre><code>mv /tmp/etcd.yaml /etc/kubernetes/manifests/</code></pre><h4 id="5-1-containerd-kubelete-재시작">5-1. containerd, kubelete 재시작</h4>
<pre><code>systemctl restart containerd
systemctl restart kubelet</code></pre><h4 id="6-etcd-멤버-상태-확인">6. ETCD 멤버 상태 확인</h4>
<p>복구 후, etcd 클러스터의 상태를 확인하여 멤버들이 정상적으로 조인되었는지 확인</p>
<pre><code>etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        member list</code></pre><p><img alt="" src="https://velog.velcdn.com/images/limes22/post/a8c5cddd-6385-428d-9192-ba1e756266f1/image.png" /></p>
<h4 id="7-kubernetes-클러스터-상태-확인">7. Kubernetes 클러스터 상태 확인</h4>
<pre><code>kubectl get nodes
kubectl get pods --all-namespaces</code></pre><h4 id="8-문제-해결">8. 문제 해결</h4>
<p>복구 중 문제가 발생할 경우:</p>
<p>멤버 재조인 실패: 문제가 있는 노드를 etcdctl member remove 명령어로 제거한 후 다시 추가</p>
<p>member remove</p>
<pre><code>export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        member remove &lt;멤버-ID&gt;</code></pre><p>member 조인</p>
<pre><code>etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        member add &lt;새 멤버 이름&gt; --peer-urls=https://&lt;새로운 멤버 IP&gt;:2380</code></pre><p>member 확인</p>
<pre><code>etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        member list --write-out=table</code></pre>