<h2 id="custom-controller-operator-pattern">&lt; Custom Controller Operator Pattern &gt;</h2>
<h4 id="1-operator-sdk">1. Operator-SDK</h4>
<p>애플리케이션을 패키징, 배포 및 관리하는 것
컨트롤러, 커스텀 리소스 생성, 등록을 자동화해 놓은 것  </p>
<h4 id="2-crdcustom-resource-definition-란">2. CRD(Custom Resource Definition) 란?</h4>
<p>직접 리소스의 종류를 정의해서 사용할 수 있음. 
K8s 컨트롤러
선언형(Declarative) 방식 : 최종적으로 도달해야 하는 바람직한 상태(Desired State)를 정의 한 뒤, 현재 상태(Current State)가 바람직한 상태와 다를 경우 이를 일치하도록 만드는 방법이다.
<img alt="" src="https://velog.velcdn.com/images/limes22/post/f55ec72c-b5b6-45cf-a87b-e349c36817d2/image.png" />
<img alt="" src="https://velog.velcdn.com/images/limes22/post/41f66513-02f2-4028-8ac8-d4dba6de2fb9/image.png" /></p>
<p>Operator 패턴 : CRD를 사용할 수 있도록 컨트롤러를 구현하는 방법
Reconcile: 현재 상태가 desired State 가 되도록 특정 동작을 수행 하는 것</p>
<p><img alt="" src="https://velog.velcdn.com/images/limes22/post/a45e3b31-eddf-4cc8-ab7c-ce647cb3b74f/image.png" />
<img alt="" src="https://velog.velcdn.com/images/limes22/post/b277cb9c-3f37-4c63-b02b-4bcfd6e772a1/image.png" /></p>
<h4 id="3-operator-sdk-디렉터리-구조">3. Operator-sdk 디렉터리 구조</h4>
<p>1) Main.go
Outcluster 방식으로 동작 시킬 수 있도록 파일 생성되어 있음
2) Api / v1
Api//_types.go CRD의 API 정의되어 있음
3) Controllers/
컨트롤러 구현하는 디렉터리이다. Controller/_controller.go 파일을 편집하여 지정된 리소스 유형을 처리하도록 컨트롤러의 Reconsile logic을 정의한다.
4) Confg/bases/~.yaml
CRD .yaml 파일 위치이다.
Config/crd/kustomization.yaml
오브젝트 생성과 설정이 정의되어 있다.
5) Makefile
컨트롤러를 빌드하고 배포하는데 사용한다.</p>
<h4 id="4-operator-sdk-로-crd-cr-생성">4. Operator-sdk 로 CRD, CR 생성</h4>
<p>Github : <a href="https://github.com/limes22/operator-sdk">https://github.com/limes22/operator-sdk</a></p>
<p>1) Crd spec 정의 (api -&gt; crd)
$ make generate 
Memcahched_type 파일은 기본적인 내용만 있어서 상세 내용 추가하여 zz_generated.deepcopy.go 파일을 생성한다.
<img alt="" src="https://velog.velcdn.com/images/limes22/post/02a17312-16ed-4e59-af1c-f6d3d25f3a50/image.png" /></p>
<p>2) CRD 작업 사항 template 코드로 떨구기
$make manifests 
zz_generated.deepcopy.go 기반으로 config base 파일이 만들어짐
<img alt="" src="https://velog.velcdn.com/images/limes22/post/60a2e414-7bfb-40fd-b430-6a18b4d58660/image.png" /></p>
<p>3) Crd manifest 파일 생성</p>
<p>CRD를 ETCD에 등록
$ make install  //CRD 생성 
<img alt="" src="https://velog.velcdn.com/images/limes22/post/563f592a-15fa-4df5-9c87-04cf7ea4bb3b/image.png" /></p>
<p>4) Kustomization 파일을 통해서 CRD 생성됨
Crd 가 api server 에 전달되서 etcd에 저장된다.
Config/crd/bases/.yaml -&gt; etcd 에 등록</p>
<p>5) Controller 생성
Config 디렉토리에 manager.yaml 파일 내용대로 만들어진다.
<img alt="" src="https://velog.velcdn.com/images/limes22/post/96e30be7-b402-4ab3-9920-be6417e0b8ef/image.png" />
<img alt="" src="https://velog.velcdn.com/images/limes22/post/d72b7619-daac-4235-b8b5-84bb74ee50c9/image.png" /></p>
<p>6) Namespace와 deployment 가 선언되어 있다.
<img alt="" src="https://velog.velcdn.com/images/limes22/post/5cc88ff2-8906-4828-b333-f46574c0aa9a/image.png" /></p>
<p>Controller.go 파일도 reconcile, createdeployment 함수 추가해서 수정한다.</p>
<p>7) Docker operator’s image registry 설정
도커 이미지 이름변경 : docker registry/imagename : tag
$ make docker-build docker-push</p>
<p>8) Operator 실행
Outside cluster 모드로 실행
terminal에 process 개념으로 띄우는 것
$ make install  // 아웃클러스터 모드로 디플로이먼트 생성
$ go run main.go  //main.go 파일 실행
Inside cluster deployment 로 실행
$ make deploy</p>
<p>CR 생성
<img alt="업로드중.." src="blob:https://velog.io/ecb419db-6e64-4191-91b7-a348aee3a410" /></p>
<p>$ kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml</p>
<h4 id="reference">Reference</h4>
<ol>
<li>오퍼레이터 패턴 - <a href="https://kubernetes.io/ko/docs/concepts/extend-kubernetes/operator/">https://kubernetes.io/ko/docs/concepts/extend-kubernetes/operator/</a></li>
<li>Operator-sdk - <a href="https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/">https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/</a></li>
<li>Operator-sdk - <a href="https://sdk.operatorframework.io/docs/overview/project-layout/">https://sdk.operatorframework.io/docs/overview/project-layout/</a></li>
<li>Operator-sdk - <a href="https://sdk.operatorframework.io/docs/building-operators/golang/installation/">https://sdk.operatorframework.io/docs/building-operators/golang/installation/</a></li>
<li>Kube-api - <a href="https://book.kubebuilder.io/cronjob-tutorial/gvks.html">https://book.kubebuilder.io/cronjob-tutorial/gvks.html</a></li>
</ol>