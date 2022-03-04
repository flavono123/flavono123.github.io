---
title: "NetworkPolicy 설정 시 주의점"
date: 2022-03-04T20:57:52+09:00
tags:
- kubernetes
---

CKAD killer.sh 시뮬레이터를 풀다가, 배점이 가장 높은 문제에서 계속 틀렸다. 이번에도 네트워크 정책 문제였다. 이번에도 몇가지 실험해서 알게 된 점을 정리한다.

문제에서 egress 설정을 잘한거 같은데 안됐다. 예를 들면 1234로 서비스 노출한 특정 파드(nginx)에 특정 파드만(san) 접근할 수 있게 하는 것이다:

```sh
❯ k -n netpol-test run  nginx --image=nginx --port 80
pod/nginx created

❯ k -n netpol-test expose po nginx --port=1234 --target-port=80
service/nginx exposed

```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: san-to-nginx
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      run: san
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: nginx
    ports:
    - protocol: TCP
      port: 1234

```

```sh
❯ k -n netpol-test run san --rm -i --image=busybox --restart=Never -- nc -zv -w 3 nginx 1234                            
If you don't see a command prompt, try pressing enter.                                               
nc: bad address 'nginx'
pod "san" deleted
pod netpol-test/san terminated (Error)
```

## DNS
실패한 이유가 'bad address'이다. nginx 호스트로 노출한 서비스를 찾을 수 없다. 만약 파드에 직접 요청한다면 이유가 다르다:
```sh
❯ k get pod nginx -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          26h   10.244.120.68   minikube   <none>           <none>

❯ k run san --rm -i --image=busybox --restart=Never -- nc -zv -w 3 10.244.120.68 1234                    
If you don't see a command prompt, try pressing enter.                                                   
nc: 10.244.120.68 (10.244.120.68:1234): Connection timed out                                             
pod "san" deleted                                                                                        
pod netpol-test/san terminated (Error)

```

[앞서 했던 ingress와 예제](/posts/debug-netpol-with-nc)와 달리 egress는 DNS resolve를 위한 정책도 필요하다:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: san-to-nginx
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      run: san
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: nginx
    ports:
    - protocol: TCP
      port: 1234
  - ports:
    - protocol: UDP
      port: 53

```

UDP/53에 대한 egress는 분리된 규칙으로 만들었다. DNS resolve는 run=nginx 인 파드로 뿐만 아니라, 일반적으로 모두 필요하기 때문이다. 또 [spec.egress.to를 아예 안적거나 빈 배열을 주는 것은, 모든 목적지에 매치하는, 같은 동작을 한다(반대로 spec.ingress.from도 출발지에 대해 그러하다)](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/network-policy-v1/):

> - egress.to ([]NetworkPolicyPeer) ... f this field is empty or missing, this rule matches all destinations (traffic not restricted by destination) ...

따라서 위 정의는 다음과도 같다:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: san-to-nginx
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      run: san
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: nginx
    ports:
    - protocol: TCP
      port: 1234
  - to: []
    ports:
    - protocol: UDP
      port: 53

```

DNS는 보통 UDP를 쓰지만 TCP를 쓰는 경우도 있다고 한다. 기준은 [페이로드 크기가 512바이트가 넘는 경우이고 이 경우 UDP로 커버가 안된다](https://www.infoblox.com/dns-security-resource-center/dns-security-faq/is-dns-tcp-or-udp-port-53/)고 한다. 512바이트는 IPv4 표준인 RFC 791에서 정의한 패킷 길이 576바이트에서 헤더 등을 뺀 페이로드 길이라고 한다. 더 긴 페이로드를 처리하는 DNS(e.g. [DNSSEC](https://ko.wikipedia.org/wiki/DNSSEC)) 프로토콜의 경우 TCP를 사용한다.

내가 실습한 로컬 minikube의 Calico나 killer.sh 시뮬레이터의 Weavenet CNI에선 둘 다 CoreDNS를 DNS 서버로 사용하는데 UDP로도 문제 없었다. Security 없이 IPv4만 사용해서 그런것 같다.

DNS를 찾을 수 있도록 egress 정책을 추가해도 여전히 'Connection timed out'으로 실패한다:
```sh
❯ k run san --rm -i --image=busybox --restart=Never -- nc -zv -w 3 nginx 1234
If you don't see a command prompt, try pressing enter.
nc: nginx (10.103.52.155:1234): Connection timed out
pod "san" deleted
pod netpol-test/san terminated (Error)

```

## 네트워크 정책의 LabelSelector 대상은 파드
nginx로 resolve 된 IP(10.103.52.155)를 보면, 위에서 직접 입력한 파드 IP(10.244.120.68)와 다르다. [서비스도 파드와 통신 가능하게 해주는 네트워크 엔티티지만 네트워크 정책 규칙에서 식별하진 않는다.](https://kubernetes.io/docs/concepts/services-networking/network-policies/) 네트워크 정책은 다음으로 네트워크 엔티티를 식별한다:
- podSelector
- namespaceSelector
- ipBlock(CIDR)

이렇게 매끄럽게 설명했지만 과정은 그렇지 않았다. 서비스 이름과 노출 포트(port)를 같이 묶어서 생각하다 보니 오류를 바로 잡기 쉽지 않았다. 결국 노출 포트가 아닌 서비스의 목적 포트(targetPort)이자 파드의 포트인 80을 허용해줘야 한다:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: san-to-nginx
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      run: san
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: nginx
    ports:
    - protocol: TCP
      port: 80
  - to: []
    ports:
    - protocol: UDP
      port: 53

```

이제 포트 스캔이 성공한다:
```sh
❯ k run san --rm -i --image=busybox --restart=Never -- nc -zv -w 3 nginx 1234
nginx (10.103.52.155:1234) open
pod "san" deleted

```

## Service는 NAT rule
스캐닝 하는 포트(1234)와 egress 규칙에서 허용 포트(80)가 달라 여전히 헷갈릴 수 있다. 여기서 서비스의 구현과 역할을 이해하는게 중요하다.

서비스의 IP 선택한 파드 집합의 IP로 요청을 프록시 가상IP(VIP)이다. [kube-proxy의 `--proxy-mode` 옵션 기본 값](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)인 [iptables](https://linux.die.net/man/8/iptables)에 의해 NAT rule로 설정 된다. 따라서 서비스의 IP와 포트는 실제 호스트, 프로세스 같은 리소스와 직접 매치하지 않고 프록시 처리 된다.

이번에도 KTHW를 세팅하지 않아 iptables 명령을 직접 해보진 못했다(...). 맥에선 iptables를 사용하지 않아, minikube에선 다른 방법으로 프록시할 것 같다.

## 정리
- 네트워크 정책의 egress 설정 시 DNS도 막지 안도록 주의하자. 보통은 원치 않는 동작일 것이다.
- 네트워크 정책의 대상 엔티티는 파드들이다. 서비스 노출 포트와 헷갈리지 않게 주의하자.
- 서비스는 파드들의 NAT rule 설정된 VIP + 포트이다.


---

## 참고
- https://www.infoblox.com/dns-security-resource-center/dns-security-faq/is-dns-tcp-or-udp-port-53/
- https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/network-policy-v1/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/concepts/services-networking/service/
