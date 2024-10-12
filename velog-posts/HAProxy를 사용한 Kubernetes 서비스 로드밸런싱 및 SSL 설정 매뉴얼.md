<p>HAProxy를 통해 Jenkins, ArgoCD, Airflow, Grafana, Harbor, Kubeflow, Prometheus, Tableau 등의 Kubernetes 서비스에 대해 로드밸런싱을 설정하고, 각 서비스에 SSL 인증서를 적용하는 방법을 다룹니다.</p>
<h4 id="1-ssl-인증서-준비">1. SSL 인증서 준비</h4>
<p>각 서비스에 사용할 SSL 인증서를 .pem 형식으로 준비합니다. 도메인별로 각각의 인증서를 준비해야 합니다. </p>
<h4 id="2-인증서-파일-병합-및-저장">2. 인증서 파일 병합 및 저장</h4>
<p>인증서(.crt)와 개인키(.key)를 하나의 .pem 파일로 병합하여 /etc/haproxy/certs/ 경로에 저장합니다.</p>
<pre><code>cat your_certificate.crt your_private.key &gt; your_combined.pem</code></pre><p>각 서비스별 인증서를 다음과 같이 저장합니다:</p>
<pre><code>/etc/haproxy/certs/jenkins.lge.com.pem
/etc/haproxy/certs/argocd.lge.com.pem
/etc/haproxy/certs/airflow.lge.com.pem</code></pre><h4 id="3-haproxy-설정-파일-haproxycfg-수정">3. HAProxy 설정 파일 (haproxy.cfg) 수정</h4>
<p>haproxy.cfg 파일을 열어 각 서비스에 대한 로드밸런싱 및 SSL 설정을 추가합니다. 예시는 다음과 같습니다:</p>
<pre><code>global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    ssl-default-bind-options no-sslv3
    ssl-default-bind-ciphers PROFILE=SYSTEM

defaults
    log     global
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/
    mode http

    acl jenkins_acl hdr(host) -i jenkins.com
    acl argocd_acl hdr(host) -i argocd.com
    acl airflow_acl hdr(host) -i airflow.com
    acl grafana_acl hdr(host) -i grafana.com
    acl harbor_acl hdr(host) -i harbor.com
    acl kubeflow_acl hdr(host) -i kubeflow.com
    acl prometheus_acl hdr(host) -i prometheus.com
    acl tableau_acl hdr(host) -i tableau.com

    use_backend jenkins_backend if jenkins_acl
    use_backend argocd_backend if argocd_acl
    use_backend airflow_backend if airflow_acl
    use_backend grafana_backend if grafana_acl
    use_backend harbor_backend if harbor_acl
    use_backend kubeflow_backend if kubeflow_acl
    use_backend prometheus_backend if prometheus_acl
    use_backend tableau_backend if tableau_acl

backend jenkins_backend
    mode http
    server jenkins_server  jenkins.default.svc.cluster.local:8080

backend argocd_backend
    mode http
    server argocd_server &lt;ARGOCD_SERVICE_IP&gt;:8080

backend airflow_backend
    mode http
    server airflow_server &lt;AIRFLOW_SERVICE_IP&gt;:8080

backend grafana_backend
    mode http
    server grafana_server &lt;GRAFANA_SERVICE_IP&gt;:3000

backend harbor_backend
    mode http
    server harbor_server &lt;HARBOR_SERVICE_IP&gt;:443 ssl verify none

backend kubeflow_backend
    mode http
    server kubeflow_server &lt;KUBEFLOW_SERVICE_IP&gt;:8080

backend prometheus_backend
    mode http
    server prometheus_server &lt;PROMETHEUS_SERVICE_IP&gt;:9090

backend tableau_backend
    mode http
    server tableau_server &lt;TABLEAU_SERVICE_IP&gt;:80</code></pre><h4 id="4-설정-파일-설명">4. 설정 파일 설명</h4>
<p>SSL 인증서 적용: bind *:443 ssl crt /etc/haproxy/certs/는 443 포트에 SSL 인증서를 적용합니다.
도메인별 ACL: acl을 사용하여 도메인별로 요청을 식별하고, 각 도메인에 따라 적절한 백엔드로 요청을 라우팅합니다.
백엔드 서버 설정: 각 서비스에 대해 클러스터 IP 또는 NodePort를 통해 접근하도록 백엔드를 설정합니다. Kubernetes DNS 이름을 사용할 수도 있습니다.</p>
<h4 id="5-haproxy-재시작-및-설정-확인">5. HAProxy 재시작 및 설정 확인</h4>
<p>설정이 완료되면 HAProxy 서비스를 재시작하고 설정 파일이 유효한지 확인합니다.</p>
<pre><code>sudo systemctl restart haproxy
sudo haproxy -f /etc/haproxy/haproxy.cfg -c</code></pre><h4 id="6-kubernetes와의-통합-고려-사항">6. Kubernetes와의 통합 고려 사항</h4>
<p>Kubernetes 내에서 HAProxy 실행: HAProxy를 Kubernetes 클러스터 내에서 Pod로 실행하거나, Ingress Controller로 설정할 수 있습니다.
외부에서의 접근: HAProxy가 클러스터 외부에 있다면, NodePort 또는 외부 IP를 사용하여 서비스에 접근할 수 있습니다.</p>
<h4 id="참고-자료">참고 자료</h4>
<p>HAProxy 공식 문서: <a href="https://haproxy.org/">https://haproxy.org/</a>
Kubernetes 공식 문서: <a href="https://kubernetes.io/docs/">https://kubernetes.io/docs/</a></p>