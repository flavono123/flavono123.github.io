---
title: "nc(Netcat)으로 NetworkPolicy 디버깅"
date: 2022-03-03T20:30:09+09:00
tags:
- nc
- kubernetes
---

CKAD 문제를 풀던 중 네트워크 정책을 설정하고 검증해야하는 문제가 있었는데 telnet 명령이 들질 않았다:
```sh
root@controlplane:~# k exec <from-pod> -- telnet <svc> 80
OCI runtime exec failed: exec failed: container_linux.go:367: starting container process caused: exec: "telnet": executable file not found in $PATH: unknown
command terminated with exit code 126
```

답지에선 nc(Netcat)을 사용해서 검증했다. nc라는 명령을 처음 알게 됐다. 문제를 재구성 해봤고 그 과정에서 알게 된 사실을 공유한다.

## nc
앞서 말한 [nc](https://linux.die.net/man/1/nc)는 telnet과 비슷하게 호스트의 포트가 열려 있는지 간단하게 확인 가능하다. 예시를 보면 범위를 지정하여 포트스캐닝도 가능하고, raw data를 그대로 전송하는게 가능해 보인다. 이에 반해 telnet은 문자 데이터를 주고 받기 위해 하는, Unix의 line feed를 [NVT](https://store.chipkin.com/articles/telnet-what-is-the-network-virtual-terminal)에 맞게 변환한다던가, 처리를 한다([참고](https://superuser.com/questions/1461609/what-is-the-difference-between-telnet-and-netcat)). 또 찾다보니 nc를 해킹에 많이 사용하는걸로 보인다. nc가 telnet보다 더 낮은 레벨의 제어가 가능한 툴로 이해했다.

내가 주목한것은 위처럼 telnet은 없지만 nc는 있는 이미지가 있다는 점이었다. 문제에서 사용한 이미지의 [Dockerfile](https://hub.docker.com/layers/kodekloud/webapp-color/latest/images/sha256-99c3821ea49b89c7a22d3eebab5c2e1ec651452e7675af243485034a72eb1423?context=explore)을 보니, 아마 nc는 기본 커널에 있는 도구일거 같다. telnet과 nc가 리눅스 배포판마다 각각 어떤 패키지에 속하는지, 그리고 어떤게 더 기본(?) 패키지 또는 유틸리티인지 자세히 찾아보진 못했지만, 위에서 설명한것처럼 nc가 더 기본적인 도구라는 점을 생각하면 telnet은 없더라도 nc는 있겠구나 정도로 생각했다. 실제로 [MacOS에선 telnet이 제외됐다는 글](https://www.unixfu.ch/use-netcat-instead-of-telnet/)도 있다(내 로컬은, High Sierra 이전부터 써서, 하위 호환성 때문인지 여전히 있다).

아무튼 nc를 단순히 해당 포트로 TCP 연결이 가능한지 정도로 쓸 것이다.

## Network Policy 디버깅
문제 상황은 이미 네임스페이스에 모든 ingress를 deny하는 정책이 있을 때, 서비스를 노출한 파드로 특정 파드에서 요청이 가능해야 한다.

먼저 netpol-test라는 네임스페이스를 만들고 진행했다. 서비스를 노출할 파드는 nginx를 띄워 80번 포트를 노출했다:
```sh
❯ k run -n netpol-test nginx --image=nginx --port 80
pod/nginx created

❯ k expose po nginx
service/nginx created

```

그리고 테스트할 파드로 스위스 군용 칼(san), busybox를 띄워줬다:
```sh
❯ k run -n netpol-test san --image=busybox -- sleep 5000
pod/san created

```

nc 명령 사용법은 다음과 같다:
```sh
❯ k exec -n netpol-test san -- nc -h
nc: invalid option -- h
BusyBox v1.34.1 (2022-02-03 01:05:27 UTC) multi-call binary.

Usage: nc [OPTIONS] HOST PORT  - connect
nc [OPTIONS] -l -p PORT [HOST] [PORT]  - listen

	-e PROG	Run PROG after connect (must be last)
	-l	Listen mode, for inbound connects
	-lk	With -e, provides persistent server
	-p PORT	Local port
	-s ADDR	Local address
	-w SEC	Timeout for connects and final net reads
	-i SEC	Delay interval for lines sent
	-n	Don't do DNS resolution
	-u	UDP mode
	-b	Allow broadcasts
	-v	Verbose
	-o FILE	Hex dump traffic
	-z	Zero-I/O mode (scanning)
command terminated with exit code 1
```

여기선 포트가 열려 있는지만 확인할거라 스캐닝옵션(-z)을 사용한다. 또 성공해도 응답이 없을거라 verbose(-v) 옵션을 사용한다. 또 -w 옵션으로 timeout을 준다.

문제 상황의 '네임스페이스에 모든 ingress를 deny하는 정책이 있을 때'는 [다음 네트워크 정책](https://kubernetes.io/ko/docs/concepts/services-networking/network-policies/#%EA%B8%B0%EB%B3%B8%EC%A0%81%EC%9C%BC%EB%A1%9C-%EB%AA%A8%EB%93%A0-%EC%9D%B8%EA%B7%B8%EB%A0%88%EC%8A%A4-%ED%8A%B8%EB%9E%98%ED%94%BD-%EA%B1%B0%EB%B6%80)이 생성된 상태이다:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

이 상태로 nginx 서비스 80 포트를 스캔하면 오류가 난다:
```sh
❯ k exec -n netpol-test san -- nc -zv -w 3 nginx 80
nc: nginx (10.107.204.80:80): Connection timed out
command terminated with exit code 1
```

san -> nginx로 80번 포트만 허용하는 인그레스 정책을 만들자:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-from-busybox
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      run: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: san
    ports:
    - protocol: TCP
      port: 80
```

포트 스캔이 성공할 것이다:
```sh
❯ k exec -n netpol-test san -- nc -zv -w 3 nginx 80
nginx (10.107.204.80:80) open

```

네트워크 정책은 **화이트리스트를 선언**하는 방법이다. 따라서 룰(spec.ingress 또는 egress)에 아무것도 적지 않는다면, 맨 처음 정책처럼 모든 트래픽을 막을 것이다(선언한 것만 허용하기 때문이다). 위 예제에서 [spec.ingress 키에 빈 배열을 주더라도 똑같이 모든 ingress 트래픽을 막는다](https://stackoverflow.com/questions/54827386/how-to-check-if-network-policy-have-been-applied-to-pod/71324868#71324868):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: []
```

네트워크 정책은 적용(configure) 순서와 무관하게 정책 룰을 병합해서 화이트리스트를 만든다.

## minikube에서 NetworkPolicy 사용하기

만약 minikube에서 위 실습을 따라 했으면 기대한대로 동작하지 않을 수 있다. [minikube는 기본적으로 NetworkPolicy를 지원하지 않는 CNI를 사용한다.](https://minikube.sigs.k8s.io/docs/handbook/network_policy/) 하지만 간단히 사용할 수 있도록, CNI를 Calico로 바꾸는 법을 설명하고 있다. 나도 이 방법을 따라 minikube를 간단히 재시작해서 테스트를 했다:
```sh
❯ minikube stop
...
❯ minikube start --network-plugin=cni --cni=calico
...
```

## 정리

마지막 섹션을 간단하게 적었지만 여기서 엄청 헤맸다. 네트워크 정책을 디버깅하는 실습을 만드는데 네트워크 정책 자체가 적용 안되고 있을 줄은 몰랐다. 강의 내용 중 어렴풋이 CNI와 관련 있다 라는걸 기억해 관련 키워드로 찾아나갔다. minikube는 kubelet도 아예 안띄우는것 같다 🤔. 미뤄두었던 [KTHW](https://github.com/kelseyhightower/kubernetes-the-hard-way)를 얼른 구성해봐야겠다...

네트워크 정책은 적용하고 디버깅이 어려운 부분 중 하나인것 같다. 여기선 문제를 재구성했지만, 실제 환경에서 디버깅에도 문제가 없을거 같다.

현재 latest인, 1.34.1 버전의 busybox엔 telnet도 있었다. 하지만 앞으론 헤매지 않게(?) nc를 사용하려고 한다.

--- 

## 참고
- https://linux.die.net/man/1/nc
- https://superuser.com/questions/1461609/what-is-the-difference-between-telnet-and-netcat
- https://kubernetes.io/ko/docs/concepts/services-networking/network-policies/
- https://stackoverflow.com/questions/54827386/how-to-check-if-network-policy-have-been-applied-to-pod/71324868#71324868
- https://minikube.sigs.k8s.io/docs/handbook/network_policy/
