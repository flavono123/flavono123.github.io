---
title: "HPA"
date: 2022-06-20T10:27:40+09:00
tags:
- kubernetes
---

쿠버네티스에서 오토스케일은 크게 세가지가 있다. 클러스터 오토스케일러, 수직적 파드 오토스케일러(Vertical Pod Autoscaler; VPA) 그리고 수평적 파드 오토스케일러(Horizontal Pod Autoscaler)이다. 클러스터는 노드 수준 그리고 뒤의 두가지는 파드 수준의 오토스케일이다.

클러스터 오토스케일러는 노드 개수를 스케일 아웃/인 한다. VPA는 파드에 할당되는 컴퓨팅 자원(CPU, 메모리의) 스케일 업/다운을 그리고 **HPA는 파드 개수(replicas)의 스케일 아웃/인** 한다([HPA 관련한 쿠버네티스 문서엔 전부 스케일 업/다운이라는 표현을 쓰지만](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/) 통상적인 의미의 스케일 아웃/인이라고 해석했다).

클러스터 오토스케일러와 VPA는 쿠버네티스 API로 제공되지 않고 [저장소](https://github.com/kubernetes/autoscaler)도 분리되어 있다. 이와 달리, 이번에 집중하여 볼 것인, HPA는 쿠버네티스 API로 제공되는 기능이다. [이 태스크 문서](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)를 중심으로 살펴 볼 것이다. 그런데 외부 컴포넌트를 추가해야 된다던가 생소한 개념이 많다(개인적으론 문서도 쿠버네티스의 것 치곤 최신화나 설명이 부실하다고 느꼈다). 완전히 이해는 못했지만 허들을 다 넘기엔 러닝커브가 너무 큰거 같아서, 쭉 공부하려던 내용을 끊고 이해한 것을 우선 정리한다.

## Metrics server 설치

[Metrics server](https://github.com/kubernetes-sigs/metrics-server) 쿠버네티스 클러스터의 노드, 파드의 지표를 수집하고 오토스케일을 위해 이를 API로 제공하는 외부 컴포넌트이다. [Helm chart](https://artifacthub.io/packages/helm/metrics-server/metrics-server)가 있다. 여담으로 이번에 Ansible [kuberenetes.core](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html) 모듈의 존재를 알게 되어, 이걸 사용해 설치하고 [기존 command 태스크도 전부 바꾸었다](https://github.com/flavono123/kubernetes-the-hard-way/commit/5a2c7a96241c310f075fb662fcbcdab901428b20). 또 Ansible의 기본 멱등성 검사를 보완할 수 있는 [helm-diff](https://github.com/databus23/helm-diff) Helm 플러그인도 설치해줬다.


Metrcis server가 동작하기 위해선 각 노드 쿠블릿과 TLS로 통신해야한다. 이 [문서](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs) 지시사항을 따르면 된다. `kube-system`의 `kubelet-config-<major>.<minor>` 컨피그맵의 `kubelet` 키와 각 노드 쿠블릿 설정 파일(/var/lib/kubelet/config.yaml)에 `serverTLSBootstrap: true`를 추가한다. [`kubeadm init` 시 부트스트랩 쿠블릿 설정(/etc/kubernetes/bootstrap-kubelet.conf)을 바꾸고 재시작 하는 방법](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/_print/#workflow-when-using-kubeadm-join)은 동작하지 않았다. 그래서 init, join 후 모든 쿠블릿 설정 파일을 직접 바꾼 후 재시작했다:
```yaml
  - name: Get the kubelet-config ConfigMaps
    kubernetes.core.k8s_info:
      kind: ConfigMap
      name: kubelet-config-1.23
      namespace: kube-system
    register: configmap_result
    when: "'controlplane' in group_names"

  - name: Patch serverTLSBootstrap to kubelet-config
    kubernetes.core.k8s_json_patch:
      kind: ConfigMap
      namespace: kube-system
      name: kubelet-config-1.23
      patch:
      - op: replace
        path: /data/kubelet
        value: "{{ configmap_result.resources[0].data.kubelet | from_yaml | combine({'serverTLSBoostrap': true }) | to_yaml }}"
    when: "'controlplane' in group_names"

  - name: Get current kubelet config
    shell: cat /var/lib/kubelet/config.yaml
    register: kubelet_config

  - name: Add serverTLSBootstrap to kubelet config
    set_fact:
      kubelet_config: "{{ kubelet_config.stdout | from_yaml | combine({'serverTLSBootstrap': true}) | to_yaml(indent=2, width=1337) }}"

  - name: Copy new kubelet config
    copy:
      content: "{{ kubelet_config }}"
      dest: /var/lib/kubelet/config.yaml
      mode: "0644"
      owner: root
      group: root
    notify: restart kubelet

```


YAML 다루는 기술은 늘었지만 코드 가독성이 좋아보이진 않는다. 이렇게 설정만 바꾸면 CSR이 자동으로 생성되고 승인하면 쿠블릿 포트인 10250에 대해 TLS 통신이 가능해진다. 이 때 만드는 CSR(singer는 `kubernetes.io/kubelet-serving`이고 requestor는 노드 `system:node:<hostname>`)은 [컨트롤러 매니저에 의해 절대 자동 승인 되지 않는다고 한다](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers).

따라서 이런 종류의 인증서 rotation을 위한 [서드 파티](https://github.com/postfinance/kubelet-csr-approver)도 추천해주는데, 오버킬이라 생각하여 Ansible 모듈을 사용해서 CSR을 승인했다. 모듈에 `certificate approve`에 해당하는 것이 없어 command로 했다:
```yaml
  - name: Get CSRs
    kubernetes.core.k8s_info:
      kind: CertificateSigningRequest
      api_version: certificates.k8s.io/v1
      field_selectors: spec.signerName=kubernetes.io/kubelet-serving
    register: csr_result
    when: "'controlplane' in group_names"

  - name: Show CSRs
    debug:
      msg: "{{ csr_result.resources | map(attribute='metadata.name') | list }}"
    when: "'controlplane' in group_names"

  - name: Approve CSRs
    command: "kubectl certificate approve {{ item }}"
    with_items: "{{ csr_result.resources | map(attribute='metadata.name') | list }}"
    when: "'controlplane' in group_names"
```

이 작업은 metrics server Helm 차트 설치 이후에 해주어도 된다. 그러면 readiness가 0/1이었던 metrics server 파드가 정상으로 바뀔 것이다.

이제 Metrics server에 API 요청을 해볼 수 있다:
```sh
$ k get --raw /apis/metrics.k8s.io/v1beta1 | jq
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "nodes",
      "singularName": "",
      "namespaced": false,
      "kind": "NodeMetrics",
      "verbs": [
        "get",
        "list"
      ]
    },
    {
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "PodMetrics",
      "verbs": [
        "get",
        "list"
      ]
    }
  ]
}
```

## HPA

HPA가 지표 기반으로 파드 수를 조정하는 공식의 간단한 버전은 다음과 같다:

> X  = N * c/t

- N - 현재 파드 개수
- c - 현재 지표 값
- t - 목표 지표 값
- X - 조정될(desired) 파드 개수

실습의 예시로 설명해보면, 목표 지표 값이 파드당 CPU 사용 백분율(`targetCPUUtilizationPercentage`) 50%일 때, 측정되는 지표 값이 300%라면 파드 개수는 현재의 여섯배로 조정될 것이다(300/50 * 1 = 6).

개념이 아주 간단하고 실습도 크게 어려운 내용이 없어 명령을 그대로 따라하면 결과를 볼 수 있다:
```sh
# HPA 적용할 디플로이먼트 생성, 서비스 노출
$ k apply -f https://k8s.io/examples/application/php-apache.yaml
deployment.apps/php-apache created
service/php-apache created

# HPA 생성
$ k autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled

# 초반엔 목표 지표 값이 <unknown>으로 나올 수 있다
$  k get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          94s

# 부하 생성
$ k run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; d do wget -q -O- http://php-apache; done"


# 모니터
## HPA
$ watch 'kubectl get hpa'

## 파드
$ watch 'kubectl get po'

```

부하가 증가하여 HPA의 현재 지표 값이 올라가면 파드가 늘어나는 것을 확인할 수 있다. 문서 설명처럼 7개 파드에선 CPU 사용률이 50% 미만으로 유지되어 더 이상 증가하지 않았다. 부하 생성하는 파드를 종료하여 지표 값이 낮아지면 파드 개수가 다시 1개로 줄어든다.

## v1 vs. v2

여기까진 문서를 보고 실습 따라함에 큰 무리가 없다. 그런데 HPA spec의 targetCPUUtilizationPercentage가 metrics로 바뀌었다는 뜬금 없는 소리를 한다. 무슨 이야긴가 싶었는데, HPA API 버전이 변경된 것을 말하는 것이었다. [API 참고 문서 워크로드 리소스 쪽에 가면 HorizontalPodAutoscaler는 두개가 있다](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/)(v2beta2는 제외했다). 각각 autoscaling/v1, autoscaling/v2로 그룹이 다르다.

[autoscaling/v1](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v1/)에선 spec에 설정할 수 있는 지표로 `targetCPUUtilizationPercentage`, 평균 CPU 사용율, 만 쓸 수 있었다. [autoscale](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#autoscale) 서브 명령도 이에 맞는 옵션만 있는걸로 보인다.


[autoscaling/v2](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)는 더 많은 동작에 대한 정의(`behavior`)와 지표(`metrics`)를 설정할 수 있다([과거 제안서](https://github.com/kubernetes/design-proposals-archive/blob/main/autoscaling/hpa-v2.md)). 여기서 실습한 지표는 `Resource` 타입의 `cpu`이다:
```sh
$ k get hpa php-apache -oyaml | yq .spec.metrics[0]
resource:
  name: cpu
  target:
    averageUtilization: 50
    type: Utilization
type: Resource
```

Metrics server를 설치하여 얻을 수 있는 지표는 `Resource` 타입의 `cpu`와 `memory`가 전부이다. 다른 커스텀 타입을 정의하거나 확장하기 위해선 metrics server가 아닌 다른 컴포넌트가 필요하다.

## kube-prometheus-stack과 Prometheus adapter

문서에선 파드 타입 그리고 오브젝트 타입 지표를 HPA에 정의하는 예시를 보여주지만, 이 지표를 어떻게 생성할지에 대해선 나와 있지 않다.한가지 많이 사용하는 방법은 프로메테우스를 이용하는 것이다. 앱 코드에서 프로메테우스로 지표를 측정 수집하고 이를 [Prometheus adapter](https://github.com/kubernetes-sigs/prometheus-adapter)로 쿠버네티스 메트릭 API에 통합하면 HPA는 메트릭 API를 통해 커스텀 지표를 얻을 수 있다.

문서에 이러한 방법이 공식적이다라고 쓰여 있진 않지만, 쿠버네티스 커스텀 지표의 de facto standard 같아 보인다. 저장소 소속이 쿠버네티스 분과회 아래이고, PromQL이 프로메테우스가 아닌 다른 텔레메트리 생태계에서도 호환되기 때문에, 나중엔 쿠버네티스 안으로 들어올지도 모르겠단 생각이 든다. 하지만 예제 앱을 만들어 프로메테우스 메트릭을 측정하고 PromQL을 사용해 수집하는 것은, 러닝커브가 있어, 이번 포스팅에선 생략한다.

[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)이란 Helm 차트는 프로메테우스와 그라파나, alertmanager, exporter, KSM 등 프로메테우스 스택에 필요한 컴포넌트를 설치하고 쿠버네티스에서 운영하기 위한 오퍼레이터도 설치한다. prometheus-adapter도 같이 설치해준다:
```sh
$ k get all -n prometheus
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          5m47s
pod/prometheus-adapter-786d75df96-lcccl                      1/1     Running   0          4m30s
pod/prometheus-grafana-cdb9d9755-225kb                       3/3     Running   0          5m59s
pod/prometheus-kube-prometheus-operator-6589cd9b75-4jsts     1/1     Running   0          5m59s
pod/prometheus-kube-state-metrics-54c585df74-8n6l6           1/1     Running   0          5m59s
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          5m39s
pod/prometheus-prometheus-node-exporter-kgqhr                1/1     Running   0          5m58s
pod/prometheus-prometheus-node-exporter-knncz                1/1     Running   0          5m59s
pod/prometheus-prometheus-node-exporter-tgqjz                1/1     Running   0          5m59s
pod/prometheus-prometheus-node-exporter-vqt9m                1/1     Running   0          5m59s

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   5m48s
service/prometheus-adapter                        ClusterIP   10.105.238.0     <none>        443/TCP                      4m31s
service/prometheus-grafana                        ClusterIP   10.103.74.2      <none>        80/TCP                       5m59s
service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.109.182.18    <none>        9093/TCP                     5m59s
service/prometheus-kube-prometheus-operator       ClusterIP   10.110.217.60    <none>        443/TCP                      5m59s
service/prometheus-kube-prometheus-prometheus     NodePort    10.106.237.46    <none>        9090:30090/TCP               5m59s
service/prometheus-kube-state-metrics             ClusterIP   10.104.230.54    <none>        8080/TCP                     5m59s
service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     5m41s
service/prometheus-prometheus-node-exporter       ClusterIP   10.102.152.252   <none>        9100/TCP                     5m59s

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-prometheus-node-exporter   4         4         4       4            4           <none>          5m59s

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-adapter                    1/1     1            1           4m31s
deployment.apps/prometheus-grafana                    1/1     1            1           5m59s
deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           5m59s
deployment.apps/prometheus-kube-state-metrics         1/1     1            1           5m59s

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-adapter-786d75df96                    1         1         1       4m30s
replicaset.apps/prometheus-grafana-cdb9d9755                     1         1         1       5m59s
replicaset.apps/prometheus-kube-prometheus-operator-6589cd9b75   1         1         1       5m59s
replicaset.apps/prometheus-kube-state-metrics-54c585df74         1         1         1       5m59s

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     5m48s
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     5m41s
```


prometheus-adapter 설치 시엔 kube-prometheus-stack으로 설치한 프로메테우스 서비스가 노출되도록 `prometheus.service.url` values 값을 준다(`http://<prometheus-svc>.<ns>`).

따로 커스텀 지표 정의 없이 설치만 해도 엄청 많은 커스텀 지표가 수집된다:
```sh
$ k get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq '.resources | length'
4560

$ k get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq -r '.resources' | grep pods | tail -10
    "name": "pods/alertmanager_silences_query_errors",
    "name": "pods/prometheus_target_scrapes_exceeded_body_size_limit",
    "name": "pods/node_disk_reads_completed",
    "name": "pods/coredns_forward_request_duration_seconds_bucket",
    "name": "pods/prometheus_tsdb_compaction_chunk_range_seconds_sum",
    "name": "pods/prometheus_target_sync_length_seconds_count",
    "name": "pods/kube_poddisruptionbudget_created",
    "name": "pods/prometheus_target_scrape_pools_failed",
    "name": "pods/kube_pod_status_ready",
    "name": "pods/kube_pod_status_scheduled_time",
```

HPA에 사용할만한 지표가 있나 보려 했지만 실패했다. kube_ 로 시작하는 [KSM](https://github.com/kubernetes/kube-state-metrics)들도 보인다.


## 정리
- 쿠버네티스 오토스케일러는 크게 세가지이다.
  - 클러스터 오토스케일러
  - VPA
  - HPA
- Metrics server를 사용하기 위해 각 노드 쿠블릿과 API 서버가 TLS 연결이 되어야 한다.
- Metrics server에서 기본으로 제공되는 Resource 타입의 지표는 cpu, memory 두 가지이다.
- 커스텀 지표를 정의, 수집하고 Prometheus adapter에서 메트릭 API와 통합하여 HPA에서 사용할 수 있다.

---

## 참고
- https://kubernetes.io/ko/docs/home/
- https://medium.com/@tkdgy0801/eks-autoscaling-%ED%95%98%EA%B8%B0-part-1-horizontal-pod-autoscaler-with-custom-metrics-2274566463f9
