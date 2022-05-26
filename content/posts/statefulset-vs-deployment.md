---
title: "Statefulset(vs. Deployment)"
date: 2022-05-26T12:54:04+09:00
tags:
- kubernetes
---

쿠버네티스 문서의 [스테이트풀셋 기본](https://kubernetes.io/ko/docs/tutorials/stateful-application/basic-stateful-set/) 실습을 따라해본다. 추가로, 기본적으로 스테이트리스의 파드셋을 관리하는, 디플로이먼트와 비교해본다. 하지만 스테이트풀셋과 디플로이먼트가 완전한 대척점에 있다고 보긴 어려워, 설명은 스테이트풀셋 위주로 할 것이다({스테이트풀셋의 기능} - {디플로이먼트의 기능}을 설명하게 된다).

스테이트풀셋은 공부하게 된 계기는 [DOIK(Database Operator in Kubernetes)](https://gasidaseo.notion.site/e49b329c833143d4a3b9715d75b5078d)라는 스터디에 참여했기 때문이다. 스터디는 우연한 계기로 알게 됐는데, 스테이트풀셋이 실제 제품에서 사용 가능할까? 라는 의문이 있었기 때문에 참여했다. 스터디 이름에서 알 수 있듯, 실제론 [오퍼레이터](https://kubernetes.io/ko/docs/concepts/extend-kubernetes/operator/)라는 설계 패턴으로 구현하는 것 같다. 하지만 DB를 컨테이너로 운영하는 기본은 스테이트풀셋이라 1주차엔 이것에 대한 스터디를 했다.

문서에서 말하듯 [스테이트풀셋](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), [헤드리스 서비스](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services), [영구 볼륨](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)(요청), [동적 볼륨 프로비저닝](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)의 개념을 알고 있어야 한다. 내 기준으론, 기본적인 쿠버네티스의 스테이트리스 파드셋 관리, 즉 디플로이먼트에 익숙한 사람이면 따라하기 쉬울 것 같다.

그 기준에서 조금 설명이 필요한 것은 동적 볼륨 프로비저닝인데, 실습에서 사용할 [rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)를 이야기한다. 그리고 스테이트풀과 비슷한 디플로이먼트와의 비교 실습은 다음 순서로 진행된다. 실습 환경은 이 [레포](https://github.com/flavono123/kubernetes-the-hard-way)이다:

- 네트워크 ID와 저장소
- 스케일링
- 업데이트
- 삭제
- 파드 관리 정책

## rancher/local-path-provisioner
rancher/local-path-provisioner는 파드가 실행 중인 노드(호스트)에 동적 볼륨 프로비저닝을 하는 간단한 구현체이다. 쿠버네티스에서 [동적 볼륨 프로비저닝은 저장소 클래스(StorageClass)를 생성하고 동적으로 영구 볼륨 요청(PVC)하여 영구 볼륨(PV)을 할당 받는 식](https://kubernetes.io/ko/docs/concepts/storage/dynamic-provisioning/)이다. 따라서 rancher/local-path-provisioner는 저장소 클래스와 이 클래스로 생성하는 요청 시 영구 볼륨을 만드는 프로비저너로 구성돼 있다.

그 외 RBAC, 네임스페이스 등 추가적인 자원을 포함해 설치 파일은 한개이다(v0.22는 현재 stable 버전):
```sh
$ k apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
```

동적 볼륨 프로비저닝이 어떻게 동작하는지 보기 위해 RBAC 관련 그리고 네임스페이스는 제외하고 StorageClass, Deployment, ConfigMap만 살펴보자:
```sh
$ wget -qO- https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
  | yq ea 'select(.kind=="ConfigMap" or .kind=="Deployment" or .kind=="StorageClass")
          | split_doc | [.] | sort_by(.kind) | reverse'
- apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: local-path
  provisioner: rancher.io/local-path
  volumeBindingMode: WaitForFirstConsumer
  reclaimPolicy: Delete
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: local-path-provisioner
    namespace: local-path-storage
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: local-path-provisioner
    template:
      metadata:
        labels:
          app: local-path-provisioner
      spec:
        serviceAccountName: local-path-provisioner-service-account
        containers:
          - name: local-path-provisioner
            image: rancher/local-path-provisioner:master-head
            imagePullPolicy: IfNotPresent
            command:
              - local-path-provisioner
              - --debug
              - start
              - --config
              - /etc/config/config.json
            volumeMounts:
              - name: config-volume
                mountPath: /etc/config/
            env:
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
        volumes:
          - name: config-volume
            configMap:
              name: local-path-config
- kind: ConfigMap
  apiVersion: v1
  metadata:
    name: local-path-config
    namespace: local-path-storage
  data:
    config.json: |-
      {
              "nodePathMap":[
              {
                      "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                      "paths":["/opt/local-path-provisioner"]
              }
              ]
      }
    setup: |-
      #!/bin/sh
      set -eu
      mkdir -m 0777 -p "$VOL_DIR"
    teardown: |-
      #!/bin/sh
      set -eu
      rm -rf "$VOL_DIR"
    helperPod.yaml: |-
      apiVersion: v1
      kind: Pod
      metadata:
        name: helper-pod
      spec:
        containers:
        - name: helper-pod
          image: busybox
          imagePullPolicy: IfNotPresent

```
yq 필터 설명
- `select` - 원하는 리소스 종류(kind)만 선택해
- `split_doc | [.]` - 각 파일의 YAML을 한 배열로 합침(eval-all(ea)와 같이 써야 함)
- `sort_by(.kind) | reverse` - kind에 대해 역으로 정렬하면 설명하고 싶은 순서로 나온다.

1. StorageClass - local-path
  - 이어서 설명하는 디플로이먼트를 프로비저너로 쓴다. 영세한(?) 프로비저너인지 [외부 라이브러리 목록](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner)에서 찾아볼 순 없다(실습을 하는데엔 오작동 없이 충분하다).
  - volumeBindingMode: WaitForFirstConsumer - [파드에 바인딩된 PVC가 생겨야 PV를 프로비저닝한다.](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
2. Deployment - local-path-provisioner
  - 프로비저너. 상세한 동작은 컨피그맵에 쓰여 있다.
3. ConfigMap - local-path-config
  - config.json - hostPath를 백엔드로, 즉 노드 경로에 볼륨을 만드는데 상위 디렉토리는 /opt/local-path-provisioner 이다.
  - setup - hostPath에 디렉토리 경로를 만들어 볼륨을 프로비저닝하고
  - teardown - PVC가 삭제되면 만든 디렉토리를 삭제한다.
  - helperPod.yaml - busybox를 써서 프로비저닝(mkdir, rm -rf)을 한다.

실제 사용 예시는 스테이트풀셋을 만들어 보며 볼 수 있다.

설치가 잘 됐으면, 저장소 클래스가 추가됐을 것이다:
```sh
$ k apply -f local-path.yaml
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created
$ k get sc
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  5s
```

이를 기본으로 사용하기 위해 어노테이션을 추가하자:
```sh
$ k patch sc local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/local-path patched
$ k get sc
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  21s
```


## 파일 준비
스테이프풀셋과 비교할 디플로이먼트 정의 파일을 준비한다:
```yaml
# web-sts.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-sts
  labels:
    app: nginx-sts
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx-sts
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts
spec:
  serviceName: nginx-sts
  replicas: 2
  selector:
    matchLabels:
      app: nginx-sts
  template:
    metadata:
      labels:
        app: nginx-sts
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

[문서의 예제 파일](https://raw.githubusercontent.com/kubernetes/website/main/content/ko/examples/application/web/web.yaml)을 참고해 이름과 레이블 -sts 접미사를 붙였다. 디플로이먼트와 구분하기 위함인데, PVC가 네임스페이스 리소스라 네임스페이스로 격리하는건 적절하지 않다고 생각했다. spec.volumeClaimTemplates에 storageClassName을 써주지 않으면 default인 local-path를 사용하게 될 것이다.


```yaml
# web-dep.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-dep
  labels:
    app: nginx-dep
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx-dep
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: www-web-dep
  labels:
    app: nginx-dep
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dep
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-dep
  template:
    metadata:
      labels:
        app: nginx-dep
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: www-web-dep
```
스테이트풀셋과 비교할 디플로이먼트다. 스테이트풀셋과 마찬가지로 -dep 접미사로 객체 이름이나 레이블을 구분했다. 디플로이먼트는, 어차피 파드마다 네트워크를 식별할 필요가 없어, 헤드리스 서비스가 필수는 아니지만 비교를 위해 똑같이 만들었다.

디플로이먼트는 volumeClaimTemplates 즉, PVC를 템플릿 할 순 없다. 따라서 PVC를 명시적으로 **한개만** 만들었다(스테이트풀셋과 좋은 비교점이 되는진 모르겠다).


## 네트워크 ID와 저장소 확인
스테이트풀셋과 디플로이먼트의 큰 차이점은 파드 생성, 삭제에 순번을 주어 네트워크를 식별하고 저장소를 일정하게 붙인다. 파드의 생명주기는 자체는 동적이라 이 자원들을 고정적(static)이라 표현하지 않고 안정적(stable)이라고 한다.

실습을 하며 파드 생성, 삭제 순서를 보기 위해 다음 모니터 명령을 별도 터미널에 실행해 놓자:
```sh
$ k get pods -l app=nginx-sts # 스테이트풀셋
$ k get pods -l app=nginx-dep # 디플로이먼트
```

먼저 스테이트풀셋을 실행하면 파드 이름이 '<스테이트풀셋>-<순번>'으로 실행된다:
```sh
$ k apply -f web-sts.yaml
# 모니터
web-sts-0   0/1     Pending   0          0s
web-sts-0   0/1     Pending   0          4s
web-sts-0   0/1     ContainerCreating   0          4s
web-sts-0   0/1     ContainerCreating   0          5s
web-sts-0   1/1     Running             0          5s
web-sts-1   0/1     Pending             0          0s
web-sts-1   0/1     Pending             0          5s
web-sts-1   0/1     ContainerCreating   0          5s
web-sts-1   0/1     ContainerCreating   0          6s
web-sts-1   1/1     Running             0          7s
```

디플로이먼트는 파드 이름이 '<디플로이먼트>-<디플로이먼트_다이제스트>-<파드_다이제스트>'로 식별된다:

```sh
$ k apply -f web-dep.yaml
#모니터
web-dep-74f5674789-2b7px   0/1     Pending   0          0s
web-dep-74f5674789-hlvgd   0/1     Pending   0          0s
web-dep-74f5674789-hlvgd   0/1     Pending   0          0s
web-dep-74f5674789-hlvgd   0/1     Pending   0          4s
web-dep-74f5674789-hlvgd   0/1     ContainerCreating   0          4s
web-dep-74f5674789-hlvgd   0/1     ContainerCreating   0          5s
web-dep-74f5674789-2b7px   0/1     Pending             0          5s
web-dep-74f5674789-2b7px   0/1     ContainerCreating   0          5s
web-dep-74f5674789-2b7px   0/1     ContainerCreating   0          6s
web-dep-74f5674789-hlvgd   1/1     Running             0          6s
web-dep-74f5674789-2b7px   1/1     Running             0          7s
```

스테이트풀셋의 파드 생성은 상태가 Pending -> ContainerCreating -> Running(Ready)를 거친 후 다음 순서 파드를 생성하지만, 디플로이먼트는 다른 파드의 생성이나 순서 영향 없이 병렬로 동시에 생성한다.

스테이트풀셋에서 호스트 이름을 네트워크 ID로 쓰게 된다. 각 파드의 호스트 이름을 확인하고 DNS 질의 해보자:
```sh
$ for i in 0 1; do kubectl exec "web-sts-$i" -- sh -c 'hostname'; done
web-sts-0
web-sts-1
$ k run --image busybox:1.28 dns-test --restart=Never --rm -it -- nslook^up web-sts-0.nginx-sts
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-sts-0.nginx-sts
Address 1: 172.16.25.222 web-sts-0.nginx-sts.default.svc.cluster.local
pod "dns-test" deleted
$ k run --image busybox:1.28 dns-test --restart=Never --rm -it -- nslookup web-sts-1.nginx-sts
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-sts-1.nginx-sts
Address 1: 172.16.40.87 web-sts-1.nginx-sts.default.svc.cluster.local
pod "dns-test" deleted
```

파드 이름의 DNS 레코드가 등록되어 있다. 하지만 디플로이먼트에선 몇번째 파드이느냐가 중요하지 않기 때문에 그러한 DNS 레코드는 등록하지 않는다:
```sh
$ k get pods -l app=nginx-dep --no-headers | awk '{print $1}' | xargs -I{} kubectl exec {} -- sh -c 'hostname'
web-dep-74f5674789-2b7px
web-dep-74f5674789-hlvgd
$ k run --image busybox:1.28 dns-test --restart=Never --rm -it  -- nslookup web-dep-74f5674789-2b7px.nginx-dep
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'web-dep-74f5674789-2b7px.nginx-dep'
pod "dns-test" deleted
pod default/dns-test terminated (Error)
$ k run --image busybox:1.28 dns-test --restart=Never --rm -it  -- nslookup web-dep-74f5674789-hlvgd.nginx-dep
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve 'web-dep-74f5674789-hlvgd.nginx-dep'
pod "dns-test" deleted
pod default/dns-test terminated (Error)
```

대신 디플로이먼트의 [(일반)파드는 자신의 IP를 dasherize한 레코드가 등록](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#a-aaaa-records-1)된다. 스테이트풀셋의 것은 그렇지 않다:
```sh
$ k get po -l app=nginx-dep -owide
NAME                       READY   STATUS    RESTARTS   AGE   IP             NODE               NOMINATED NODE   READINESS GATES
web-dep-74f5674789-2b7px   1/1     Running   0          12m   172.16.40.90   cluster1-worker1   <none>           <none>
web-dep-74f5674789-hlvgd   1/1     Running   0          12m   172.16.40.89   cluster1-worker1   <none>           <none>
$ k run --image busybox:1.28 dns-test --restart=Never --rm -it  -- nslookup 172-16-40-90.nginx-dep
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      172-16-40-90.nginx-dep
Address 1: 172.16.40.90 172-16-40-90.nginx-dep.default.svc.cluster.local
pod "dns-test" deleted
$ k get po -l app=nginx-sts -owide
NAME        READY   STATUS    RESTARTS   AGE   IP              NODE               NOMINATED NODE   READINESS GATES
web-sts-0   1/1     Running   0          14m   172.16.25.222   cluster1-worker2   <none>           <none>
web-sts-1   1/1     Running   0          14m   172.16.40.87    cluster1-worker1   <none>           <none>
$ k run --image busybox:1.28 dns-test --restart=Never --rm -it  -- nslookup 172-16-25-222.nginx-dep
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

nslookup: can't resolve '172-16-25-222.nginx-dep'
pod "dns-test" deleted
pod default/dns-test terminated (Error)
```

다음으로 스테이트풀셋의 동적으로 생성된 PVC와 디플로이먼트에서 선언한 것을 보자:
```sh
$ k get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-dep     Bound    pvc-6758d381-cfcd-47b5-ac73-481123ee3a43   1Gi        RWO            local-path     23m
www-web-sts-0   Bound    pvc-dc2a9ddd-8bdb-4064-8634-828e0479bbac   1Gi        RWO            local-path     25m
www-web-sts-1   Bound    pvc-bb7b15ed-817f-42f0-af0f-375436f3ce2d   1Gi        RWO            local-path     25m
```

스테이트풀셋의 파드는 순번 접미사로 구분되어 각각 PVC가 생겼지만, 디플로이먼트는 그렇지 않다. 파드가 2개임에도 하나의 PVC만 있는데 이는 위에서 그렇게 정의했기 때문에 그렇다.

파드 IP 확인하기 위해 본 파드 목록 wide 출력을 보면 디플로이먼트 파드는 한 노드에 몰려 있는 것을 볼 수 있다. 이따 디플로이먼트를 스케일 아웃하면 더 확실히 보일 것이다. PVC가 노드 종속적이기 때문이다(hostPath). 볼륨이 없는 노드엔 파드를 늘릴 수 없을 것이고, 같은 노드에 다른 컴퓨팅 리소스가 부족해도 파드를 늘릴 수 없을 것이다. 따라서 이런 상태(여기선 볼륨)가 있는 앱은 디플로이먼트가 적합하지 않다는 것을 알 수 있다(물론 하나의 PVC를 여러 디플로이먼트 파드가 쓰는 예제가 적절치 못한것도 있다).

앞서 설명한 local-path-provisioner가 hostPath로 볼륨을 프로비저닝하는 것을 확인해보자. 각 worker 노드에서 /opt/local-path-provisioner로 이동하면 파드의 PV를 볼 수 있다:
```sh
# cluster1-worker1
$ ll /opt/local-path-provisioner/
total 16
drwxr-xr-x 4 root root 4096 May 26 06:57 ./
drwxr-xr-x 5 root root 4096 May 26 01:30 ../
drwxrwxrwx 2 root root 4096 May 26 06:57 pvc-6758d381-cfcd-47b5-ac73-481123ee3a43_default_www-web-dep/
drwxrwxrwx 2 root root 4096 May 26 06:56 pvc-bb7b15ed-817f-42f0-af0f-375436f3ce2d_default_www-web-sts-1/

# cluster1-worker2
$ ll /opt/local-path-provisioner/
total 12
drwxr-xr-x 3 root root 4096 May 26 06:56 ./
drwxr-xr-x 5 root root 4096 May 26 01:29 ../
drwxrwxrwx 2 root root 4096 May 26 06:56 pvc-dc2a9ddd-8bdb-4064-8634-828e0479bbac_default_www-web-sts-0/
```

스테이트풀셋이 안정적인 상태(PV)를 유지하는지 확인해보자. 각 파드 PV에 호스트 이름을 Nginx가 서빙할 수 있는 파일에 써 상태를 주입한다:
```sh
$ for i in 0 1; do k exec "web-sts-$i" -- sh -c 'echo $(hostname) > /usr/share/nginx/html/index.html'; done
$ for i in 0 1; do k exec -it "web-sts-$i" -- curl localhost; done
web-sts-0
web-sts-1
```

파드를 모두 삭제하고 재시작하길 기다렸다가 다시 요청하면, 같은 이름의 파드에 PV가 연결된 것을 확인할 수 있다:
```sh
$ k delete pod -l app=nginx-sts
pod "web-sts-0" deleted
pod "web-sts-1" deleted
$ for i in 0 1; do kubectl exec -i -t "web-sts-$i" -- curl http://localhost/; done
web-sts-0
web-sts-1
```

디플로이먼트는 그렇지 않다:
```sh
$ k get pods -l app=nginx-dep --no-headers | awk '{print $1}' | xargs -I{} kubectl exec {} -- sh -c 'echo $(hostname) > /usr/share/nginx/html/index.html'
$ k get pods -l app=nginx-dep --no-headers | awk '{print $1}' | xargs -I{} kubectl exec {} -- curl -s localhost
web-dep-74f5674789-hlvgd
web-dep-74f5674789-hlvgd
$ k delete po -l app=nginx-dep
pod "web-dep-74f5674789-2b7px" deleted
pod "web-dep-74f5674789-hlvgd" deleted
$ k get pods -l app=nginx-dep --no-headers | awk '{print $1}' | xargs -I{} kubectl exec {} -- curl -s localhost
web-dep-74f5674789-hlvgd
web-dep-74f5674789-hlvgd
```

하나의 PV를 쓰기 때문에 마지막에 실행됐을 파드의 이름(eb-dep-74f5674789-hlvgd)이 덮어 씌워졌고, 삭제 후 새로 생성된 파드 이름과 무관하게 이전에 PV에 쓰인 값이 서빙된다.


## 스케일링
스테이프풀셋은 파드 스케일링할 때도 순서를 따른다:
```sh
$ k scale sts web-sts --replicas=5
statefulset.apps/web-sts scaled
# 모니터
web-sts-2   0/1     Pending   0          0s
web-sts-2   0/1     Pending   0          5s
web-sts-2   0/1     ContainerCreating   0          5s
web-sts-2   0/1     ContainerCreating   0          6s
web-sts-2   1/1     Running             0          7s
web-sts-3   0/1     Pending             0          0s
web-sts-3   0/1     Pending             0          5s
web-sts-3   0/1     ContainerCreating   0          5s
web-sts-3   0/1     ContainerCreating   0          5s
web-sts-3   1/1     Running             0          6s
web-sts-4   0/1     Pending             0          0s
web-sts-4   0/1     Pending             0          5s
web-sts-4   0/1     ContainerCreating   0          5s
web-sts-4   0/1     ContainerCreating   0          5s
web-sts-4   1/1     Running             0          6s

$ k scale sts web-sts --replicas=3
statefulset.apps/web-sts scaled
# 모니터
web-sts-4   1/1     Terminating         0          29s
web-sts-4   1/1     Terminating         0          29s
web-sts-4   0/1     Terminating         0          30s
web-sts-4   0/1     Terminating         0          30s
web-sts-4   0/1     Terminating         0          30s
web-sts-3   1/1     Terminating         0          36s
web-sts-3   1/1     Terminating         0          36s
web-sts-3   0/1     Terminating         0          37s
web-sts-3   0/1     Terminating         0          37s
web-sts-3   0/1     Terminating         0          37s
```

스케일 아웃 시 순번이 순증하며 파드를 생성하고, 스케일 인 시엔 뒤에서부터 역순으로 파드를 제거한다(이 문서를 포함해 쿠버네티스 문서 전체에 스케일 업/다운이란 표현을 쓴다. 하지만 이것은 보통 스케일 아웃/인에 해당하는 동작을 설명하고 있어서 후자로 전부 대체한다). 현재 파드가 실행 중이 아니더라도 PV와 PVC는 남아 있게 된다(web-sts-3, web-sts-4):
```sh
$ k get pvc -l app=nginx-sts
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-sts-0   Bound    pvc-dc2a9ddd-8bdb-4064-8634-828e0479bbac   1Gi        RWO            local-path     101m
www-web-sts-1   Bound    pvc-bb7b15ed-817f-42f0-af0f-375436f3ce2d   1Gi        RWO            local-path     101m
www-web-sts-2   Bound    pvc-c3b54572-90e8-4947-bfd1-28e340505759   1Gi        RWO            local-path     3m51s
www-web-sts-3   Bound    pvc-d7011017-f3fc-4778-99db-d6e023f5cb57   1Gi        RWO            local-path     3m44s
www-web-sts-4   Bound    pvc-28952b79-94d9-4066-95de-b14102b822b1   1Gi        RWO            local-path     3m38s
$  k get pv | grep sts
pvc-28952b79-94d9-4066-95de-b14102b822b1   1Gi        RWO            Delete           Bound    default/www-web-sts-4   local-path              2m52s
pvc-bb7b15ed-817f-42f0-af0f-375436f3ce2d   1Gi        RWO            Delete           Bound    default/www-web-sts-1   local-path              100m
pvc-c3b54572-90e8-4947-bfd1-28e340505759   1Gi        RWO            Delete           Bound    default/www-web-sts-2   local-path              3m5s
pvc-d7011017-f3fc-4778-99db-d6e023f5cb57   1Gi        RWO            Delete           Bound    default/www-web-sts-3   local-path              2m58s
pvc-dc2a9ddd-8bdb-4064-8634-828e0479bbac   1Gi        RWO            Delete           Bound    default/www-web-sts-0   local-path              100m
```

디플로이먼트는, 생성할 때와 마찬가지로, 스케일링 시 동시에 병렬적으로 진행한다:
```sh
$ k scale deploy web-dep --replicas=5
# 모니터
web-dep-74f5674789-7p2q4   0/1     Pending   0          0s
web-dep-74f5674789-7p2q4   0/1     Pending   0          0s
web-dep-74f5674789-jvgqm   0/1     Pending   0          0s
web-dep-74f5674789-zrl8m   0/1     Pending   0          0s
web-dep-74f5674789-jvgqm   0/1     Pending   0          0s
web-dep-74f5674789-zrl8m   0/1     Pending   0          0s
web-dep-74f5674789-7p2q4   0/1     ContainerCreating   0          0s
web-dep-74f5674789-jvgqm   0/1     ContainerCreating   0          0s
web-dep-74f5674789-zrl8m   0/1     ContainerCreating   0          0s
web-dep-74f5674789-7p2q4   0/1     ContainerCreating   0          1s
web-dep-74f5674789-jvgqm   0/1     ContainerCreating   0          1s
web-dep-74f5674789-zrl8m   0/1     ContainerCreating   0          1s
web-dep-74f5674789-jvgqm   1/1     Running             0          2s
web-dep-74f5674789-7p2q4   1/1     Running             0          2s
web-dep-74f5674789-zrl8m   1/1     Running             0          2s

$ k scale deploy web-dep --replicas=3
# 모니터
web-dep-74f5674789-jvgqm   1/1     Terminating         0          2m17s
web-dep-74f5674789-7p2q4   1/1     Terminating         0          2m17s
web-dep-74f5674789-jvgqm   1/1     Terminating         0          2m17s
web-dep-74f5674789-7p2q4   1/1     Terminating         0          2m17s
web-dep-74f5674789-7p2q4   0/1     Terminating         0          2m17s
web-dep-74f5674789-7p2q4   0/1     Terminating         0          2m17s
web-dep-74f5674789-7p2q4   0/1     Terminating         0          2m18s
web-dep-74f5674789-jvgqm   0/1     Terminating         0          2m18s
web-dep-74f5674789-jvgqm   0/1     Terminating         0          2m18s
web-dep-74f5674789-jvgqm   0/1     Terminating         0          2m18s
```

## 업데이트
스테이트풀셋의 업데이트 전략은 기본으로 RollingUpdate이며 디플로이먼트와 같다. 다만 순서는 뒤에서부터 역순으로 된다(파드를 삭제하고 새로 띄우는 작업이니 스케일 인과 연관해 생각해보면 되겠다). 디플로이먼트와 달리 스테이트풀셋의 RollingUpdate는 partition이라는 인자로 제어할 수 있다. partition은 업데이트가 적용될 파드 순번 이상(greater than or equal)을 뜻한다.

partition의 기본 값은 0이다. 만약 컨테이너 이미지를 바꾸는 업데이트를 한다면 뒷순번 파드부터 전체에 대해 이미지를 교체할 것이다:
```sh
$ k get sts web-sts -oyaml | yq .spec.updateStrategy
rollingUpdate:
  partition: 0
type: RollingUpdate
$ k set image sts web-sts nginx=gcr.io/google_containers/nginx-slim:0.8
statefulset.apps/web-sts image updated
# 모니터
web-sts-2   1/1     Terminating   0          13m
web-sts-2   1/1     Terminating   0          13m
web-sts-2   0/1     Terminating   0          13m
web-sts-2   0/1     Terminating   0          13m
web-sts-2   0/1     Terminating   0          13m
web-sts-2   0/1     Pending       0          0s
web-sts-2   0/1     Pending       0          0s
web-sts-2   0/1     ContainerCreating   0          0s
web-sts-2   0/1     ContainerCreating   0          1s
web-sts-2   1/1     Running             0          1s
web-sts-1   1/1     Terminating         0          69m
web-sts-1   1/1     Terminating         0          69m
web-sts-1   0/1     Terminating         0          69m
web-sts-1   0/1     Terminating         0          69m
web-sts-1   0/1     Terminating         0          69m
web-sts-1   0/1     Pending             0          0s
web-sts-1   0/1     Pending             0          0s
web-sts-1   0/1     ContainerCreating   0          0s
web-sts-1   0/1     ContainerCreating   0          1s
web-sts-1   1/1     Running             0          1s
web-sts-0   1/1     Terminating         0          69m
web-sts-0   1/1     Terminating         0          69m
web-sts-0   0/1     Terminating         0          69m
web-sts-0   0/1     Terminating         0          69m
web-sts-0   0/1     Terminating         0          69m
web-sts-0   0/1     Pending             0          0s
web-sts-0   0/1     Pending             0          0s
web-sts-0   0/1     ContainerCreating   0          0s
web-sts-0   0/1     ContainerCreating   0          1s
web-sts-0   1/1     Running             0          1s
```

partition을 3으로 하고 또 다른 이미지(k8s.gcr.io/nginx-slim:0.7)로 업데이트하면 아무 일도 일어나지 않는다. 파드가 모두 파티션 내에 있기 때문이다:
```sh
$ k patch sts web-sts -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
statefulset.apps/web-sts patched
$ k get sts web-sts -oyaml | yq .spec.updateStrategy
rollingUpdate:
  partition: 3
type: RollingUpdate
$ k set image sts web-sts nginx=k8s.gcr.io/nginx-slim:0.7
statefulset.apps/web-sts image updated
# 모니터에 변화가 없다

# 컨테이너 이미지 출력
$ for p in 0 1 2; do k get pod "web-sts-$p" --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
gcr.io/google_containers/nginx-slim:0.8
gcr.io/google_containers/nginx-slim:0.8
gcr.io/google_containers/nginx-slim:0.8
```

partition을 2로하면 web-sts-2 파드만 아까 업데이트한 새 이미지로 변경된다. 이런 식의 카나리 업데이트를 할 수 있다(모니터 로그가 길기 때문에 파드마다 컨테이너 이미지 출력으로 대체):
```sh
$ k patch sts web-sts -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
statefulset.apps/web-sts patched
$  k get sts web-sts -oyaml | yq .spec.updateStrategy
rollingUpdate:
  partition: 2
type: RollingUpdate
$ for p in 0 1 2; do k get pod "web-sts-$p" --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
gcr.io/google_containers/nginx-slim:0.8
gcr.io/google_containers/nginx-slim:0.8
k8s.gcr.io/nginx-slim:0.7
```

web-sts-2 파드에서 테스트가 충분히 됐으면, partition을 0으로 만들어 전체 업데이트하면 된다:
```sh
$ k patch sts web-sts -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
statefulset.apps/web-sts patched
$ for p in 0 1 2; do k get pod "web-sts-$p" --template '{{range $i, $c := .spec.containers}}{{$c.image}}{{end}}'; echo; done
k8s.gcr.io/nginx-slim:0.7
k8s.gcr.io/nginx-slim:0.7
k8s.gcr.io/nginx-slim:0.7
```

디플로이먼트는 partition 옵션이 없다. 디플로이먼트에서 카나리 업데이트를 레이블을 활용해 구현할 수 있지만, 스테이트풀셋처럼 인자로 제어되는 것이 아니라 여기선 다루지 않는다.

## 삭제
문서에 쓰인 Non-cascading/Cascading은 스테이트풀셋만의 기능은 아닌, [API delete의 옵션](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)이다. `cascade` 옵션 지정 없이(기본 background) 스테이트풀셋을 지우면 앞서 스케일 인에서 본 듯, 뒤의 파드부터 순서대로 지우고 스테이트풀셋 리소스도 삭제한다.

`cascade`를 orphan으로 하면 스테이트풀셋 같은 파드셋 리소스만 지우고 매칭된 파드는 지우지 않는다. 이렇게 한다면 더 이상 스테이트풀셋이 없으니 파드셋 순서와 무관하게 파드 삭제가 가능하다.

`cascade` 옵션을 쓰면 스테이트풀셋이 파드를 관리하는 특징과 무관하게 삭제할 수 있지만, `cascade`가 스테이트풀셋만 사용 가능한 옵션은 아니니, 헷갈리지 않도록, 실습은 하지 않는다.

## 파드 관리 정책
기본으로 스테이트풀셋은 파드 순번을 지키며 관리하지만, 디플로이먼트처럼 순서와 무관하게 관리할 수도 있다.

spec.podManagementPolicy는 기본 값이 OrderedReady이지만, Parallel로 설정할 수 있다. 꼭 순서가 중요하지 않은 스테이트풀 앱을 위한 옵션이라고 한다:
```sh
$ k get sts web-sts -oyaml | yq .spec.podManagementPolicy
OrderedReady
# spec.podManagementPolicy 는 패치할 수 없으므로 web-sts.yaml을 수정하고 다시 적용
$ k delete sts web-sts
$ k apply -f web-sts.yaml
# 모니터
web-sts-0   0/1     Pending             0          0s
web-sts-1   0/1     Pending             0          0s
web-sts-0   0/1     Pending             0          0s
web-sts-1   0/1     Pending             0          0s
web-sts-0   0/1     ContainerCreating   0          0s
web-sts-1   0/1     ContainerCreating   0          0s
web-sts-0   0/1     ContainerCreating   0          0s
web-sts-1   0/1     ContainerCreating   0          1s
web-sts-0   1/1     Running             0          1s
web-sts-1   1/1     Running             0          2s
```

## 정리
- 스테이트풀셋을 사용하려면 헤드리스 서비스와 동적 볼륨 프로비저닝 가능한 저장소 클래스가 필요하다.
- 스테이트풀셋은 파드의 순번으로 관리한다.
  - 순번 무관하게 관리하는 옵션도 있다 - spec.podManagementPolicy: Parallel
- 스테이트풀셋의 파드는 순번 접미사로 부여된 호스트 이름으로 네트워크 ID와 저장소를 안정적으로(stable) 연결한다.
- 스테이트풀셋 업데이트 전략 중 RollingUpdate는 partition 인자를 쓰면 카나리 업데이트가 가능하다.

---

## 참고
- https://kubernetes.io/docs/home/
- https://gasidaseo.notion.site/e49b329c833143d4a3b9715d75b5078d (스터디 참고 내용은 private 링크라 지난 모집 공고 링크로 대체)
