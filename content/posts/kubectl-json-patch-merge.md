---
title: "Kubectl Patch"
date: 2022-06-21T18:46:55+09:00
tags:
- kubernetes
- json
---

## JSON Patch

[kubectl patch](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#patch) 서브명령은 바꿀 JSON을 인자로 실행 중인 객체의 정의를 바꾼다. [edit](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#edit) 보다 편하기도, 또 누군가에게 설명하기 쉬운 경우도 많다.

하지만 인라인으로 쓰인 patch 인자가 복잡해 보일 때도 있다. 이때 보다 나은, [JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902)를 사용할 수 있다:
```sh
$ k run patch-test --image busybox -- sleep 1d
pod/patch-test created

# k label po patch-test test=true 와 같다
$ k patch po patch-test --type json \
  -p '[{"op": "add", "path": "/metadata/labels/test", "value": "true"}]'
pod/patch-test patched

$ k get po -l test=true
NAME         READY   STATUS    RESTARTS   AGE
patch-test   1/1     Running   0          108s

```

Patch할 JSON에 여러 변경사항(연산)을 인자로 쓸 수 있지만, 하날 쓰더라도 **무조건 배열에 담아 보내야 하는 것**을 주의하자. Add 외에 몇가지 연산자가 더 있다. 이걸 JSON을 그대로 보내는, patch의 기본 전략을 사용하면, 인자 JSON이 꽤 복잡해보인다. 어디를 어떻게 고치는지 잘 보이지 않는다. 예시는 간단해서 이 정도면 파악이 어렵진 않지만... 불필요한 따옴표와 중괄호가 너무 많다:
```sh
$ k patch po patch-test -p '{"metadata": {"labels": {"test": "false"}}}'
pod/patch-test patched
```

연산자와 키 경로라는 명령 스키마가 있다는 것이 하려는 동작을 보다 명확하게 보여준다. Ansible에서도 그렇다. 전자는 [kubernetes.core.k8s_json_patch](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_json_patch_module.html#ansible-collections-kubernetes-core-k8s-json-patch-module)라는 분리된 모듈에 있고, 후자는 kubernetes.core.k8s 모듈에 [state 값이 patched](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html#parameter-state)일 때 `definition`을 인자 JSON으로 patch 동작을 한다:

```yaml
# JSON Patch
- kubernetes.core.k8s_json_patch:
    kind: Pod
    name: patch-test
    patch:
    - op: add
      path: /metadata/labels/test
      value: "true"

# JSON Merge patch
- kubernetes.core.k8s:
    state: patched
    kind: Pod
    name: patch-test
    definition:
      metadata:
        labels:
          test: "true"
```

JSON Patch를 보면, 키 이름에 역 슬래시('/')가 들어간 경우 처리할 방법이 있어야 한다. 실제로 레이블이나 어노테이션 키엔 역 슬래시가 많이 들어간다. [JSON Patch path 에선 '/' 를 '~1'로, '~'를 '~0'로 처리](https://datatracker.ietf.org/doc/html/rfc6902#appendix-A.14)한다. 다시 말하면 다음처럼 매핑한다:
- `~` - `~0`
- `/` - `~1`

다음은 `.metadata.annotations.storageclass.kubernetes.io/is-default-class` 키를 수정하는 예제이다:
```yaml
- kubernetes.core.k8s_json_patch:
    kind: StorageClass
    name: local-path
    patch:
    - op: replace
      path: /metadata/annotations/storageclass.kubernetes.io~1is-default-class
      values: "true"

```

## JSON Merge patch

후자에 해당하는 방법은 자세히 설명하지 않았는데, [JSON Merge patch](https://datatracker.ietf.org/doc/html/rfc7386)라고 한다. 따로 설명이 필요 없을만큼 직관적인 동작이라, 아마 JSON Patch에 merge 연산자를 추가하지 않고 새 표준으로 만든 것 같다.

[예제 테스트 케이스](https://datatracker.ietf.org/doc/html/rfc7386#appendix-A)를 보면 patch하는 JSON 중 null 값인 키는 제외된다는 점이 눈에 띈다. 하지만 이로 인해 문제되는 경우는 kubectl API validation이 막아주지 않을까 싶다. 값의 타입이 바뀌는 경우도 마찬가지이다.


## type=strageric

엄밀히 말하면, kubectl patch의 기본 동작은 JSON Merge patch가 아니다. [태스크 문서 설명](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/#notes-on-the-strategic-merge-patch)을 보면 전략적 병합(strageric merge)라고, 파드의 컨테이너의 경우(.spec.containers) 완전히 patch JSON으로 교체되지 않고 추가된다고 설명한다. 이는 kubectl API의 구현인데, 거의 대부분은 JSON Merge patch이긴 하다.


```sh
# "x-kubernetes-patch-strategy" 키의 값 별로 카운터
$ wget -qO - https://raw.githubusercontent.com/kubernetes/kubernetes/master/api/openapi-spec/swagger.json | \
  jq '[ .. | select(."x-kubernetes-patch-strategy"? != null) | ."x-kubernetes-patch-strategy"] \
    | group_by(.)[] | {strategic: .[0], value: length}'
{
  "strategic": "merge",
  "value": 42
}
{
  "strategic": "merge,retainKeys",
  "value": 1
}
{
  "strategic": "replace",
  "value": 1
}
{
  "strategic": "retainKeys",
  "value": 1
}
```


## 정리
- kubectl patch의 기본 전략은 거의 대부분 JSON Merge patch이다.
- JSON Merge patch의 동작은 직관적이지만, patch하는 JSON이 길어지면 알아보기 어렵다.
- JSON Patch는 Merge patch에 비해 연산과 대상을 알아보기 쉽다.

---

## 참고
- https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/
- https://datatracker.ietf.org/doc/html/rfc6902
- https://datatracker.ietf.org/doc/html/rfc7386

