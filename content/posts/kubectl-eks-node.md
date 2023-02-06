---
title: "간단한 kubectl 플러그인을 간단하게 만든 이야기(eks-node)"
date: 2023-02-07T02:03:54+09:00
tags:
- kubernetes
---

블로그를 안한 사이 이직을 했고 여기선 쿠버네티스, AWS EKS, 를 사용해 제품을 운영하고 있다. EKS를 쓰긴하지만, 콘솔이나 `eksctl`가 아닌  `kubectl`를 더 많이 써서 작업하는 편이다. 그럴때마다 굉장히 불편하게 반복하는 일들이 몇가지 있다.
1. 특정 노드에서 실행 중인 파드를 확인; 이 파드 돌고 있는 노드엔 또 어떤 파드들이 있지?
2. 특정 노드의 정보 확인; 인스턴스 타입, 노드 그룹, 가용영역 등

이런 작업들이 왜 짜증났냐 하면...

## 뭐가 문제였나
첫번째, 다시 말하면 "특정 파드와 같은 노드에서 돌고 있는 다른 파드를 볼 땐", **항상 두번 명령을 썼다.**
```sh
$ k get po pod-name -owide # 노드 이름을 찾기 위해
$ k get po -A -owide | grep ip-xxx-yyy-zzz-aaa.region-code.compute.internal
```
물론 한 줄(?)로도 만들 수 있다.
```sh
$ k get po -A -oyaml | nodename=$(k get po pod-name -oyaml | yq .spec.nodeName) yq '.items[] | select(.spec.nodeName=env(nodename)).metadata.name'
```
하지만 `pod-name`을 인자로 넣기도 만만치 않고, 환경변수 사용과 파이프, 여러 명령 사용으로 복잡하다. 두 번 명령 쓰는것보다 크게 낫지 않다는 이야기다.

두번째 작업, "노드의 인스턴스 타입, 노드 그룹 또는 가용 영역을 알아내는 작업"은, 레이블로 확인할 수 있지만 **레이블 이름이 너무 긴 것이 문제였다.**
```sh
$ k get no ip-xxx-yyy-zzz-aaa.region-code.compute.internal -oyaml | yq .metadata.labels
...
beta.kubernetes.io/instance-type: t3.xlarge
...
eks.amazonaws.com/nodegroup: base
...
kubernetes.io/hostname: ip-xxx-yyy-zzz-aaa.region-code.compute.internal
...
node.kubernetes.io/instance-type: t3.xlarge
...
topology.kubernetes.io/zone: ap-northeast-2a
```

