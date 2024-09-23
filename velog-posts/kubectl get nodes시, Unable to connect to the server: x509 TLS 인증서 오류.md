<p>master 장비에서 k8s 클러스터 조인을 확인하기 위해 $ kubectl get nodes 커맨드시에 아래와 같이 TLS 인증서 오류 메세지 발생함.</p>
<p><img alt="" src="https://velog.velcdn.com/images/limes22/post/d153b730-647f-49d9-b0b1-13a73c639b6f/image.png" /></p>
<p>해당 메세지를 해결하기 위해서 </p>
<p>export KUBECONFIG=/etc/kubernetes/admin.conf로 가능하지만,</p>
<p>장비를 재기동하면 또 다시 발생한다.</p>
<p>영구적으로 해당 이슈를 피하기 위해서는 아래와 같이 작업하면 된다.</p>
<pre><code>mv $HOME/.kube $HOME/.kube.bak
mkdir $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config</code></pre>