---
title: "Yq 중첩 키 경로 파악하기(긴 Helm 차트 values 파악하기)"
date: 2022-06-20T19:40:31+09:00
tags:
- yq
- kubernetes
---

Yq의 [globbing](https://mikefarah.gitbook.io/yq/operators/recursive-descent-glob)과 내장 연산자 몇가지를 사용하면 중첩된 객체에서 특정 키 경로를 알아낼 수 있다:
```sh
❯ cat test.yaml
a:
  b:
    target: HERE

❯ yq e ' .. | select(has("target")) | path' test.yaml
- a
- b
```

실전으론, 아주 긴 쿠버네티스 매니페스트나 Helm values를 파악할 때 사용할 수 있다. 하지만 내가 쓰는 방법이 완전한 해결책이라는 생각이 들진 않는데... 일단 지금 쓰는 방식을 이야기 해본다.


한 예로 [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack)의 values는 아주 길다:
```sh
$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  wc -l
2856
```

## Grepping

문제 상황은 tolerations을 적용할 수 있는 곳을 모두 알고 싶다 이다. 우선은 grep 한다:
```sh
$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  grep tolerations
    ## If specified, the pod's tolerations.
    tolerations: []
      tolerations: []
  tolerations: []
    tolerations: []
```

인덴트를 보면 최상위 수준에 있지 않다는 것은 알 수 있다. Stack이란 이름처럼 여러 컴포넌트가 있는만큼 각 컴포넌트 매니페스트의 values가 분기되어 있을 것 같다.

## 주석 제거

결과 중엔 주석도 볼 수 있다. 특히 Helm values는 예시나 문서를 주석을 쓰는 경우가 많다. 결과가 복잡하니 [주석을 지우자](https://mikefarah.gitbook.io/yq/operators/comment-operators#remove-comment):
```sh
$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments=""' | wc -l
734

$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments=""' | grep tolerations
    tolerations: []
      tolerations: []
  tolerations: []
    tolerations: []
```

YAML 코드보다 주석이 더 많았다.

## Top keys

이 경우엔 values의 최상위 키 중 각 stack의 컴포넌트라가 있다는 기반 지식이 있다(global처럼 아닌 것도 있다). 어떤 컴포넌트가 tolerations 키를 갖고 있는지 보자:
```sh
$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments="" | keys'
- nameOverride
- namespaceOverride
- kubeTargetVersionOverride
- kubeVersionOverride
- fullnameOverride
- commonLabels
- defaultRules
- additionalPrometheusRulesMap
- global
- alertmanager
- grafana
- kubeApiServer
- kubelet
- kubeControllerManager
- coreDns
- kubeDns
- kubeEtcd
- kubeScheduler
- kubeProxy
- kubeStateMetrics
- kube-state-metrics
- nodeExporter
- prometheus-node-exporter
- prometheusOperator
- prometheus

$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | yq e '... comments="" | with_entries(select( .. | has("tolerations"))) | keys'
- alertmanager
- prometheusOperator
- prometheus
```

Grep 결과는 4개였지만, 최상위 키로 필터하면 세개이다. 어떤 tolerations는 최상위 키 기준으로 같은 노드에 있음을 알 수 있다.

## Path

이 정도 파악하고 [path 연산자](https://mikefarah.gitbook.io/yq/operators/path)를 사용하면 중첩키의 경로가 보일 것이다:
```sh
$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments="" |  .. | select(has("tolerations")) | path'
- alertmanager
- alertmanagerSpec      # 1 .alertmanager.alertmanagerSpec.tolerations
- prometheusOperator    # 2 .prometheusOperator.tolerations
- prometheusOperator
- admissionWebhooks
- patch                 # 3 .prometheusOperator.admissionWebhooks.patch.tolerations
- prometheus
- prometheusSpec        # 4 .prometheus.prometheusSpec.tolerations
```


결과의 주석은 설명을 위해 내가 추가한 것이다. Top keys를 살펴 봤을 때 prometheusOperator 아래에 두 개의 tolerations 키를 포함한 노드가 있다는 것을 알 수 있다. 따라서 네개의 tolerations까지의 경로를 알 수 있게 됐다.

## 뭔가 부족?

문제를 프로그래매틱한 방법으로 일반화하여 푼 것 같지만, 사실 다른 yq 연산자로 많이 시도하고 결국엔 valuse YAML을 눈으로 직접 보면서 tolerations 경로를 맞게 찾은건가? 검증했다.

위 결과에서 주석을 빼면 알아보기 쉽지도 않다. 나온 네개의 노드 각각에 대해 path 연산을 하고 싶은데, 배열로 묶으면 새로운 객체가 되어 노드의 경로 정보가 사라져버린다...:
```sh
$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments="" |  [ .. | select(has("tolerations")) ] | path'
[]


$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments="" |  [ .. | select(has("tolerations")) ][] | path'
- 0
- 1
- 2
- 3

$ helm show values prometheus-community/kube-prometheus-stack --version 36.0.2 | \
  yq e '... comments="" |  [ .. | select(has("tolerations")) ][0] | path'
- 0
```

한끝차이로 일반적인 방법으로서의 완성도를 못 만들고 있는거 같은데... 잘 모르겠다.


아니면 역시 눈으로 확인하는 법이 빠를 수도 있다. [dive](https://github.com/wagoodman/dive) 처럼 노드 트리를 접고 펼치면서, 검색 기능이 있는 TUI 앱을 만들어야 할까?(있지 않을까?). Helm values를 찾기 위해 만든다면 엄청난 오버킬이겠지만, dive 비슷한 앱을 만들어 보고 싶긴 하다.

---

## 참고
- https://mikefarah.gitbook.io/yq/