## 간단한걸 만들었다
[eks-node](https://github.com/flavono123/kubectl-eks-node)는 이런 문제를 해결하기 위해, 즉 위 같은 반복 작업을 간단하게 하기 위해 만들었다. [사용법(usage)](https://github.com/flavono123/kubectl-eks-node#usage)도 나름 썼으니 설치법 소개(=홍보)부터한다:
```sh
$ k krew index add flew https://github.com/flavono123/flew-index.git
$ k krew install flew/eks-node
```
[krew](https://krew.sigs.k8s.io/)로 설치할 수 있도록 했다(관련 자세한 이야기는 후술한다). krew를 쓰지 않는다면 [README의 다른 설치법](https://github.com/flavono123/kubectl-eks-node#install) 을 사용해보길.

eks-node는 앞선 문제들을 해결하는 데모를 보면:
```sh
$ k eks-node ip-xxx-yyy-zzz-aaa.region-code.compute.internal
{
  "ip-xxx-yyy-zzz-aaa.region-code.compute.internal": {
    "nodegroup": "base",
    "instance_type": "t3.xlarge",
    "zone": "ap-northeast-2a",
    "pods": [
      "pod-name",
      "another-pod-name",
      ...
    ]
  }
}
```
기본 사용법으로 인자에 노드 이름을 주면 노드 그룹, 인스턴스 타입, 가용 영역과 노드에 실행 중인 모든 파드를 출력한다. 이렇게 첫번째 두번째 문제를 인자 하나, 노드 이름으로 해결한다. 하지만 첫번째 경우에 "파드 이름 > 노드 이름 > 다른 파드"로 찾았기 때문에 파드를 인자로 받는 방법도 제공한다.
```sh
$ k eks-node po pod-name
{
  "ip-xxx-yyy-zzz-aaa.region-code.compute.internal": {
    "nodegroup": "base",
    "instance_type": "t3.xlarge",
    "zone": "ap-northeast-2a",
    "pods": [
      "pod-name",
      "another-pod-name",
      ...
    ]
  }
}
```
즉 노드 이름을 몰라도, 파드 이름만 알더라도 같은 정보를 볼 수 있다.

부가적으로 노드 그룹, 인스턴스 타입 그리고 가용 영역으로 노드를 필터해볼 수 있도록 했다.
```sh
$ k eks-node ng base
NAME                                                 STATUS   ROLES    AGE   VERSION
ip-xxx-yyy-zzz-aaa.region-code.compute.internal      Ready    <none>   60d   v1.23.9-eks-ba74326
ip-xxx-yyy-zzz-bbb.region-code.compute.internal      Ready    <none>   42d   v1.23.9-eks-ba74326
...

$ k eks-node ins t3.xlarge
NAME                                                 STATUS   ROLES    AGE   VERSION
ip-xxx-yyy-zzz-aaa.region-code.compute.internal      Ready    <none>   60d   v1.23.9-eks-ba74326
ip-xxx-yyy-zzz-bbb.region-code.compute.internal      Ready    <none>   42d   v1.23.9-eks-ba74326
ip-xxx-yyy-zzz-ccc.region-code.compute.internal      Ready    <none>   42d   v1.23.9-eks-ba74326
...

$ k eks-node az ap-northeast-2a
NAME                                                 STATUS   ROLES    AGE   VERSION
ip-xxx-yyy-zzz-aaa.region-code.compute.internal      Ready    <none>   60d   v1.23.9-eks-ba74326
ip-xxx-yyy-zzz-ddd.region-code.compute.internal      Ready    <none>   42d   v1.23.9-eks-ba74326
...
```
특히 인스턴스 타입이나 노드 그룹은, 같은 인스턴스 타입/노드 그룹의 다른 노드는 상태가 어때? 이런게 궁금했기 때문이다.

## 간단하게 만들었다
eks-node의 현재 버전은([v0.1.1](https://github.com/flavono123/kubectl-eks-node/releases/tag/v0.1.1)) 쉘 스크립트를 사용해 만들었다. 그래서 앞서 설명한, 파이프로 연결된 한 줄 명령과 `yq` 사용을, 실제 플러그인에도 거의 비슷하게 구현했다. 아마 `kubectl`을 자주 쓰는 사람이라면 `yq(jq)`가 익숙할거 같다.

그런데  쉘 스크립트로 krew 플러그인을 만들어 볼 도전을 할 계기를 만들어 준 빌드업이 있었다. **"쉘에서 어떻게든 한 줄 짜리 명령으로 끄적거린 이런게 플러그인을 만들어도 돼?"라는 물음에 답을 준** 다음 경험이다.

### 원래 그런거네?
[krew 메인 index](https://krew.sigs.k8s.io/plugins/)에도 등록된, [prompt](https://github.com/jordanwilson230/kubectl-plugins/tree/krew#kubectl-prompt) 라는 플러그인이 있다. `kubectl`의 파괴적인 명령, 예를 들어 `create`, `delete` 처럼 클러스터의 상태를 바꾸는, `get` 같지 않은 명령들을 쓸 때 yes or no?의 확인 프롬프트를 띄워준다. 또 프롬프트를 띄울지 말지 컨텍스트나 네임스페이스 별로 정할 수 있다.

회사에서, 중요한 컨텍스트 또는 네임스페이스에서 실수로 명령을 쉽게 날리는 걸 막기 위해, 이 플러그인을 사용했다. 그런데 파드 생성하는 `run` 명령이 프롬프트로 막히지 않았다. 검증을 `run`으로 했기에 살짝 어이 없어하며 코드를 살펴봤는데, 왠걸 [bash로 작성돼 있었다](https://github.com/jordanwilson230/kubectl-plugins/blob/krew/kubectl-prompt#L1). 그래서  [`run` 명령을 지원하는 PR](https://github.com/jordanwilson230/kubectl-plugins/pull/47)을 쉽게 만들었다(merge는 아주 빨리됐지만, 릴리즈는 한참 늦거나 안할거 같아 난 로컬 쉘 스크립트를 수정해서 쓰고 있다). 

이것이 kubectl krew 플러그인을 만만하게(?) 보게 된 첫번째 썰이다.

### ctx, ns 너네도?
쿠버네티스, 특히 `kubectl` 을 많이 쓰고 또 `krew` 역시 사용한다면, `ctx`와 `ns` 두 플러그인은 정말 많이 쓰고 있을 것 같다. 이 두 플러그인 역시 bash 스크립트이다. 생각해보면 컨텍스트나 네임스페이스를 교체하는 일은 `kubectl` 으로도 할 수 있지만 명령이 길어 사용하기 불편하다. 그걸 alias 수준으로, 서브 커맨드(플러그인)으로 짧게 제공하는 것이, 이 둘을 포함한, 대부분의 krew 플러그인의 구현이다.
- ctx: `k config use-context <context>`([github](https://github.com/ahmetb/kubectx/blob/master/kubectx#L111))
- ns: `k config set-context <context> --namespace=<namespace>`([github](https://github.com/ahmetb/kubectx/blob/master/kubens#L103))

유명한 플러그인까지 쉘(bash)로 간단하게 구현되어 있으니, eks-node를 만들 때 거침 없이 쉘 스크립트로 만들 수 있었다. 특히 ctx는 만들며 참고를 많이 했다.

## 대단해(?)져 보려고 했다
대단해져 보려한 것은, 다름이 아닌, krew index에 등록하는 것이었다. 나도 쟤네(?)처럼 쉘 스크립트로, 하는 일이라곤 alias 같은걸 만들었는데, 전 세계 쿠버네티스, krew 사용자들이 사용할 수도 있다니, 놓치고 싶지 않은 기회였다.

결과부터 말하면 메인 krew index에 등록하는 것은 실패했다. 하지만 그 과정에서 경험한 것이 꽤 재밌어서 공유한다.

### 이게 무슨 케밥 스네이크 케이스?(kebab-snake_case)
eks-node의 실행 파일명을 보면 누군가는 경악을 할 수도 있다.

`kubectl-eks_node` ...

으엑 네이밍 컨벤션도 모르는 개노답 개발자! 라고 할지도 모르나 이건 [또 다른, kubectl 플러그인의, 네이밍 컨벤션](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/#names-with-dashes-and-underscores)이다.

또한 [명료해야 한다는(Be specific) krew 플러그인 네이밍 가이드](https://krew.sigs.k8s.io/docs/developer-guide/develop/naming-guide/)도 따르기 위해 `eks-node` 라고 지었다. krew에 내보겠다는 욕심이 없었으면 `eksn`나 `eno` 정도로 짧게 하지 않았을까 싶다.

### PR 만들기
[제출 전 체크리스트](https://krew.sigs.k8s.io/docs/developer-guide/release/new-plugin/) 의 1,2번은 열심히 한 셈이고 Github에서 릴리즈를 만들거나 태그에 semver를 적용하는 일은 쉬운 편이니 생략한다. [로컬에서 설치 테스트](https://krew.sigs.k8s.io/docs/developer-guide/testing-locally/)도 해보았다. 어차피 이 때 사용하는 `Plugin` 매니페스트가 PR 내용의 전부이다(커스텀 그룹이다 `krew.googlecontainertools.github.com`). 이번에도 [merge된 ctx의 PR](https://github.com/kubernetes-sigs/krew-index/pull/309/files)을 참고하여 쉽게 만들었다.

그렇게 PR을 열었는데 의외의 CI에 걸렸다.

먼저 **CLA(Contributor License Agreement)** 라는 것을 처음보게 됐는데, 내가 **NOT COVERED** 개발자라며 빨간 딱지가 떡하니 붙었다.
![1.png](/images/kubectl-eks-node/1.png)

(순간 길에서 경찰차를 만났을 때의 오묘한 무서움(?)이 들었다)

구글링을 좀 해보면, CLA는 오픈소스 기여자의 라이선스를 보장하고 법적 분쟁을 최소화하려는 장치인것 같다. 쿠버네티스가 리눅스 재단 프로젝트이니 이런 것을 확실히 하려는 것 같았다(오픈소스 라이선스 가지구 장난해서 뺏어가고 팔아먹고 싸우고 어쩌구 저쩌구가 떠올랐다). PR CI 코멘트로 뜬  NOT COVERED를 누르면 CLA에 가입하는 링크가 뜬다. 계속 넘어가면 도큐사인에서 이메일로 계약서가 오고 CLA 서명이 금방 동기화 되어 [CI를 통과](https://github.com/kubernetes-sigs/krew-index/pull/2899#issuecomment-1407689149)할 수 있다.
![2.png](/images/kubectl-eks-node/2.png)

그 다음에 걸린 CI는 validation이었는데, 코드의 문법 같은 문제가 아니라, 내 **플러그인에 라이선스가 없다는 것**이었다.

[ctx의 `Plugin`을 참고하며 라이선스를 써준걸 보긴했지만](https://github.com/kubernetes-sigs/krew-index/pull/314/commits/a287c488f8c2cc6c3b09f0d3c50ba07832286997#diff-97748851f4a91719edddf21f156b5e608a1042507c5108d5d1d3cd948474d6e3R32-R33), 대수롭지 않게 여겼는데, 앞선 CLA와 비슷한 이유로 이를 강제하는 것 같았다. 평소에 Github 레포에 라이선스를 잘 달아보지도 않았고, 신중하게 고르는게 큰 의미 없다고 생각해서 ctx와 같은 [Apache 2.0](https://github.com/flavono123/kubectl-eks-node/blob/main/LICENSE)으로 했다(그리고 CI를 빨리 패스시키고 싶었다). (그런데 다시 한번 살펴보니 제출 전 체크리스트에 라이선스 챙기라고 두번이나 강조하고 있었다 ...ㅎㅎ)

이렇게 리뷰 받을 준비를 마치고 기다렸건만, [서윗한 리젝 코멘트](https://github.com/kubernetes-sigs/krew-index/pull/2899#issuecomment-1409475026)와 함께 PR은 닫혔다(그리고 [리뷰어가 ctx/ns 개발자이자 이젠 krew index 멤버였다](https://github.com/ahmetb)). 사실 기대는 크게 하고 있지 않아서 괜찮았지만, 의외의 수확이 몇가지 있었다. 우선 코멘트가 꽤 구체적이었는데:
- 너무 간단한 기능의 플러그인이다.
- 이런 방향으로 확장시켜보는게 어떻겠냐?(e.g. 멀티 클라우드 지원).
- 계획은 말해 뭐해 ^^ 만들어 가지고와.
- [커스텀 krew index에 올릴 수도 있다.](https://krew.sigs.k8s.io/docs/developer-guide/custom-indexes/)

라고 해주어서, 가장 쉽게 할 수 있는 마지막 옵션을 사용했다. 나만의 [krew index](https://github.com/flavono123/flew-index)를 만들었다.

또, 비록 구현은 간단할지라도, 꽤 큰(?), 컨벤션이 잘 갖춰진, 쿠버네티스 분과(SIG) 오픈소스에 기여를 시도한 것에서 배운 점도 많았다. 특히 마지막의 경험은, 엄격한 CI를 만드는걸 좋아하는 나로썬, 아주 흥미로웠다.

사실 처음부터 어느정돈 krew index를 목표로 만들긴 했으나, 막상 PR을 만들기 전까진 고민도 꽤 있었다.
ctx/ns만큼 유용한가? 대중성이 있을까?(아직도 난 내가 만든 EKS 노드 패턴이 다 이렇게 쓰는지 우리 회사만 그런지 잘 모른다) Github Star를 모아서 갈까? 사용법을 GIF로 만들고 로고쯤은 만들어야 먹힐까? 등등. 결국엔 그럴 필요 없이 PR부터 찔러본게 여기까지 경험의 지름길이 된거 같다.

## 대단한 플러그인이 되기

> - 계획은 말해 뭐해 ^^ 만들어 가지고와.

>   (And we typically merge plugins based on their current status; not their planned additions.)

희망고문을 받았기 때문일까? 난 아직 krew index 입성을 포기하지 않았다. 우선 단기적으론 쉘 스크립트를 Go로 재구현해서 몇가지 기능 추가와  `yq` 의존성을 제거할 것이다. 코멘트의 "멀티 클라우드 지원"은 당장 계획에 없다. 내가 회사에서 안쓸거 같아서... 하지만 혹시 모르지 않나? 누군가 함께 해준다면, 같이 대단한 일을 할 수 있을지도?

글을 보시는 분들 중에 관심이 있다면 참여를 바란다. 아이디어, 의견 아니 star라도(ㅎㅎ) 어떤 형태의 관심과 참여도 좋을거 같다. 가장 좋은 것은 다운 받아 사용해보고 피드백을 주는 것이겠다. 혹시 이 플러그인을 보고 나처럼 "만들기 진짜 쉬운거네?"라는 생각으로 여러분의 것을 만든다면, 그것도 꽤 멋있는 나의 기여라고 생각한다.

https://github.com/flavono123/kubectl-eks-node
