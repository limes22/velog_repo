<p>설명 : 
VScode 에서 Kubernetes 환경에 접근하는 과정입니다. Remot ssh 플러그인 설치 후 진행합니다. 
요약 : 
•    VScode remote ssh 플러그인 설치
•    Kubernetes pod 안에 ssh 설치
•    NodePort 타입 Service 생성 
•    Portfowarding (필요시)
•    Pod 접속</p>
<p>1)    Remote - ssh 플러그인 설치
 <img alt="" src="https://velog.velcdn.com/images/limes22/post/daa2d69d-0b92-42b7-8681-d2c25fd2311f/image.png" /></p>
<p>VScode에서 확장 플러그인을 설치해줍니다.</p>
<p>2) Kubernetes pod 안에 ssh 를 설치해 줍니다.
$&gt; apt-get update &amp;&amp; apt-get install -y openssh-server vim
$&gt; vi /etc/ssh/sshd_config</p>
<p> <img alt="" src="https://velog.velcdn.com/images/limes22/post/5102cde0-7425-463d-9769-b5208808a3c1/image.png" /></p>
<ol>
<li>Sshd_config 파일을 열어 해당값을 수정합니다.
$&gt; service ssh restart</li>
</ol>
<p>3) node port type svc 생성
 <img alt="" src="https://velog.velcdn.com/images/limes22/post/cf4a1bfb-0b7c-4d75-ab36-fa1f1a241a07/image.png" /></p>
<p>3-1) 파드의 ssh 포트 포워딩 (필요시)</p>
<ol>
<li>파드에서 실행 중인 특정 애플리케이션에 접근이 필요하므로 포트포워딩을 설정합니다.
$&gt; kubectl port-forward --address 0.0.0.0 pod/nginx-test-pod 30081:22 -n default
<img alt="" src="https://velog.velcdn.com/images/limes22/post/f8e464cc-e081-4e47-90e1-3b91275aa862/image.png" /></li>
</ol>
<p>4) VScode 에서 파드 접속 확인
VScode에서 파드 접속합니다.
 <img alt="" src="https://velog.velcdn.com/images/limes22/post/a196f91c-526e-4e61-bf63-943c35a93d87/image.png" /></p>
<p>파드에 접속을 확인 합니다.
<img alt="" src="https://velog.velcdn.com/images/limes22/post/ef0cc7bb-5def-4f47-b723-245bd88caf41/image.png" /></p>