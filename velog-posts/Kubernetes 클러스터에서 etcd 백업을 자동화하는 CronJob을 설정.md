<h3 id="전제조건">전제조건</h3>
<p>리눅스 시간을 한국 표준시(KST)로 변경하기</p>
<pre><code>date
sudo ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime</code></pre><p>etcd leader 식별 해서 lable 생성하는 cronjob 생성</p>
<pre><code>#!/bin/bash

#member ID 확인
MEMBER_ID=$(ETCDCTL_API=3 etcdctl endpoint status --write-out=json \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | jq -r '.[].Status.leader')

# 리더 노드를 확인
LEADER_NODE=$(ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379         --cacert=/etc/kubernetes/pki/etcd/ca.crt         --cert=/etc/kubernetes/pki/etcd/server.crt         --key=/etc/kubernetes/pki/etcd/server.key         member list --write-out=json | jq -r --arg MEMBER_ID &quot;$MEMBER_ID&quot; '.members[] | select(.ID | tostring == $MEMBER_ID) | .name')

# 기존 노드의 레이블 제거 (기존 리더 레이블 제거)
kubectl label node --all role-

# 새로운 리더 노드에 레이블 추가
kubectl label node $LEADER_NODE role=etcd-leader```

### 1. Docker 이미지 생성
먼저 etcd 백업 스크립트를 포함한 Docker 이미지를 만들어야 합니다.

#### 1-1. 백업 스크립트 작성
vi /etcd-backup.sh</code></pre><pre><code># Ubuntu 이미지를 베이스로 사용
FROM ubuntu:20.04

#필요한 도구 설치
RUN apt-get update &amp;&amp; \
    apt-get install -y jq etcd-client curl &amp;&amp; \
    curl -LO &quot;https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl&quot; &amp;&amp; \
    chmod +x kubectl &amp;&amp; \
    mv kubectl /usr/local/bin/kubectl

# 백업 스크립트 복사
COPY leader.sh /usr/local/bin/leader.sh

# 스크립트 실행
ENTRYPOINT [&quot;/usr/local/bin/leader.sh&quot;]```

# 필요한 도구 설치
if ! command -v jq &amp;&gt; /dev/null
then
    echo &quot;jq를 설치 중입니다...&quot;
    apt-get update &amp;&amp; apt-get install -y jq
fi

if ! command -v etcdctl &amp;&gt; /dev/null
then
    echo &quot;etcdctl를 설치 중입니다...&quot;
    apt-get update &amp;&amp; apt-get install -y etcd-client
fi

# 백업을 저장할 경로
BACKUP_DIR=&quot;/data/backup&quot;

# 현재 노드 이름 가져오기
NODE_NAME=$(hostname)

# etcd 리더 노드 찾기
LEADER_NODE=$(ETCDCTL_API=3 etcdctl endpoint status --write-out=json \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | jq -r '.[].Status.leader')

MEMBER_ID=$(ETCDCTL_API=3 etcdctl endpoint status --write-out=json \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | jq -r '.[].Status.header.member_id')


# 현재 노드가 리더 노드인지 확인
if [[ &quot;$LEADER_NODE&quot; == *&quot;$MEMBER_ID&quot;* ]]; then
  echo &quot;이 노드는 리더 노드입니다: $NODE_NAME&quot;
  BACKUP_FILE=&quot;$BACKUP_DIR/etcd-snapshot-${NODE_NAME}-$(date +%Y-%m-%d_%H-%M-%S).db&quot;

  # 백업 수행
  ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

  if [ $? -eq 0 ]; then
    echo &quot;백업이 성공적으로 완료되었습니다: $BACKUP_FILE&quot;
  else
    echo &quot;백업에 실패했습니다.&quot;
    exit 1
  fi
else
  echo &quot;이 노드는 리더 노드가 아닙니다. 백업을 건너뜁니다.&quot;
fi
</code></pre><p>default namespace 파드 안에서 kubectl 명령어를 사용할 수 있도록 권한 설정</p>
<pre><code>apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default</code></pre><pre><code>apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-patcher
rules:
- apiGroups: [&quot;&quot;]
  resources: [&quot;nodes&quot;]
  verbs: [&quot;get&quot;, &quot;list&quot;, &quot;patch&quot;]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: patch-nodes-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-patcher
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default</code></pre><pre><code>apiVersion: batch/v1
kind: CronJob
metadata:
  name: update-leader-node
spec:
  schedule: &quot;*/5 * * * *&quot;  # 5분마다 실행
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          serviceAccountName: my-service-account
          nodeSelector:
            role: etcd-leader
          containers:
          - name: update-leader
            image: howdi2000/etcd-leader:v0.3  # 스크립트가 포함된 이미지
            command:
            - /bin/bash
            - -c
            - |
              /usr/local/bin/leader.sh
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd</code></pre><h3 id="1-docker-이미지-생성">1. Docker 이미지 생성</h3>
<p>먼저 etcd 백업 스크립트를 포함한 Docker 이미지를 만들어야 합니다.</p>
<h4 id="1-1-백업-스크립트-작성">1-1. 백업 스크립트 작성</h4>
<p>vi /etcd-backup.sh</p>
<pre><code>#!/bin/bash

# 필요한 도구 설치
if ! command -v jq &amp;&gt; /dev/null
then
    echo &quot;jq를 설치 중입니다...&quot;
    apt-get update &amp;&amp; apt-get install -y jq
