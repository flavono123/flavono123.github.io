---
title: "Minikube로 Docker Desktop를 대체해보며 쿠버네티스 통빡으로 맞춰보기"
date: 2022-01-04T00:51:44+09:00
tags:
- docker
- kubernetes
---

https://novemberde.github.io/post/2021/09/02/podman-minikube/

이 글을 읽고 회사 개발 계정 메일로 오던 도커에서 프로모 코드 뿌리는, 확인 안하던, 메일들이 생각났다.

역시 [도커 데스크탑이 유료로 바뀌니](https://www.docker.com/blog/the-grace-period-for-the-docker-subscription-service-agreement-ends-soon-heres-what-you-need-to-know/?utm_campaign=2021-12-30-discount-dt15-db15&utm_medium=email&utm_source=marketo&mkt_tok=NzkwLVNTQi0zNzUAAAGBqmcwviggye9AsyQZ2nTMX2z75F7q1eZqpnmVAzi54MOG2wwN-vvCUXAgP2hq-aP-Pp0a_-riX7owFkT1N_0SWA09Cge3BlRMk3qUiYQHHw) 주는 할인 코드였고 마침 쿠버네티스에 대한 관심도가 마구마구 오르는 시기라 호기롭게 위 포스팅을 따라하였다.

도커 데스크탑을 삭제하고 minikube 와 kompose 설치, 기존 docker-compose 파일을 변환해서 apply 까지 마쳤다. 마침 대상으로 해본 docker-compose 의 이미지들은 어차피 로컬(맥)에선 잘 안도는거라, 컨테이너가 잘 띄워졌는지 확인하진 않았다(솔직히 쿠버네티스도 컨테이너를 띄우는가? 도커를 대체한다니깐 그러지 않을까? 라고 확신 없는 생각하고 말았다). `kubectl get po -A` 를 단순히 `docker ps` 정도로만 생각하고 명령 결과를 잘 확인하지 않았다..

```sh
❯ kubectl get po -A
NAMESPACE     NAME                               READY   STATUS             RESTARTS      AGE
default       aaaa-798b5fdf96-bhz8n              0/1     ErrImagePull       0             89s
default       bbbb-58d984c8cd-kg742              0/1     ImagePullBackOff   0             89s
default       cccc-7c7f9d769-fnzr9               0/1     ErrImagePull       0             89s
default       dddd-8b5f8d488-whxbk               0/1     ImagePullBackOff   0             89s
default       eeee-fd4fdb5c6-nntg9               0/1     ImagePullBackOff   0             89s
default       ffff-5868c455f7-szbp7              0/1     ImagePullBackOff   0             89s
default       gggg-6bff9d66d6-c8mbd              0/1     ImagePullBackOff   0             88s
default       hhhh-c9b8fc96-r7hnh                0/1     ImagePullBackOff   0             88s
default       iiii-5859b7b695-hmxg6              0/1     ImagePullBackOff   0             88s
default       jjjj-7698798d95-k5ksf              0/1     ImagePullBackOff   0             88s
default       kkkk-648c7649c-nq8dz               0/1     ImagePullBackOff   0             88s
default       llll-94999f75-dnbqn                0/1     ImagePullBackOff   0             87s
default       mmmm-6f5576c8f4-thvpq              0/1     ErrImagePull       0             87s
default       nnnn-switch-8686568675-cw2xl       0/1     ImagePullBackOff   0             87s
kube-system   coredns-78fcd69978-ql5p2           1/1     Running            0             5d5h
kube-system   etcd-minikube                      1/1     Running            0             5d5h
kube-system   kube-apiserver-minikube            1/1     Running            0             5d5h
kube-system   kube-controller-manager-minikube   1/1     Running            0             5d5h
kube-system   kube-proxy-6l6hc                   1/1     Running            0             5d5h
kube-system   kube-scheduler-minikube            1/1     Running            0             5d5h
kube-system   storage-provisioner                1/1     Running            2 (15h ago)   5d5h
```

## ErrImagePull
그리고 다음날 로컬에서 직접 필요한 컨테이너를 올려 사용해보기로 했다. 회사에선 아직 컨테이너를 축소판 우분투 호스트정도로 쓰고 있어서 SSH 서버를 설치하고 접속해서 쓴다. 어제와 같이 해당하는 docker-compose 를 변환하고 apply 했으나 접속되지 않았다. 그제서야 STATUS 탭의 ErrImagePull 이 눈에 들어왔다.

별 삽질할것 없이, '아 우리 이미지 도커 허브에 프라이빗 레폰데 접근을 못하나보다' 라고 딱 생각이 들었다. 몇몇 컨테이너(정확힌 파드다)는 백오프로 빠져 대기하고 있다(ImagePullBackOff).

정확히 이걸 [해결하기 위한 문서](https://kubernetes.io/ko/docs/tasks/configure-pod-container/pull-image-private-registry/)가 있었다. ["커맨드 라인에서 자격 증명을 통하여 시크릿 생성하기"](https://kubernetes.io/ko/docs/tasks/configure-pod-container/pull-image-private-registry/#%EC%BB%A4%EB%A7%A8%EB%93%9C-%EB%9D%BC%EC%9D%B8%EC%97%90%EC%84%9C-%EC%9E%90%EA%B2%A9-%EC%A6%9D%EB%AA%85%EC%9D%84-%ED%86%B5%ED%95%98%EC%97%AC-%EC%8B%9C%ED%81%AC%EB%A6%BF-%EC%83%9D%EC%84%B1%ED%95%98%EA%B8%B0)을 따라 회사 도커 허브 계정 정보로 새 시크릿을 만들었다.

```sh
kubectl create secret docker-registry docker-login --docker-server=https://index.docker.io/v1/ --docker-username=회사도커허브계정 --docker-password=비밀번호 --docker-email=dev@cre.ma # 생성
kubectl get secret docker-login --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode | jq # 확인
```

그리고 yml 설정에 `imagePullSecrets` 를 추가한다. 여기서 소위 통빡을 발휘했다. CKA 강의를 구매해서 본격적으로 쿠버네티스를 배울 생각은 있었으나, 지금의 이 도커 컨테이너 띄우는걸 공부 후로 미룰 순 없었다. 변환 전 docker-compose 랑 변환 후 설정을 비교해보면 대충 감이 올거라 생각했다.

변경 전 docker-compose 는 비교적 간단하게 서비스(`services`) 하나를 띄우는 설정인데(파일은 회사거라 생략함),
이 부분이 변경 후 `Deployment` 부분이 된거 같다. 그리고 문서와 비교해보며 `spec.template.spec` 의 키로 추가해주었다. 이번엔 apply 후 **컨테이너 상태가 Running 이 됐다.**:

```sh
❯ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
dddd-768768d4b5-php5p     1/1     Running   0          22m
```

## 접속 실패

아마 컨테이너는 잘 올라간것으로 보이나 SSH 로 접속은 실패했다. 이 컨테이너는 SSH 접속을 하기 위해 브릿지 네트워크를 사용하고, 호스트의 20122 포트로 자신의 22 포트를 포워딩하여 SSH 접속을 하게 설정됐다:

```sh
Host 컨테이너호스트명
  ForwardAgent yes
  User ubuntu
  IdentityFile 키파일경로
  StrictHostKeyChecking no
  Hostname 127.0.0.1
  Port 20122
```

apply 시 쿠버네티스에서 파드 외에 서비스라는것도 만들던게 기억났다. 설정 파일 내용을 보니 게스트 22 -> 호스트 20122 로 포트포워딩한다는 내용 같았고 그럼 접속이 되어야 하는게 아닌가 의문이 들었다:

```yaml
...
  - apiVersion: v1
    kind: Service
    metadata:
    ...
    spec:
    ports:
      - name: "20122"
        port: 20122
        targetPort: 22
    ...
...
```

디버깅이나 로그를 보는것도 어려울거 같았다. 서비스란 키워드로 문서를 검색하다 ["서비스와 애플리케이션 연결하기"](https://kubernetes.io/ko/docs/concepts/services-networking/connect-applications-service/) 문서를 발견했다. 이와 같은 방법으로 생성한 파드의 IP 를 확인했고 서비스마다 IP 가 할당됨을 알았다.

SSH 설정의 IP로 바꿨는데 이번엔 명령이 hang 걸리고 Opertaion timed out 이 발생했다. 
'connect to pod over ssh' 같은 키워드로 구글링 하니 죄다 `ssh` 가 아닌 [exec bash 로 붙으면 된다는 글](https://stackoverflow.com/questions/45714658/need-to-do-ssh-to-kubernetes-pod)이 많았다. 이것에 대해선 지금 자세히 알아보기로 하진 않았다. 


## 결론

쿠버네티스에 대한 깊이 있는 이해 없이, 맥에서 도커 데스크탑을 대체하며 몇가지 허들을 넘었다. 또 마침 CKA 자격증 공부를 시작하며 미리 로컬에 명령을 간단하게 날려 볼 수 있는 환경 구성을 해두게 됐다. 이게 서로 상호작용하여, 초반 강의를 들으니 위 작업을 하며 몇가지 헤맸던 것들이 어렴풋이 이해 가기도 한다. 정확한 이해 없이 통빡으로 넘겨 찍은 부분은 확실히 이해해야겠다. 하지만 docker(compose) 설정에서 쿠버네티스 설정으로 변환도 잘 호환 되는거 같고, 도커를 주 컨테이너 엔진으로 쓰는 쿠버네티스의 디자인이 깔끔한거 같아 벌써 기대가 된다.

### 몇 가지 팁과 글에서 빠진 삽질
- 필요 없는, ErrImagePull/ImagePullBackOff 인 컨테이너를 삭제하기 위해 `kubectl delete pods --all` 실행
- 하지만 다시 만들어지는데 `kubectl delete deployments --all` 로 삭제
- 이 과정에서 실수로 `kubectl delete nodes --all` 도 하여 kube-system 네임스페이스의 마스터 노드가 삭제됐다 😱
- `minikube start` 로 다시 띄워주면 된다 🥰
