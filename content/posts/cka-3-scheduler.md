---
title: "CKA with Practice Tests 정리: Scheduler"
date: 2022-01-23T13:21:29+09:00
tags:
- kubernetes
- cka
---

## Manage scheduling
스케줄러는 파드를 어떤 노드에 할당(bind)할지 판단한다. 지금까지 파드 생성 시 정의에 명시하지 않았지만, `spec.nodeName`에 할당할 파드를 명시할 수 있다. 이런 방법은 추천하지 않는것 같고, 스케줄러에게 맡기되 그걸 제어할 수 있는 방법을 이번 장에서 다룬다.

스케줄러는 [core Binding API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#binding-v1-core)(`target.kind: Node`)를 이용해 특정 노드에 파드 할당을 요청한다.

스케줄러는 컨트롤플레인 노드에서, 뒤에서 설명할, 스태틱 파드로 실행중이다:
```sh
❯ kubectl describe po kube-scheduler-minikube -n kube-system | grep Controlled
Controlled By:  Node/minikube
```

## 레이블 & 셀렉터
- 레이블: 쿠버네티스 오브젝트를 특정하기 위한 태그. `metadata.labels` 밑에 키-값으로 정의.
- 셀렉터: 하나 이상을 써서 쿠버네티스 오브젝트를 특정한다(`spec.selector.matchLabels`).

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```
예를 들어 위 디플로이먼트 `name: redis-cache`(`metadata.labels`)는 디플로이먼트 자신의 레이블이다. 그리고 `spec.selector.matchLabels`를 써서 특정할 **파드의 레이블**을 써준다. 위 예시에선 파드 스펙도 같이 정의하기 때문에 `template.metadata.labels`는 파드의 레이블이다. 따라서 앞선 두개가 `app: store`로 일치해야 의도한대로 레디스 앱이 배포가 된다.

`selector.matchLabels`는 일치성 기준(equality-based) 요건의 정의 키로 아래 키-값이 정확히 일치하는 셀렉터이다. 그 외에 집합성 기준(set-based) 요건도 정의할 수 있다. 키는 `selector.matchExpresions` 연산자(`selector.matchExpressions[].operator`)로 `In`, `NotIn`, `Exists`를 사용할 수 있다([참고 문서: 레이블과 셀렉터](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/labels/)).

또 레이블이 아니면서 metadata에 *첨부* 할 수 있는 어노테이션도 있다. 보통 연락처(전화번호, 이메일), 버전, 빌드 정보 같은 내용을 적는다는데 그냥 레이블-셀렉터 같은 기능이 없는 말 그대로의 메타데이터인듯 하다. 자세히 다루지 않고 연습문제도 나오지 않는다([참고 문서: 어노테이션](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/annotations/)).

## 테인트 & 톨러레이션
테인트는 노드에 기본적으로 파드를 실행할 수 없도록 표시하는 것이고, 톨러레이션은 파드가 그런 테인트를 무시하고 실행될 수 있게 설정하는 것이다. 강의에선 노드를 사람, 파드를 벌레에 비유하여 벌레가 사람에 붙을 수 있는지(binding)로 설명한다:

![1](/images/cka-3-scheduler/1.png)
![2](/images/cka-3-scheduler/2.png)

사람에 기피제를 뿌리면 벌레가 붙을 수 없다. 하지만 기피제에 내성이 생긴 벌레는 기피제를 뿌린 사람이라도 붙을 수 있다. 여기서 기피제와 내성이 테인트와 톨러레이션에 대응한다. 테인트가 있는 노드엔 보통 파드는 할당될 수 없다. 하지만 해당 테인트에 톨러레이션이 있는 파드는 할당될 수 있다. 따라서 테인트와 톨러레이션은 기본적으로 **특정 파드가 특정 노드에선 *스케줄 하지 않음*을 보장하기 위해 정의**하는 것이다(톨러레이션으로 예외를 둔다.).

테인트 설정은 `kubectl taint nodes <node> <key>=<value>:<effect>`로 한다. 효과(effect)는 `NoSchedule`이 기본이고 다음과 같다:
- Noschedule: 톨러레인트가 없는 파드는 실행하지 않는다.
- PreferNoSchedule: 실행하지 않도록 노력하지만, 완전히 보장하진 않는다.
- NoExecute: 이미 실행 중인 파드가 있다면 방출한다.

설정된 테인트를 제거하려면 설정과 같은 명령 뒤에 빼기(`-`)를 붙인다. `kubectl taint nodes <node> <key>=<value>:<effect>-`

톨러레이션은 파드 `spec.tolerations[]`에 `key`, `operator`, `value`, `effect` 키가 있는 오브젝트로 정의한다. 먼저 연산자는 두개가 있다:
- `Equal`: (기본 값)테인트의 키-값과 '일치'하는 톨러레이션.
- `Exists`: 테인트의 키만 일치하면 값에 상관 없이 톨러레이션이 있다. 따라서 `value`는 써주지 않는다.

효과는 위 테인트 효과 중 두 개 `Noschedule`, `NoExecute`이다. 노드의 모든 테인트의 톨러레이션이 있어야 파드는 할당될 수 있다. 테인트는 톨러레이션에 대한 필터처럼 작동한다. 따라서 파드 `spec.tolerations[].key|value`가 모든 노드 테인트에 대한 부분 집합이어야 할당 가능하다. `NoSchedule` 톨러레이션은 `NoExecute` 테인트에 의해 방출될 수 있다. 따라서 `NoExecute`이 더 **강한** 톨러레이션이라 할 수 있다. 상세한 설명은 [문서: 테인트(Taints)와 톨러레이션(Tolerations)의 예제](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/taint-and-toleration/)를 참고하길 바란다.

## 노드 셀렉터
파드에서 노드를 선택하는 방법이다. 노드에도 레이블을 주고 파드에서 셀렉터를 정의한다(순서상 레이블/셀렉터에 이어서 하는게 낫지 않나 싶네...).

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    size: Large
```

위처럼 파드 스펙에 노드 셀렉터를 정의한다(`spec.nodeSelector`). 노드에 레이블은 다음 명령으로 한다:
`kubectl label node <node> <key>=<val>`. 위 파드를 할당하고 싶으면 `size=Large` 해주면 될 것이다.


## 노드 어피니티
노드 셀렉터보다 더 확장된 기능인 어피니티가 있다. 노드 셀렉터는 일치성 매칭 밖에 제공하지 않는다. 예를 들어 위처럼 노드 크기에 따라 레이블을 달았을 경우 `NOT Small` 같은 매칭을 정의하지 못한다. 만약 size 레이블이 `Large`, `Small`만 있다면, `size=Large`로 충분하겠지만, `Medium`인 노드가 추가된다면 해당 노드엔 파드가 할당되지 않아 `NOT Small`의 기능을 할 수 없다. 이를 해결하는 것이 어피니티이다.

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    size: Large
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExection:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
```

위의 노드 셀렉터 예제를 어피니티로 표현하면 이와 같다. 파드 스펙 `affinity.nodeAffinity.<type>.nodeTerms[]`에 쓰는 `matchExpressions` 파드 리소스 정의의 셀렉터와 유사하다. 하위 오브젝트의 키인 `operator`도 `In`, `NotIn`, `Exists`로 같다.

타입 이름이 상당히 긴데, 다음 세가지가 있다:
- `requiredDuringSchedulingIgnoredDuringExecution`
- `prefferedDuringSchedulingIgnoredDuringExecution`
- `requiredDuringSchedulingRequiredDuringExecution`

먼저 위의 두개(available types)와 아래 하나(planned type)의 차이점은 뒷부분, `DuringExection`이 `Ignored`냐 `Required`이냐이다. 이는 노드에 레이블이 추가, 변경, 삭제 됐을 때 실행 중인 파드 적용 여부를 뜻한다. 따라서 `Ignored`인 두 available types의 어피니티 파드는 노드의 레이블 변경 여부와 무관하지만, planned type인 `requiredDuringSchedulingRequiredDuringExecution`일 경우 노드 레이블이 어피니티를 만족하지 않을 경우, 노드에서 파드가 삭제될 것이다.

두 available types은 다시 앞부분의 `DuringScheduling`로 구분된다. `required`는 어피니티를 만족하는 레이블의 노드에 스케줄 한다는 뜻이고 `preffered`는 꼭 만족하는 노드가 아니더라도 스케줄 될 수 있음을 뜻한다. 이름은 길지만 테인트 & 톨러레이션의 효과와 정책이 비슷하다. 어피니티는 테인트 & 톨러레이션과 달리 **특정 노드에서 *실행할 수 있는* 파드를 특정**하는 것이다.

## 노드 어피니티 vs. 테인트 & 톨러레이션
위에서도 강조했지만, 어피티니와 테인트/톨러레이션은 역할이 다르다. 각자가 무얼 못하는지 보면:
- 톨러레이션은 테인트 노드에서 파드가 스케줄 되는걸 보장하지 않는다.
- 어피니티는 톨러레이션 없는 파드가 테인트 노드에서 실행 안되는걸 보장하지 않는다.

두번째 상황은 이미 첫번째의 테인트/톨러레이션이 설정되어 있는 상황을 가정한다. 테인트/톨러레이션만으론 노드-파드 매칭을 할 수 없기에 파드에 어피니티를 추가했다. 하지만 이 경우 어피티니가 없는 파드 역시 (특정 테인트가 있는)노드에서 실행될 수 있다.

강의에선 클러스터 내에 같은 색깔의 테인트/톨러레이션으로 노드-파드 매칭하여 스케줄 되어야 하고, (회색) 다른 클러스터의 노드와 파드도 알 경우 우리 클러스터의 것과 섞이지 않길 바라는 상황이다.

![3](/images/cka-3-scheduler/3.png)

사용하기 나름인거 같다. 실제론 어떤 식으로 사용하는지 모르겠지만, 테인트/톨러레이션을 먼저 정의하고 어피니티를 추가하는 것이, 모든 것을 deny하고 allow list를 정의하는 방화벽 정책과 비슷하다고 생각했다.

## 자원 요청과 제한
CPU, 메모리와 같은 노드의 자원은 캡(capacity)이 있기 때문에 이를 관리해줘야 한다. 즉, 스케줄러가 노드 자원 상황에 맞게 파드를 스케줄 해야한다.

파드가 필요한 자원을 스펙에 정의할 수 있다(`spec.containers[].resources.requests.(memory|cpu)`). 또 제한량도 정의한다(`spec.containers[].resources.limits.(memory|cpu)`).

여기서 CPU의 단위는 하나의 하이퍼스레드를 의미한다. AWS vCPU, GCP core 그리고 Azure core 단위와 같다고 한다(난 클라우드 경험이 많이 없지만... 쿠버네티스는 클라우드 네이티브하다). 소수점으로 표현할 수 있고 1m(milli) 즉, 0.001까지 정확도를 제공한다. CPU의 경우 제한량 이상을 쓸 수 없도록 쿠버네티스가 쓰로틀한다.

메모리는 바이트 단위이며 M(메가바이트), G(기가바이트)와 같은 이진 바이트와 Mi(메비바이트), Gi(기비바이트)와 같은 십진 바이트 suffix를 지원한다. 메모리 제한량을 초과할 경우 파드가 죽는다(난 프로세스의 CPU를 제한해 본적은 없지만, 메모리의 경우 OOM kill과 유사하게 동작한다). [참고 문서: 컨테이너 리소스 관리](https://kubernetes.io/ko/docs/concepts/configuration/manage-resources-containers/#%EB%A9%94%EB%AA%A8%EB%A6%AC%EC%9D%98-%EC%9D%98%EB%AF%B8)


## LimitRange
네임스페이스에 컨테이너(파드)의 CPU, 메모리 요청과 제한의 기본값을 정할 수 있는 어드민 리소스 오브젝트이다. 자원마다 각 주소의 오브젝트는 다음과 같다:
```yml
# https://k8s.io/examples/admin/resource/cpu-defaults.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
---
# https://k8s.io/examples/admin/resource/memory-defaults.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```
`spec.limits[]`의 `default`가 기본 제한값, `defaultRequest`가 기본 요청값이다.

## DaemonSets
지금까지 설명한 스케줄러 정책에 의해 실행되지 않는 특별한 파드 두가지가 있다. 먼저 대몬셋에 대해 알아본다.

대몬셋은 모든 노드 당 하나씩만 항상 스케줄하는 파드이다. 따라서 예시로 모든 노드에 필요한 모니터 또는 로깅 파드가 가능하다고 이야기한다. 레플리카셋 정의와 비슷하며 노드 셀렉터를 정의하여 모든 노드가 아닌 일부 노드에서만 실행할 수도 있다:

```yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```

또한 노드 어피니티도 적용할 수 있다고 한다. 대몬셋은 컨트롤플레인의 기본 스케줄러 파드(kube-scheduler)에 의해 스케줄 된다.

컨트롤플레인에서의 usecase로 kube-proxy와 네트워크 관련한 파드인 weave-net이 있다고 한다.

실행 중인 파드가 대몬셋인지 확인하는 법은 describe 출력의 'Controlled by'를 확인하는 것이다:
```sh
❯ kubectl describe po -A -n kube-system | grep ontrolledControlled By:  ReplicaSet/nginx-cert-76f7f8748f
Controlled By:  Node/minikube
Controlled By:  Node/minikube
Controlled By:  Node/minikube
Controlled By:  DaemonSet/kube-proxy
Controlled By:  Node/minikube
```

슬래시 뒤에 파드 이름이 표시되어 kube-proxy 라는걸 알 수 있지만, 나머지(스태틱 파드이다)는 어떤 파드인지 알기 어렵다. 좀 더 정확히 필터하여 보는 법과 대충? 한눈에 볼 수 있는 법이 있다.

먼저 파드 정의 `metadata.ownerReferences[].kind`를 `DaemonSet`으로 필터하자. ownerReference kind는 앞선 yaml 출력에서 `Controlled by` 필드의 슬래시 앞부분이다:
```sh
❯ kubectl get pods --all-namespaces -o json | jq -r '.items | map(select(.metadata.ownerReferences[]?.kind == "DaemonSet" ) | .metadata.name) | .[]'
kube-proxy-68nvv
```

파드의 정확한 이름까지 나온다(로컬에서 사용하는 minikube 대몬셋엔 weave-net 파드는 없었다). 또 출력 필드에 이를 지정하여 한눈에 보는 법도 있다:
```sh
❯ kubectl get pods -n kube-system -o custom-columns=NAME:.metadata.name,CONTROLLER:.metadata.ownerReferences[].kind,NAMESPACE:.metadata.namespace
NAME                               CONTROLLER   NAMESPACE
coredns-78fcd69978-mkxm8           ReplicaSet   kube-system
etcd-minikube                      Node         kube-system
kube-apiserver-minikube            Node         kube-system
kube-controller-manager-minikube   Node         kube-system
kube-proxy-68nvv                   DaemonSet    kube-system
kube-scheduler-minikube            Node         kube-system
storage-provisioner                <none>       kube-system
```

여기서 대충 보는 법이란 파드 이름 뒤를 확인하는 것이다. CONTROLLER DaemonSet은 해시 값이 붙지만, Node는 노드 이름이 따라 붙는다(`-node`).

[참고: How to identify static pods via kubectl command?](https://stackoverflow.com/questions/65657808/how-to-identify-static-pods-via-kubectl-command)

## 스태틱 파드
또 하나의 특별하게 스케줄 되는 파드 유형은 스태틱 파드이다. 이름 그대로 스케줄러에 의해 동적으로 스케줄링 되지 않는다. 노드의 쿠블릿이 직접 생성한다. 따라서 kube-apiserver가 요청하지도 않는다. 쿠블릿이 특정 경로의 파일을 직접 읽어 파드를 생성하고, 온라인으로 state를 반영한다. 쿠블릿 서비스 옵션인 `--pod-manifest-path` 또는  `--config` 옵션인 파일 안에 `staticPodPath`에 스태틱 파드 설정 파일이 있는 경로를 지정한다. 파드만 생성할 수 있으며 레플리카셋이나 디플로이먼트 같은 파드 리소스는 불가하다.

`docker ps`나 위에서 설명한 방법으로 `kubectl get/describe`로 확인 가능하다. 나는 로컬 맥에선 Docker Desktop을 지우고 minikube + hyperkit을 사용하기 때문에 후자의 방법으로 확인했다(이번엔 ownerReference kind가 Node이다):
```sh
❯ kubectl get pods --all-namespaces -o json | jq -r '.items | map(select(.metadata.ownerReferences[]?.kind == "Node" ) | .metadata.name) | .[]'
etcd-minikube
kube-apiserver-minikube
kube-controller-manager-minikube
kube-scheduler-minikube
```

그러나 edit은 불가하다. 설명한대로 스태틱 파드 설정 파일을 변경하면 쿠블릿이 반영한다. 컨트롤플레인의 usecase로 controller-manager, apiserver, etcd, scheduler가 있다.


## Multiple Schedulers
기본, 컨트롤플레인의, 스케줄러 외에도 스케줄러를 커스텀하거나 그런 스케줄러를 여러개 쓸 수 있다. 이 부분은 버전마다 분기하는 부분도 있는거 같고, 강의 연습문제를 풀 땐 처음으로 답을 봤다. 강의 설명만으론 불충분했다 😑.

기본 스케줄러의 스태틱 파드 정의 파일을 복사해서 만들면되는데 `containers[].command`의 서비스 옵션 중:
- `--scheduler-name`을 따로 지정하고
- `--leader-elect=false`로 하면 되는거 같다(스케줄러 고가용성 확보 시 multi-leader를 위한 옵션 같다).

이렇게 생성한 커스텀 스케줄러는 파드 스펙의 `schedulerName`에서 특정해서 스케줄되게 할 수 있다([참고 문서: Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)):
```yml
apiVersion: v1
kind: Pod
metadata:
  name: annotation-default-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: default-scheduler
  containers:
  - name: pod-with-default-annotation-container
    image: k8s.gcr.io/pause:2.0
```

스케줄링 그리고 앞에서 설명한 리소스 초과는 모두 이벤트에 기록된다. `kubectl get events -o wide`를 출력하면 SOURCE 필드에 스케줄러를 확인할 수 있다.