fi

if ! command -v etcdctl &amp;&gt; /dev/null
then
    echo &quot;etcdctl를 설치 중입니다...&quot;
    apt-get update &amp;&amp; apt-get install -y etcd-client
fi

# 백업을 저장할 경로
BACKUP_DIR=&quot;/data/backup&quot;

# 현재 노드 이름 가져오기
NODE_NAME=$(hostname)

# etcd 리더 노드 찾기
LEADER_NODE=$(ETCDCTL_API=3 etcdctl endpoint status --write-out=json \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | jq -r '.[].Status.leader')

MEMBER_ID=$(ETCDCTL_API=3 etcdctl endpoint status --write-out=json \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | jq -r '.[].Status.header.member_id')


# 현재 노드가 리더 노드인지 확인
if [[ &quot;$LEADER_NODE&quot; == *&quot;$MEMBER_ID&quot;* ]]; then
  echo &quot;이 노드는 리더 노드입니다: $NODE_NAME&quot;
  BACKUP_FILE=&quot;$BACKUP_DIR/etcd-snapshot-${NODE_NAME}-$(date +%Y-%m-%d_%H-%M-%S).db&quot;

  # 백업 수행
  ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key

  if [ $? -eq 0 ]; then
    echo &quot;백업이 성공적으로 완료되었습니다: $BACKUP_FILE&quot;
  else
    echo &quot;백업에 실패했습니다.&quot;
    exit 1
  fi
else
  echo &quot;이 노드는 리더 노드가 아닙니다. 백업을 건너뜁니다.&quot;
fi
</code></pre><h4 id="1-2-dockerfile-작성">1-2. Dockerfile 작성</h4>
<p>vi Dockerfile</p>
<pre><code># Ubuntu 이미지를 베이스로 사용
FROM ubuntu:20.04


# 백업 스크립트 복사
COPY etcd-leader-backup2.sh /usr/local/bin/etcd-leader-backup.sh

# 백업 스크립트에 실행 권한 부여

# 스크립트 실행
ENTRYPOINT [&quot;/usr/local/bin/etcd-leader-backup.sh&quot;]</code></pre><h4 id="1-4-docker-이미지-빌드">1-4. Docker 이미지 빌드</h4>
<pre><code>docker build -t howdi2000/etcd-backup:v0.1 .</code></pre><h4 id="1-5-docker-이미지-레지스트리에-푸시">1-5. Docker 이미지 레지스트리에 푸시</h4>
<pre><code>docker push your-registry/etcd-backup:v0.1</code></pre><h3 id="2-리더-노드-식별하고-레이블-추가">2. 리더 노드 식별하고 레이블 추가</h3>
<h4 id="2-1-리더-노드-식별">2-1. 리더 노드 식별</h4>
<pre><code>ETCDCTL_API=3 etcdctl endpoint status --write-out=json \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | jq -r '.[] | select(.Status.leader == .Status.header.member_id) | .Endpoint'</code></pre><h4 id="리더-노드에-레이블-추가">리더 노드에 레이블 추가</h4>
<pre><code>kubectl label node &lt;리더 노드 이름&gt; role=etcd-leader</code></pre><h3 id="3-cronjob-생성">3. cronjob 생성</h3>
<pre><code>apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
spec:
  schedule: &quot;0 0 * * 0&quot; # 매주 일요일 자정에 백업
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            role: etcd-leader
          containers:
          - name: etcd-backup
            image: howdi2000/etcd-backup:v0.1 # 백업 스크립트를 포함한 커스텀 이미지
            command:
            - /bin/bash
            - -c
            - |
              /usr/local/bin/etcd-leader-backup.sh
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-storage
              mountPath: /data/backup
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd # etcd 인증서 경로
          - name: backup-storage
            hostPath:
              path: /data/backup # 백업이 저장될 경로</code></pre><h3 id="4-network-policy-생성">4. network policy 생성</h3>
<p>hostNetwork: true를 사용할 때, Pod의 네트워크 통신을 제어하기 위해 네트워크 정책(NetworkPolicy)을 적용할 수 있습니다. 이를 통해 Pod가 특정 IP 또는 포트만 통신하도록 제한할 수 있습니다.</p>
<pre><code>apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: limit-pod-traffic
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: etcd-backup
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.244.0.0/16  # 허용된 네트워크 범위
    ports:
    - protocol: TCP
      port: 2379  # etcd 클라이언트 포트
  egress:
  - to:
    - ipBlock:
        cidr: 10.244.0.0/16
    ports:
    - protocol: TCP
      port: 2380  # etcd 피어 통신 포트</code></pre><p>이 정책은 Pod가 2379 및 2380 포트에서만 통신할 수 있도록 제한합니다. 이를 통해 Pod의 네트워크 접근을 제어하고, 필요하지 않은 외부 통신을 차단할 수 있습니다.</p>
<h3 id="5-cronjob-배포">5. cronjob 배포</h3>
<pre><code>kubectl apply -f etcd-backup-cronjob.yaml</code></pre><h3 id="6-cronjob-실행-확인">6. cronjob 실행 확인</h3>
<pre><code>kubectl get cronjob</code></pre><p><img alt="" src="https://velog.velcdn.com/images/limes22/post/2a9aa21c-282d-4b48-9942-a5bedab01cbe/image.png" /></p>