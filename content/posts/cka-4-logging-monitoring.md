---
title: "CKA with Practice Tests: Logging & Monitoring(+ Prometheus, Grafana)"
date: 2022-01-25T17:33:38+09:00
tags:
- cka
- prometheus
---

## 클러스터 컴포넌트 모니터
모니터 대상은 크게 노드와 파드 두개로 나뉜다. 기본적인 [메트릭 API](https://github.com/kubernetes/metrics/blob/master/pkg/apis/metrics/v1beta1/types.go) 있다. 이를 집계하려면 별도의 메트릭 서버가 필요하다.

먼저 로컬에 구성한 minikube의 경우 metrics-server를 애드온으로 추가하면 `kubectl top` 명령으로 CPU와 메모리 사용량을 알 수 있다.
```sh
❯ minikube version
minikube version: v1.24.0
commit: 76b94fb3c4e8ac5062daf70d60cf03ddcc0a741b

❯ minikube addons enable metrics-server
    ▪ Using image k8s.gcr.io/metrics-server/metrics-server:v0.4.2
🌟  'metrics-server' 애드온이 활성화되었습니다

❯ minikube addons list
|-----------------------------|----------|--------------|-----------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |      MAINTAINER       |
|-----------------------------|----------|--------------|-----------------------|
| ambassador                  | minikube | disabled     | unknown (third-party) |
| auto-pause                  | minikube | disabled     | google                |
| csi-hostpath-driver         | minikube | disabled     | kubernetes            |
| dashboard                   | minikube | disabled     | kubernetes            |
| default-storageclass        | minikube | enabled ✅   | kubernetes            |
| efk                         | minikube | disabled     | unknown (third-party) |
| freshpod                    | minikube | disabled     | google                |
| gcp-auth                    | minikube | disabled     | google                |
| gvisor                      | minikube | disabled     | google                |
| helm-tiller                 | minikube | disabled     | unknown (third-party) |
| ingress                     | minikube | disabled     | unknown (third-party) |
| ingress-dns                 | minikube | disabled     | unknown (third-party) |
| istio                       | minikube | disabled     | unknown (third-party) |
| istio-provisioner           | minikube | disabled     | unknown (third-party) |
| kubevirt                    | minikube | disabled     | unknown (third-party) |
| logviewer                   | minikube | disabled     | google                |
| metallb                     | minikube | disabled     | unknown (third-party) |
| metrics-server              | minikube | enabled ✅   | kubernetes            |
| nvidia-driver-installer     | minikube | disabled     | google                |
| nvidia-gpu-device-plugin    | minikube | disabled     | unknown (third-party) |
| olm                         | minikube | disabled     | unknown (third-party) |
| pod-security-policy         | minikube | disabled     | unknown (third-party) |
| portainer                   | minikube | disabled     | portainer.io          |
| registry                    | minikube | disabled     | google                |
| registry-aliases            | minikube | disabled     | unknown (third-party) |
| registry-creds              | minikube | disabled     | unknown (third-party) |
| storage-provisioner         | minikube | enabled ✅   | kubernetes            |
| storage-provisioner-gluster | minikube | disabled     | unknown (third-party) |
| volumesnapshots             | minikube | disabled     | kubernetes            |
|-----------------------------|----------|--------------|-----------------------|

❯ kubectl top node
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
minikube   515m         25%    3114Mi          81% 

❯ kubectl top pod -n kube-system
NAME                                  CPU(cores)   MEMORY(bytes)   
coredns-78fcd69978-mkxm8              4m           22Mi            
etcd-minikube                         40m          57Mi            
kube-apiserver-minikube               135m         412Mi           
kube-controller-manager-minikube      50m          62Mi            
kube-proxy-68nvv                      1m           17Mi            
kube-scheduler-minikube               7m           20Mi            
kube-state-metrics-796889f8b9-9z78d   2m           14Mi            
metrics-server-77c99ccb96-rcjpw       6m           11Mi            
storage-provisioner                   4m           13Mi
```

메트릭 결과는 기본적으로 프로메테우스 형식으로 나온다고 한다. 메트릭 서버는 클러스터의 모든 노드에 CPU, 메모리 사용량을 질의한다. 또 노드 쿠블릿의 cAdvisor(container advisor)를 통해서 파드의 메트릭을 수집한다. 강의의 설명은 이게 전부다. 그리고 연습문제도 top 찍어보고 끝난다. 

... 좀 시시하지만 어차피 클러스터를 운영하는 처지가 아닌 *시험 보기 위해 공부하는데* 이쪽을 더 깊게 알아보려 해봤자 한계가 있을거 같았다. 따라서 cAdvisor나 강의에서 자세히 다루지 않는 부분을 더 찾아보진 않았다.

다만 프로메테우스를 메트릭 서버로 연동해보곤 싶었다. 워낙 유명하고 기술 스택으로 쓰는 곳이 많아서 궁금했다. 따라서 이 글 끝에 minikube에서 쿠버네티스 클러스터 메트릭 서버로서 프로메테우스 + 그라파나 연동을 다룬다.

[참고: 리소스 모니터링 도구](https://kubernetes.io/ko/docs/tasks/debug-application-cluster/resource-usage-monitoring/)

## 애플리케이션 로그
여기서 애플리케이션이란 파드에서 실행하는 컨테이너를 의미한다. 실행 중인 파드에 대해 `kubectl log -f <pod>`하면 표준출력인 로그를 볼 수 있다(Detached 도커에서 `docker log <container>` 실행하는 것과 같다). 파드가 여러 컨테이너 실행 중일 경우 `kubectl log -f <pod> <container>`로 특정할 수 있고, 컨테이너를 누락할 시 대화형으로 물어본다.

## 시험 명령/팁 모음
시험과 관련한 명령이나 팁을 정리한다. 원랜 저번 Scheduler 정리부터 필요하다고 생각했지만 이번 섹션 분량이 적어 여기서부터 넣어야지 했다. 그런데 이번 섹션엔 정리할게 딱히 없다. 그래서 지난 섹션에 정리해둔 것이지만, 내용은 특정 섹션에만 연관 있지 않는 시험 전반적으로 적용할 수 있는 명령과 팁이다.

- 명령으로 여러 레이블 셀렉터를 쓸 때 콤마로 잇는다: e.g. `kubectl get all -l key1=val1,key2=val2`
- 이미지로 파드 생성: `kubectl run <name> --image=<image>`
- 명령 옵션으로 특정할 수 없는 파드 스펙이 있을 때 뼈대 파일 만들기: `kubectl run <name> --image=<image> --dry-run=client -o yaml > <name>.yml`
- 노드 테인트 삭제: `kubectl taint no <node> key=val-`
- 파드 리스트에서 노드 확인하기: `kubectl get po -o wide`
- 노드 레이블링: `kubectl label node <node> <key>=<val>`
- 디플로이먼트 생성: `kubectl create deploy <name> --image=<image> --replicas=<replicas>`
- 모든 네임스페이스에서 리소스 조회: `kubectl get <object> -A`
- 새 대몬셋 정의 파일을 만들 땐, 레플리카셋을 dry-run하고 kind를 수정
- 노드 IP는 `kubectl get no -o wide` 로 확인 가능
- `kubectl get events -o wide` SOURCE 탭에서 이벤트의 스케줄러 확인 가능

Edit 불가능한 파드 스펙 수정하기
- spec.containers[*].image
- spec.initContainers[*].image
- spec.activeDeadlineSeconds
- spec.tolerations
- 이 부분을 수정하려면 우선 edit 하여 다른 이름으로 저장한다(vi로 `:w <name>.yml`)
- 기존 파드는 delete 하고 <name>.yml 수정하여 apply
- 디플로이먼트는 template 아래를 수정하면 새 파드를 실행한다(= edit 가능)


일반(다이나믹) 파드, 스태틱 파드, 대몬셋 구분법

1.`kubectl describe <pod> | grep Controlled`
  - Node/ 이면 스태틱 노드
  - DaemonSet/ 이면 대몬셋
  - 나머진 일반 파드
2. get으로 리스트하여 이름 뒤에 `-<node>` 가 붙으면 스태틱 노드
  - spec.ownerReferences[*].kind를 확인하거나 필터해도 되지만 명령이 복잡함(앞선 Controlled by의 앞부분과 같다)
  - 참고: https://stackoverflow.com/questions/65657808/how-to-identify-static-pods-via-kubectl-command


## 메트릭 서버로 프로메테우스 사용하기
프로메테우스는 [기본적으로 쿠버네티스의 노드를 모니터랑 할 수 있다](https://kubernetes.io/ko/docs/tasks/debug-application-cluster/resource-usage-monitoring/#%EC%99%84%EC%A0%84%ED%95%9C-%EB%A9%94%ED%8A%B8%EB%A6%AD-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8)고 한다. 또 [CNCF 졸업 프로젝트로 쿠버네티스보다 졸업일도 빠르다(!)](https://www.cncf.io/projects/prometheus/). 많은 곳에서 쓰이며, 난 회사에서 데이터독을 쓰는데 비슷한 역할로 이해하고 있다.


(soundcloud가 만든 오픈소스라는데, 로고의 톤앤매너를 보니(=듣고보니) 그런거 같다)

프로메테우스의 특징, 장단점, 을 소개하는 글은 많이 있다. 다음 두가지를 주로 이야기하는데:
1. 호스트(노드)에서 메트릭을 push 하지 않고 서버에서 pull 한다.
2. 클러스터 구성(HA, scale-out, ...)이 어렵다.

먼저 첫번째는, 데이터독을 써본 내 입장에서, 완전한 장점은 아닌거 같다. 데이터독은 모니터 안하던 메트릭도 계속 쌓아두고 있다. 그래서 나중에 그쪽에 문제가 생기면 과거 메트릭도 볼 수 있어서 좋았다.

하지만 지금은 간단한 실습용으로 구성하는 것이 목적인데, 이런 면에선 좋은거 같다. 데이터독은 호스트에 에이전트 설치하고 지원하는 앱 통합(integration) 설정하면 알아서 해주어 편하긴 하지만, 이러면 *어떤 메트릭을 수집해야지에 대한 지식이나 기술은 내것이 아니게 된다.* 이 부분이 지금의 프로메테우스 연동 연습을 하는데 큰 동기가 됐다. 또 실제 운영시 메트릭 서버를 직접 구성하니, 필요한 메트릭만 수집하여 리소스를 최적화한다는 장점이 있다.

두번째 특징인 클러스터 구성에 대해선 단점이라고 이야기 하는 글들이 많다. 하지만 지금 실습에선 전혀 문제되지 않는다.

지금부터 프로메테우스 실습에선 쿠버네티스 관련한 내용은 시맨틱 설명만 한다. 시험 준비하면서 정리했던 내용이나 또 앞으로 배울 내용과 겹치기 때문이다. 대신 프로메테우스 관련한 부분은 조금 자세히 살펴본다(*참고한 링크 예시들과 달리 파드는 제외한 노드 메트릭만 수집했다. 당장 활발히 사용하는 파드가 없어서 그렇게 했다. 필요하다면 나중에 추가해본다*).

### 환경
```sh
❯ sw_vers
ProductName:    macOS
ProductVersion: 11.6.1
BuildVersion:   20G224

❯ minikube version
minikube version: v1.24.0
commit: 76b94fb3c4e8ac5062daf70d60cf03ddcc0a741b
```

### kube-state-metrics(KSM)
쿠버네티스의 기본 컴포넌트가 아닌 오브젝트(파드, 노드, 디플로이먼트 등)의 메트릭을 수집하는 도구다. kube-apiserver를 listen하여 이를 수행한다. 이 역시 파드로 실행된다.

[examples/standard](https://github.com/kubernetes/kube-state-metrics/tree/master/examples/standard)의 정의를 그대로 사용했다. sparse clone 하든 복사하든 로컬로 옮겨서 apply 하면
kube-system에 배포된걸 볼 수 있다:

```sh
❯ kubectl get svc -n kube-system
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
kube-dns             ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   27d
kube-state-metrics   ClusterIP   None           <none>        8080/TCP,8081/TCP        25h
metrics-server       ClusterIP   10.97.185.37   <none>        443/TCP                  112m

❯ kubectl get deploy -n kube-system
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
coredns              1/1     1            1           27d
kube-state-metrics   1/1     1            1           25h
metrics-server       1/1     1            1           112m
```

KSM은 프로메테우스에서 쿠버네티스 오브젝트 메트릭을 수집하는데 사용하게 된다. 따라서 NodePort로 외부 노출하지 않고 내부 IP로만 통신한다.ConfigRole을 아직 공부하진 않았지만, KSM의 것을 보면 configmap, secrete, nodes, pods 등에 대한 읽기(list, watch) 권한을 가진 역할을 만든다고 생각할 수 있다. ClusterRoleBinding은 스케줄링 시 pod binding API처럼 역할 생성 API라 이해했다.

### 프로메테우스

먼저 모니터링 관련한 네임스페이스를 생성한다:
```sh
❯ kubectl create ns monitoring
```

프로메테우스도 KSM과 비슷한 권한을 주기 위해 프로메테우스의 관심사에 대해 ConfigRole을 정의하고 생성(ConfigRoleBinding)한다:

```yml
# prometheus-cluster-role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: monitoring
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```

그리고 프로메테우스 서버 파드 관련한 리소스를 정의한다. 연습용이니 파드 하나만 배포한다. 로컬(외부)에서 접속할 수 있게 외부 노출 서비스도 필요하다:
```yml
# prometheus.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf

        - name: prometheus-storage-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector:
    app: prometheus-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 9090
```

스토리지 볼륨은 그냥 빈 디렉토리를 준다. 프로메테우스 설정 파일을 파드에 넘겨줄 수 있는 ConfigMap을 정의한다. 먼저 프로메테우스 설정 파일은 다음과 같다. 위의 파드 리소스 정의와 헷갈리지 않게 프로메테우스 config/prometheus.yml에 파일을 썼다:
```yml
# config/prometheus.yml
global:
  scrape_interval: 5s
  evaluation_interval: 5s
scrape_configs:
  - job_name: 'kubernetes-apiservers'

    kubernetes_sd_configs:
    - role: endpoints
    scheme: https

    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https

  - job_name: 'kubernetes-nodes'

    scheme: https

    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    kubernetes_sd_configs:
    - role: node

    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: kubernetes.default.svc:443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics


  - job_name: 'kubernetes-pods'

    kubernetes_sd_configs:
    - role: pod

    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: kubernetes_pod_name

  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

  - job_name: 'kubernetes-cadvisor'

    scheme: https

    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

    kubernetes_sd_configs:
    - role: node

    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: kubernetes.default.svc:443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

  - job_name: 'kubernetes-service-endpoints'

    kubernetes_sd_configs:
    - role: endpoints

    relabel_configs:
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
      action: replace
      target_label: __scheme__
      regex: (https?)
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]
      action: replace
      target_label: kubernetes_name
```
- global.scrape_interval: 타겟을 스크랩하는 기본 주기
- global.evaluation_interval: 룰을 평가하는 기본 주기
- scrape_configs[*].job_name: 스크랩하는 타겟 이름
- scrape_configs[*].kubernetes_sd_configs: 쿠버네티스 서비스 디스커버리 설정, kube-apiserver에서 타겟을 스크랩하는 것에 대한 설정
- [scrape_configs[\*].kubernetes_sd_configs[\*].role](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)

위와 같은 프로메테우스가 기본으로 제공 서비스 디스커버리 설정이 많이 돌아다닌다. KSM의 경우는 위에서 설치한 엔드포인트를 static_configs.[0].targets 중 하나로 노출한다. 위 파일로 ConfigMap을 만들어 프로메테우스 파드에 마운트한다:
```yml
 kubectl create configmap prometheus-server-conf --from-file=config/prometheus.yml -n monitoring
```

지금까지 만든 오브젝트 정의를 모두 적용하면(`kubectl apply -f .`) 프로메테우스가 시작된다. 프로메테우스 서비스 주소를 찾아 브라우저로 접속하면 기본 UI를 볼 수 있다:
```sh
❯ minikube service --url prometheus-service -n monitoring
http://192.168.64.2:30003
```

![1](/images/cka-4-logging-monitoring/1.png)

상단 메뉴에서 Status > Targets을 클릭하면 프로메테우스 설정(config/prometheus.yml)에서 정의한 job 단위로 수집하는 메트릭 출처의 엔드포인트를 볼 수 있다. Status의 다른 메뉴인 Service Discovery를 보면 설정의 source_labels와 Discovered Labels가 매칭된다:

![2](/images/cka-4-logging-monitoring/2.png)

## 노드 익스포터
kube-apiserver KSM으로부터 수집하는 시스템 메트릭 외의 메트릭을 수집하기 위해 노드 익스포터를 설치한다. 익스포터는 호스트의 필요한 메트릭을 프로메테우스로 보내주는 역할을 한다. 컨테이너 이미지가 있기 때문에 간단하게 설치할 수 있다. 노드마다 필요하기 때문에 대몬셋으로 설치한다:
```yml
# prometheus-node-exporter.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - image: prom/node-exporter
        name: node-exporter
        ports:
        - containerPort: 9100
          protocol: TCP
          name: http
```
(노드 익스포터는 프로메테우스와 통신만 하면 되기 때문에 서비스 노출은 필요 없다. 그런데 참고한 예제를 포함해서 웬만한 예제들은 다 NodePort 서비스를 하나씩 만들더라... 그래서 여기선 안 만들었다)

노드 익스포터를 실행하고 프라이빗 IP를 확인한다:
```sh
❯ kubectl describe po node-exporter-7nrhs -n monitoring | grep IP
IP:           172.17.0.9
IPs:
  IP:           172.17.0.9
```

프로메테우스가 들을 수 있게 서비스 디스커버리에 추가한다. containerPort인 9100을 타겟으로 한다:
```yml
# config/prometheus.yml
global:
  ...
scrape_configs:
- job_name: 'server-infos'
  static_configs:
  - targets: ['172.17.0.9:9100']
  ...
```
![3](/images/cka-4-logging-monitoring/3.png)

설정에 추가한 엔드포인트에 대한 타겟이 생긴걸 확인할 수 있다.

### 그라파나
프로메테우스 기본 UI보다 더 보기 편한 대시보드를 구성하기 위해 그라파나를 사용한다. 마찬가지로 한 파드만 배포하고 로컬 브라우저에서 볼 수 있도록 NodePort 서비스를 노출한다:
```yml
# grafana.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - name: grafana
          containerPort: 3000
        env:
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          value: /
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '3000'
spec:
  selector:
    app: grafana
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
```
apply 후 서비스 URL을 찾아 접속해본다:
```sh
❯ minikube service --url grafana -n monitoring
http://192.168.64.2:30004
```
![4](/images/cka-4-logging-monitoring/4.png)

첫 페이지에서 Data Sources 를 눌러 Prometheus를 고르고 프로메테우스와 데이터를 연동한다. HTTP URL 설정을 기본 값이 아닌 프로메테우스 서비스를 찾아 바꿔준다. 이 예제에선 포트는 8080으로 했다:
```sh
❯ kubectl get svc  -n monitoring
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
grafana              NodePort   10.102.154.48   <none>        3000:30004/TCP   21h
prometheus-service   NodePort   10.96.39.174    <none>        8080:30003/TCP   32h
```

나머진 그대로 두고 화면 밑으로 내려 Save & Test 한다. 첫 화면으로 돌아와 대시보드를 만든다. 좌측 메뉴 +(Create) > Import를 누르면 grafana.com에 이미 공개되어 있는 대시보드 파일을 임포트할 수 있다. [그라파나 대시보드 페이지](https://grafana.com/grafana/dashboards/)에서 kubernetes로 검색하면 프로메테우스 메트릭으로부터 사용할 수 있는 대시보드들이 나온다. 난 노드 관련 대시보드를 만들기 위해 [8171](https://grafana.com/grafana/dashboards/8171)를 임포트 했다. 위 화면에서 Import via grafana.com 폼에 8171만 넣고 Load 하면 된다.

![5](/images/cka-4-logging-monitoring/5.png)

대시보드 이름이 맞는지 정도 확인하고 Prometheus에서 Prometheus(default)를 고른다. 금방 연동한 데이터 소스 하나 밖에 없을 것이다.

![6](/images/cka-4-logging-monitoring/6.png)

임포트하면 노드의 CPU, 메모리, 디스크, 네트워크에 대한 간단한 대시보드가 구성된다.

참고:
- [Kubernetes Monitoring - Prometheus 실습](https://gruuuuu.github.io/cloud/monitoring-02/)
- [쿠버네티스 #14 - 모니터링 (2/3) Prometheus](https://bcho.tistory.com/1270)
- [KUBERNETES MONITORING – PROMETHEUS 설치](https://linux.systemv.pe.kr/kubernetes-monitoring-prometheus-%EC%84%A4%EC%B9%98/)

---

프로메테우스와 그라파나를 처음부터 연동해봐서 좋았다. 하지만 역시 실제로 모니터할 필요가 크게 없으니 무얼봐야 할진 잘 모르겠다. 참고엔 있었지만 여기선 빠뜨린 파드 메트릭이나 프로메테우스의 alert manager도 다음에 추가해봐야겠다.

그리고 당분간 CKA 챕터 정리가 뜸할수도 있다. 정리하는건 좋은데 자꾸 이렇게 시험과 무관한 쪽에 관심이 생기다보니 진도가 잘 안빠지는것 같아서 빠르게 1회강을 해보려고 한다.
