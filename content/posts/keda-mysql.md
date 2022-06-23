---
title: "KEDA(MySQL Scaler)"
date: 2022-06-23T12:34:47+09:00
tags:
- kubernetes
- keda
---

[HPA 포스트](/posts/hpa)에서 한가지 써보지 방법이 있다. 기본 지표(`metrics.k8s.io`)나 커스텀 지표(`custom.metrics.k8s.io`)가 아닌 **외부 지표(`external.metrics.k8s.io`)**를 사용한 확장이다. 사실 커스텀 지표는 HPA 확장까지 실습하진 못하고, 프로메테우스 스택을 설치하여 커스텀 지표를 확인만 했다.

하지만 외부 지표에 대해선 [KEDA(Kubernetes Event-driven Autoscaling)](https://keda.sh/)이란, 비교적 쉽게 따라해볼 수 있는 서드파티가 있다. KEDA는 지표 대신 이벤트 기반의 오토스케일을 한다고 설명한다. 뒤에 MySQL 실습을 해보면 쿼리 결과를 지표로 하는 HPA이다. 이것이 지표 설정, 수집 같은 과정을 애플리케이션(도메인?) 특정하여 추상화하기 때문에 좀 더 해당 애플리케이션 전문가가 쉽게 스케일 지표를 설정할 수 있다고 느껴진다.

HPA의 대상이 될만한 여러 애플리케이션의 [Scaler](https://keda.sh/docs/2.7/scalers/)를 제공한다. 그 중 [MySQL Scaler](https://keda.sh/docs/2.7/scalers/mysql/)를 실습해본다.


## MySQL Operator

지난 [Sysbench로 MySQL Operator 벤치마크](/posts/sysbench-mysql-operator)에서 사용한 [MySQL Operator](https://dev.mysql.com/doc/mysql-operator/en/)와 [InnodbCluster](https://dev.mysql.com/doc/mysql-operator/en/mysql-operator-innodbcluster.html)을 그대로 사용한다. 설명은 지난 포스트 링크로 대체하고, 명령어 셋만 적는다:
```sh
$ helm repo add mysql-operator https://mysql.github.io/mysql-operator/
$ helm install mysql-operator mysql-operator/mysql-operator \
  -n mysql-operator --create-namespace --version 2.0.4
$ helm install mycluster mysql-operator/mysql-innodbcluster \
  --set credentials.root.password='sakila' \
  --set tls.useSelfSigned=true \
  -n mysql-cluster --create-namespace --version 2.0.4

$ k exec -it -n mysql-operator deploy/mysql-operator -- \
  mysqlsh mysqlx://root@mycluster.mysql-cluster --password=sakila --sqlx \
  -e "CREATE DATABASE IF NOT EXISTS sysbench;
      CREATE USER IF NOT EXISTS 'sysbench'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'sysbench';
      GRANT ALL ON sysbench.* to 'sysbench'@'%';
      FLUSH PRIVILEGES;"
```

## Sysbench 템플릿

이번에도 Sysbench를 사용할 것이다. MySQL 인스턴스 스케일에 테이블 행 수를 이벤트로 사용할 것이기 때문에, 행을 만드는 도구로 적합하다고 생각했다(방법은 뒤에서 설명한다). 다만 저번엔 읽기로 성능테스만 했기 때문에 다른 테스트를 사용하기 위해 확장 가능한 템플릿으로 만들었다:
```yaml
# sysbench-oltp-template.yaml

apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: ${TEST_NAME}
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      initContainers:
      - args:
        - --db-driver=mysql
        - --mysql-host=mycluster.mysql-cluster
        - --mysql-port=3306
        - --mysql-user=sysbench
        - --mysql-password=sysbench
        - --mysql-db=sysbench
        - --table-size=10000
        - --threads=4
        - --tables=1
        - ${TEST_FILE_PATH}
        - prepare
        image: vonogoru123/sysbench
        name: sysbench-prepare
      - args:
        - --db-driver=mysql
        - --mysql-host=mycluster.mysql-cluster
        - --mysql-port=3306
        - --mysql-user=sysbench
        - --mysql-password=sysbench
        - --mysql-db=sysbench
        - --report-interval=10
        - --table-size=10000
        - --threads=4
        - --tables=1
        - --time=180
        - ${TEST_FILE_PATH}
        - run
        image: vonogoru123/sysbench
        name: sysbench-run
      containers:
      - args:
        - --db-driver=mysql
        - --mysql-host=mycluster.mysql-cluster
        - --mysql-port=3306
        - --mysql-user=sysbench
        - --mysql-password=sysbench
        - --mysql-db=sysbench
        - --table-size=10000
        - --threads=4
        - --tables=1
        - ${TEST_FILE_PATH}
        - cleanup
        image: vonogoru123/sysbench
        name: sysbench-cleanup
      nodeSelector:
        kubernetes.io/hostname: cluster1-master1
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      restartPolicy: Never
status: {}
```

`TEST_FILE_PATH`와 `TEST_NAME`이란 변수를 [envsubst](https://linux.die.net/man/1/envsubst)를 사용해 템플릿 할 것이다(DOIK 스터디에서 배운 방법이다). 저번과 달라진 점은 테이블은 한개로 그리고 초기 테이블 사이즈를 1만개로 줄였다. 인자로 바꾸길 원하는 부분을 `TEST_` 변수와 비슷하게 바꾼다면 확장 가능할 것이다.

## KEDA

이번에도 설치를 우선하고 구조를 살펴본다:
```sh
# https://keda.sh/docs/2.7/deploy/
$ helm repo add kedacore https://kedacore.github.io/charts
$ helm repo update
$ helm install keda kedacore/keda -n keda --create-namespace

$ k get all -n keda
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/keda-operator-688c7b949-vf9jk                      1/1     Running   0          16h
pod/keda-operator-metrics-apiserver-7b6549ff6c-rwjmk   1/1     Running   0          16h

NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/keda-operator-metrics-apiserver   ClusterIP   10.102.212.185   <none>        443/TCP,80/TCP   16h

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/keda-operator                     1/1     1            1           16h
deployment.apps/keda-operator-metrics-apiserver   1/1     1            1           16h

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/keda-operator-688c7b949                      1         1         1       16h
replicaset.apps/keda-operator-metrics-apiserver-7b6549ff6c   1         1         1       16h
```

두 개의 디플로이먼트가 전부이다. 각각 [개념 문서](https://keda.sh/docs/2.7/concepts/#how-keda-works)의 Agent와 Metrics이자 [구조 문서](https://keda.sh/docs/2.7/concepts/#architecture)의 Controller와 Metrics adapter이다. 오퍼레이터 패턴으로 HPA를 운영한다. Metrics apiserver는 외부 지표 API를 받아 쿠버네티스 metrics apiserver와 통합하기 위한 컴포넌트이다. 다양한 Scaler는 컨트롤러(오퍼레이터)에 내장되어 있는듯하며, 이어서 ScaledObject를 정의하여 원하는 MySQL HPA를 만들어 본다.


```sh
$ k get crd | grep keda
clustertriggerauthentications.keda.sh                 2022-06-22T11:16:40Z
scaledjobs.keda.sh                                    2022-06-22T11:16:40Z
scaledobjects.keda.sh                                 2022-06-22T11:16:40Z
triggerauthentications.keda.sh                        2022-06-22T11:16:40Z
```

[CRD](https://keda.sh/docs/2.7/concepts/#custom-resources-crd) 4개는 크게 두 종류이다. 하나는 앞서 설명한, Scaler의 대상 오브젝트로, 이벤트 소스와 오토스케일 대상 리소스를 정의하는 ScaledObject와 대상 리소스가 [쿠버네티스 Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)으로 한정되는 ScaledJob이다. 다른 하나는 KEDA Scaler가 이벤트를 수집, 모니터하기 위해 필요한 인증 정보를 적는 TriggerAuthentication, ClusterTriggerAuthentication이다. 두 리소스의 차이는 namespaced 여부이다.

## Secret, TriggerAuthentication

[MySQL Scaler 예시 매니페스트](https://keda.sh/docs/2.7/scalers/mysql/#example)와 비슷한 매니페스트 YAML 파일을 오브젝트 각각 만들어본다. 먼저 Secret이다:
```sh
$ k -n mysql-cluster create secret generic keda-mysql \
  --from-literal username=root \
  --from-literal pasword=sakila \
  --from-literal host=mycluster.mysql-cluster \
  --from-literal port=3306 \
  --from-literal db=sysbench \
  --dry-run=client -oyaml
apiVersion: v1
data:
  db: c3lzYmVuY2g=
  host: bXljbHVzdGVyLm15c3FsLWNsdXN0ZXI=
  pasword: c2FraWxh
  port: MzMwNg==
  username: cm9vdA==
kind: Secret
metadata:
  creationTimestamp: null
  name: keda-mysql
  namespace: mysql-cluster
```

KEDA의 MySQL Scaler가 이벤트를 요청, 즉 쿼리를 해보기 위해 필요한 접속 정보이다. [KEDA는 이런 credentials를 재사용 가능하다](https://keda.sh/docs/2.7/concepts/authentication/#re-use-credentials-and-delegate-auth-with-triggerauthentication)고 하는데, 한 눈에 확인하기 위해 새로 만들었다. 또 mysql-innodbcluster 차트를 생성하면 자동으로 생성되는 cluster-secret엔 host나 db가 정의되어 있지 않다. username과 password는 그 secret의 것을 재사용해도 무방하다. 다음은 TriggerAuthentication이다:
```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-mysql-secret
spec:
  secretTargetRef:
  - parameter: dbName
    name: keda-mysql
    key: db
  - parameter: host
    name: keda-mysql
    key: host
  - parameter: password
    name: keda-mysql
    key: password
  - parameter: port
    name: keda-mysql
    key: port
  - parameter: username
    name: keda-mysql
    key: username
```

위에서 정의한 Secret의 데이터와 [Scaler가 사용할 Authentication Parameters](https://keda.sh/docs/2.7/scalers/mysql/#authentication-parameters)를 매핑한다. 데이터 소스가 Secret이 아닌 [환경변수](https://keda.sh/docs/2.7/concepts/authentication/#environment-variables) 또는 [외부 서드파티](https://keda.sh/docs/2.7/concepts/authentication/#secrets)에서 가져오는 방법도 있다.

위 여러 인증 파라미터 대신 하나로 해결할 수 있는 connectionString을 시도해봤지만 실패했다. 일반적인 MySQL connection string도 아닌 것 같고.. 문서 설명이 충분치 않은데, 다음 몇가지를 시도했지만 전부 실패했다:
- root:sakila@tcp(mycluster.mysql-cluster:3306)/sysbench
- root:sakila@mycluster.mysql-cluster:3306/sysbench
- root:sakila@mycluster.mysql-cluster/sysbench:3306

## ScaledObject
마지막으로 정의할 매니페스트는 ScaledObject이다:
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mysql-scaledobject
  namespace: mysql-cluster
spec:
  pollingInterval: 10
  cooldownPeriod: 10
  minReplicaCount: 3
  maxReplicaCount: 6
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefuleSet
    name: mycluster
    envSourceContainerName: mysql
  triggers:
  - type: mysql
    metadata:
      queryValue: "10000"
      query: SELECT COUNT(*) FROM sbtest1
    authenticationRef:
      name: keda-trigger-auth-mysql-secret
  advanced:
    restoreToOriginalReplicaCount: true
```

`.spec.triggers[0].authenticationRef`를 보면 위에 정의한 TriggerAuthentication를 참조한다. MySQL ScaledObject의 [트리거 스펙](https://keda.sh/docs/2.7/scalers/mysql/#trigger-specification)은 `query`에 MySQL 쿼리를 평문으로 작성하여 이 결과 값이 HPA의 지표로 사용한다. `queryValue`는 HPA의 목표값을 적어준다. 여기선 테이블 행 만개 단위로 mysqld 서버 파드가 있길 기대하며 작성했다.

`.spec`의 나머지 키들의 정의는 [이 문서](https://keda.sh/docs/2.7/concepts/scaling-deployments/#scaledobject-spec) 참조하자. 여기선 [예제](https://keda.sh/docs/2.7/scalers/mysql/#example)와 달리 `scaleTargetRef`를 더 자세히 써주었다. 우선 기본 리소스가 디플로이먼트이기 때문이다. 또 mysql-innodbcluster가 만드는 파드는 sidecar 컨테이너가 하나 더 있기 때문에 `envSourceContainerName`로 파드 내 대상 컨테이너를 특정해준다.

또 하나 눈에 띄는 설정은 `advanced.restoreToOriginalReplicaCount`인데 [설정해주면 나중에 지표가 낮아진다면 오토스케일 이전의 파드 개수로 스케일 다운(인)한다](https://keda.sh/docs/2.7/concepts/scaling-deployments/#advanced). 그 외에 [HPA v2의 명세의 behavior](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)도 직접 설정할 수 있다.

위 매니페스트를 전부 생성하자. 성공했다면 다음 같은 이벤트를 확인할 수 있다:

```sh
$ k describe scaledobjects.keda.sh -n mysql-cluster mysql-scaledobject
...(생략)
Events:
  Type     Reason              Age                 From           Message
  ----     ------              ----                ----           -------
  Normal   KEDAScalersStarted  111s                keda-operator  Started scalers watch
  Normal   ScaledObjectReady   96s (x2 over 111s)  keda-operator  ScaledObject is ready for scaling
  Warning  KEDAScalerFailed    1s (x12 over 111s)  keda-operator  Error 1146: Table 'sysbench.sbtest1' doesn't exist
```

'ScaledObject is ready for scaling'로 잘 배포됐음을 확인한다. Warning(그리고 메세지엔 에러...!)가 좀 의아할 수 있는데, 아직 sysbench prepare로 대상 테이블을 만들지 않았기 때문에 그렇다. HPA를 살펴보면 만들어졌지만 지표를 수집 못하고 있는 것을 볼 수 있다:
```sh
$ k get hpa -n mysql-cluster
NAME                          REFERENCE               TARGETS               MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-mysql-scaledobject   StatefulSet/mycluster   <unknown>/10k (avg)   3         5         3          4m21s
```

## 스케일 아웃
Sysbench로 테이블 행을 늘려 KEDA에 의해 스케일 아웃되게 해보자. 앞서 만든 sysbench 템플릿에 oltp-insert.lua 테스트를 만들자:
```sh
$ TEST_FILE_PATH=/usr/local/share/sysbench/oltp_insert.lua \
  TEST_NAME=$(echo "${TEST_FILE_PATH}" | cut -d/ -f6 | tr [:punct:] -) envsubst < sysbench-oltp-template.yaml \
    > sysbench-oltp-insert.yaml
```


테스트 실행 전 mysqld 파드 개수와 테이블 행 개수를 모니터할 수 있게 일반 터미널과 MySQL 클라이언트를 하나씩 준비하자:
```sh
# 모니터 1: 파드 개수
$ watch 'kubectl get po -owide -n mysql-cluster'
Every 2.0s: kubectl get po -owide -n mysql-cluster                                                                 cluster1-master1: Thu Jun 23 09:37:14 2022

NAME                                READY   STATUS    RESTARTS      AGE    IP               NODE               NOMINATED NODE   READINESS GATES
mycluster-0                         2/2     Running   2 (65m ago)   7h9m   172.16.25.208    cluster1-worker2   <none>           2/2
mycluster-1                         2/2     Running   3 (66m ago)   7h9m   172.16.40.78     cluster1-worker1   <none>           2/2
mycluster-2                         2/2     Running   2 (65m ago)   7h9m   172.16.101.209   cluster1-worker3   <none>           2/2
mycluster-router-5f9758dc8f-wcbjt   1/1     Running   3 (65m ago)   7h8m   172.16.25.207    cluster1-worker2   <none>           <none>

```


```sh
# 모니터 2: MySQL Client
$ k exec -it -n mysql-operator deploy/mysql-operator -- \
  mysqlsh mysqlx://root@mycluster.mysql-cluster --password=sakila --sqlx
...(생략)
 MySQL  mycluster.mysql-cluster:33060+ ssl  SQL > use sysbench;
Default schema set to `sysbench`.
Fetching table and column names from `sysbench` for auto-completion... Press ^C to stop.
```


이제 테스트를 시작하고 테스트 파드의 로그를 보자:
```sh
$ k apply -f sysbench-oltp-insert.yaml
job.batch/oltp-insert-lua created
$ k logs -f job/oltp-insert-lua sysbench-run
...(생략)
```


내 PC에선 테스트 동안 행이 6만여개 정도 생긴다. 이 때 파드는 5개까지 증가하는데, 목표 지표 값보다 높지만(60000/5 > 10000) maxReplicaCount에 의해 더 이상 증가하지 않는다:
```sh
# 모니터 1
Every 2.0s: kubectl get po -owide -n mysql-cluster                                                                 cluster1-master1: Thu Jun 23 09:45:43 2022

NAME                                READY   STATUS    RESTARTS        AGE     IP               NODE               NOMINATED NODE   READINESS GATES
mycluster-0                         2/2     Running   2 (74m ago)     7h17m   172.16.25.208    cluster1-worker2   <none>           2/2
mycluster-1                         2/2     Running   3 (74m ago)     7h17m   172.16.40.78     cluster1-worker1   <none>           2/2
mycluster-2                         2/2     Running   2 (73m ago)     7h17m   172.16.101.209   cluster1-worker3   <none>           2/2
mycluster-3                         2/2     Running   1 (3m39s ago)   4m21s   172.16.25.209    cluster1-worker2   <none>           2/2
mycluster-4                         2/2     Running   1 (2m46s ago)   3m51s   172.16.40.81     cluster1-worker1   <none>           2/2
mycluster-router-5f9758dc8f-wcbjt   1/1     Running   3 (73m ago)     7h16m   172.16.25.207    cluster1-worker2   <none>           <none>

# 모니터 2
 MySQL  mycluster.mysql-cluster:33060+ ssl  sysbench  SQL > SELECT COUNT(*) FROM sbtest1;
+----------+
| COUNT(*) |
+----------+
|    59705 |
+----------+
1 row in set (0.0186 sec)
```

## 스케일 인
[기본 스케일 다운(인) threshold인 5분(behavior.scaleDown default)](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/)을 기다려도 파드가 줄지 않을 것이다. 테스트에서 테스트 후 sbtest1를 지워버려 HPA가 지표를 수집하지 못하기 때문이다. 가장 마지막 수집한 파드 당 테이블 행을 기록하고 있을 것이다:
```sh
$ k get hpa -n mysql-cluster
NAME                          REFERENCE               TARGETS               MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-mysql-scaledobject   StatefulSet/mycluster   11567600m/10k (avg)   3         5         5          19m
```

스케일 인 동작을 보기 위해 빈 sbtest1 테이블을 만들어 주자:
```sh
# 모니터 2
 MySQL  mycluster.mysql-cluster:33060+ ssl  sysbench  SQL > CREATE TABLE sbtest1 (id int);
Query OK, 0 rows affected (0.0303 sec)
```

mycluster-4와 mycluster-5가 종료되고 minReplicaCount 개수만큼 파드가 남아 있게 된다:
```sh
# 모니터 1
Every 2.0s: kubectl get po -owide -n mysql-cluster                                   cluster1-master1: Thu Jun 23 09:51:36 2022

NAME                                READY   STATUS    RESTARTS      AGE     IP               NODE               NOMINATED NODE
  READINESS GATES
mycluster-0                         2/2     Running   2 (80m ago)   7h23m   172.16.25.208    cluster1-worker2   <none>
  2/2
mycluster-1                         2/2     Running   3 (80m ago)   7h23m   172.16.40.78     cluster1-worker1   <none>
  2/2
mycluster-2                         2/2     Running   2 (79m ago)   7h23m   172.16.101.209   cluster1-worker3   <none>
  2/2
mycluster-router-5f9758dc8f-wcbjt   1/1     Running   3 (79m ago)   7h22m   172.16.25.207    cluster1-worker2   <none>
  <none>
```

실습이 끝났으니 테스트 Job을 삭제한다:
```sh
$ k delete job oltp-insert-lua
job.batch "oltp-insert-lua" deleted
```

## 문제점
이 실습은 KEDA는 잘 동작했지만, 다른 이유 때문에 어려움이 많았다.

### 1. Mysql operator

첫번째론 mysql-operator 동작 문제이다. 이 실습은 순서대로 진행해야 한다. 만약 KEDA를 먼저 설치하고 mysql-innodbcluster를 턴업하면 아무 동작을 안할 수도 있다. 이유로 추정하는 것은 mysql-operator를 구현한 kopf에서 나오는 오류이다:
```sh
k -n mysql-operator logs mysql-operator-659ff68ccf-b775g mysql-operator  | grep -i too
[2022-06-23 08:31:17,425] kopf.objects         [ERROR   ] Throttling for 1 seconds due to an unexpected error: APIError('Secret "sh.helm.release.v1.keda.v1" is invalid: metadata.annotations: Too long: must have at most 262144 bytes', {'kind': 'Status', 'apiVersion': 'v1', 'metadata': {}, 'status': 'Failure', 'message': 'Secret "sh.helm.release.v1.keda.v1" is invalid: metadata.annotations: Too long: must have at most 262144 bytes', 'reason': 'Invalid', 'details': {'name': 'sh.helm.release.v1.keda.v1', 'kind': 'Secret', 'causes': [{'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}, {'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}, {'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}, {'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}]}, 'code': 422})
kopf.clients.errors.APIError: ('Secret "sh.helm.release.v1.keda.v1" is invalid: metadata.annotations: Too long: must have at most 262144 bytes', {'kind': 'Status', 'apiVersion': 'v1', 'metadata': {}, 'status': 'Failure', 'message': 'Secret "sh.helm.release.v1.keda.v1" is invalid: metadata.annotations: Too long: must have at most 262144 bytes', 'reason': 'Invalid', 'details': {'name': 'sh.helm.release.v1.keda.v1', 'kind': 'Secret', 'causes': [{'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}, {'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}, {'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}, {'reason': 'FieldValueTooLong', 'message': 'Too long: must have at most 262144 bytes', 'field': 'metadata.annotations'}]}, 'code': 422})
```

실제로 KEDA 설치 후 생기는 secret(`sh.helm.release.v1.keda.v1`)를 보면 데이터가 엄청 길긴하다. mysql-operator를 먼저 설치하고 KEDA를 설치하더라도 저 오류는 뜨긴한다. 그래서 추정만 할 뿐 정확한 원인은 알 수 없다.

그럴 때의 InnoDB Cluster 상태는 정말 난장판이다. 오퍼레이터가 일을 못해서 InnoDBCluster 객체의 상태와 파드 개수가 불일치하기도 하고, 파드를 포함해 리소스 종료가 잘 안된다. 이 때 한가지 알아낸 방법은 리소스 명세의 [finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)를 삭제하면 강제 종료 가능하다([참고](https://containersolutions.github.io/runbooks/posts/kubernetes/pod-stuck-in-terminating-state/#solution-a)). 정확히 말하면 finalizers가 종료를 막고 있는 것이다.


### 2. 노드 장애

두번째 문제는 노드 메모리가 부족한 문제였다. 내 맥북 메모리는 16GB라서 노드 하나 당 kubeadm 최소 사항인 2GB 밖에 주지 못했다. 이런 리소스에 DB 인스턴스를 여러 개 올리는게 이상하긴 하다. 그래서 3GB까지 늘려줬는데 의외로 아무 일도 없었다...

그래도 문제는 발생하기 일쑤였다. ScaledObject의 maxReplicaCount를 더 크게 설정했을 때 한 노드에 DB 인스턴스 파드가 몰리면, 해당 노드는 kubectl이나 SSH 접속 안되는 장애가 발생했다.

첫번째 이슈와 겹쳐서 MySQL operator는 별로다... 하고 눈을 돌린 것이 [Percona Operator for MySQL based on Percona XtraDB Cluster(이하 pxc operator)](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html#pxcoperator)이다. 스터디에서 권장하기도 했고 써보니 이 문제에 대해서도 해결책을 제공하고 있었다.

그것은 다름 아닌 pxc operator는 **DB 인스턴스 파드에 리소스를 명세했다는 점**이다. 노드 당 메모리를 2GB로 하고 pxc operator와 [pxc-db(innodbcluster에 해당한다)](https://artifacthub.io/packages/helm/percona/pxc-db)를 배포했을 때 최소 레플리카인 3개도 생성하지 못했는데, 파드에 리소스 요청이 있으니 적당한, 메모리 여유 있는 노드가 없어, 파드를 실행하지 않고 대기하게 된다(percona/pxc-db도 values가 복잡한데 [yq를 활용해 찾을 수 있었다](/posts/yq-parse-long-nested-helm-values)):
```sh
$  helm show values percona/pxc-db | yq .pxc.resources
requests:
  memory: 1G
  cpu: 600m
limits: {}
# memory: 1G
# cpu: 600m
```

아주 간단한, 쿠버네티스가 기본적으로 제공하는 기능이지만, 무분별한 스케일 아웃으로 노드 장애가 되는 것을 막을 수 있다. [파드 리소스는 실행 중에 변경할 수 없는데](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#resources) mysql-innodbcluster 차트 values로도 제공하지 않아서 이를 적용할 수 없었다.


그럼 왜 pxc로 실습을 진행하지 않았느냐? 마침 내가 실습할 때 1.11.0 버전 업이 되더니 이상한 버그가 생겨 DB 레플리카 인스턴스를 늘리지 못하고 있었다. 이전 버전으로 해도 마찬가지이고... [이 해결 PR](https://github.com/percona/percona-helm-charts/pull/146)과 관련 이슈로 보이는데 빨리 해결했으면 싶다... 아무튼 그런 이유로 실습에선 maxReplicaCount도 보수적으로 하고, 파드 출력 시 `-owide` 옵션으로 어느 노드에 배치됐는지도 눈으로 확인하고.. `k top no`를 진짜 많이 찍어 봤다.

mysql-operator도 파드 리소스 제한을 걸길 바란다.

### 3. 수동으로 모든 자원 회수

세번째는 정확한 문제라기 보단 첫번째, 두번째 문제의 연장이며 이와 관련된 해결법(troubleshooting)이다. 위처럼 오퍼레이터나 오작동이나 노드 장애 때문에 CR의 상태가 꼬이면, Helm 차트나 오퍼레이터의 CRD로 생성하는 모든 자원을 직접 잘 삭제해야 한다.

특히 CR(e.g. InnoDBCluster)은 `kubectl get all` 같은 명령엔 걸리지 않기 때문에 놓치기 쉽다. 스터디에서 kubectl의 기능을 올려주는 몇몇 플러그인을 추천 받았는데, CR도 한 눈에 볼 수 있는지 확인해봐야겠다.

또 첫번째에서 언급한 finalizers를 삭제하는 것은 파드 뿐만 아니라 모든 리소스가 삭제 명령에 대해 hang일 때 확인하고 시도해보면 좋다. 물론 오퍼레이터가 기능할 것이라 예상되면 .. 조금은 기다려 보자.


## 정리
- KEDA는 외부 지표(external.metrics.k8s.io)를 정의하여 HPA를 확장한다.
- ScaledObjects에, 외부 지표로, 애플리케이션 특정 목표 지표를 설정하여 모니터링 및 지표 수집 관련한 부분을 크게 추상화 할 수 있다.
- (Cluster)TriggerAuthentication은 쿠버네티스의 Secret이나 환경변수 또는 서드파티의 credentials을 재활용해  ScaledObjects 객체와 매핑해 외부 지표(이벤트) 모니터 및 수집을 가능하게 한다.
- 여러 애플리케이션에 대한 Scaler를 제공한다.
- 아직 커스텀 지표와 외부 지표 사용한 HPA 확장의 차이점은 잘 모르겠다.

---

## 참고
- https://keda.sh/docs/2.7/
- https://youtu.be/H5eZEq_wqSE
- https://kubernetes.io/docs/home/
- https://github.com/nyoung08/nyoung08.github.io/blob/master/_posts/2022-06-12-mysql-operator%EC%99%80-keda-%EC%B2%B4%ED%97%98%EA%B8%B0.md
- https://containersolutions.github.io/runbooks/posts/kubernetes/pod-stuck-in-terminating-state/
- https://gasidaseo.notion.site/e49b329c833143d4a3b9715d75b5078d
