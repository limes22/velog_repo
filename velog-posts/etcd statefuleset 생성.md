<pre><code>---
apiVersion: v1
kind: Service
metadata:
  name: etcd
  namespace: default
  labels:  # labels 필드를 metadata 섹션 아래로 이동
    app: etcd
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: etcd
  ports:
    - name: etcd-client
      port: 2379
    - name: etcd-server
      port: 2380
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
  namespace: default
spec:
  serviceName: &quot;etcd&quot;
  replicas: 1
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.15
        imagePullPolicy: IfNotPresent
        ports:
        - name: etcd-client
          containerPort: 2379
        - name: etcd-server
          containerPort: 2380
        env:
        - name: ETCDCTL_API
          value: &quot;3&quot;
        - name: ETCD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ETCD_DATA_DIR
          value: /etcd-data
        - name: ETCD_INITIAL_ADVERTISE_PEER_URLS
          value: http://$(ETCD_NAME).etcd:2380
        - name: ETCD_ADVERTISE_CLIENT_URLS
          value: http://$(ETCD_NAME).etcd:2379
        - name: ETCD_LISTEN_PEER_URLS
          value: http://0.0.0.0:2380
        - name: ETCD_LISTEN_CLIENT_URLS
          value: http://0.0.0.0:2379
        - name: ETCD_INITIAL_CLUSTER
          value: &quot;etcd-0=http://etcd-0.etcd:2380,etcd-1=http://etcd-1.etcd:2380,etcd-2=http://etcd-2.etcd:2380&quot;
        - name: ETCD_INITIAL_CLUSTER_STATE
          value: &quot;new&quot;
        - name: ETCD_INITIAL_CLUSTER_TOKEN
          value: &quot;etcd-cluster&quot;
        volumeMounts:
        - name: etcd-data
          mountPath: /etcd-data
  volumeClaimTemplates:
  - metadata:
      name: etcd-data
    spec:
      accessModes: [&quot;ReadWriteOnce&quot;]
      resources:
        requests:
          storage: 5Gi
</code></pre><p>cluster role</p>
<pre><code>apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-pvc-access
rules:
- apiGroups: [&quot;&quot;]
  resources: [&quot;persistentvolumeclaims&quot;, &quot;persistentvolumes&quot;]
  verbs: [&quot;get&quot;, &quot;list&quot;, &quot;watch&quot;]
- apiGroups: [&quot;&quot;]
  resources: [&quot;serviceaccounts&quot;, &quot;serviceaccounts/token&quot;]
  verbs: [&quot;create&quot;, &quot;get&quot;, &quot;list&quot;, &quot;watch&quot;]</code></pre><pre><code>apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-pvc-access-binding
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-pvc-access
  apiGroup: rbac.authorization.k8s.io</code></pre>