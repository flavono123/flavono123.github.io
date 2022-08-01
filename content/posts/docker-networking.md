---
title: "도커 네트워크"
date: 2022-07-12T00:27:51+09:00
draft: true
tags:
- container
- network
---

컨테이너는 한 호스트 또는 여러 다른 네트워크에 배포될 수 있기 때문에 컨테이너끼리 연결, 네트워크는 복잡하다. 그래서 나도 이해하는게 오래 걸렸다. 크게 두 가지 이유로 어려웠다. 하나는 개념, 키워드를 배울 수록 이게 어떤 계층의 이야기인지 알기 어려웠다. 리눅스의 네트워크 스택인지 하이퍼바이저의 장비 가상화에 대한 내용인지 컨테이너인지 오케스트레이션(쿠버네티스)인지 정리가 잘 안됐다. 다른 하나는 네트워크 스택은, 특히 가상화된 장비 쪽으로 가면, 코드나 명령으로 결과를 바로 보기 어려워서 잘 이해를 못했다.

그러다 [Docker Deep Dive](https://www.oreilly.com/library/view/docker-deep-dive/9781800565135/) 네트워킹 파트를 보고 조금은 이해할 수 있게 됐다. 이 글은 책의 11, 12장 위주로 정리했다.

## CNM, libnetwork, 드라이버
Container Network Model(CNM)은 말 그대로 컨테이너 네트워크의 설계 명세이다. 여기엔 중요한 세개의 구성이 있다:

- 샌드박스: 컨테이너의 고립된 네트워크 스택(네트워크 네임스페이스). 네트워크 인터페이스, 라우팅 테이블, DNS 구성을 포함한다. 여러 네트워크의 **여러 엔드포인트**를 가질 수 있다.
- 엔드포인트: 가상의 인터페이스(veth), 컨테이너 종단간 쌍을 이룬다. Open vSwitch 내부 포트와 비슷한 개념. **네트워크와 샌드박스를 연결한다.**
- 네트워크: 스위치(802.1d, 브릿지라고도 함)의 소프트웨어 구현. 통신해야할 **여러 엔드포인트**의 묶음.

[명세 문서의 그림](https://github.com/moby/libnetwork/blob/master/docs/design.md#the-container-network-model)을 보며 이해하자. 컨테이너는 샌드박스를 통해 여러 엔드포인트로 여러 네트워크에 연결할 수 있다(가운데 컨테이너). 또 그림에 표현되진 않았지만, 컨테이너가 같은 호스트에 위치하지 않더라도 즉, 물리적으로 다른 네트워크에 위치해도 컨테이너는 CNM의 네트워크를 통해 직접 연결될 수 있어야 한다.

CNM의 구현은 `libnetwork`이다. 앞서 말한 CNM의 3대 구성, 샌드박스, 엔드포인트, 네트워크가 구현되어 있다. 추가로 내장된 서비스 디스커버리와 인그레스 기반 컨테이너 로드밸런싱이 구현되어 있다. `libnetwork`는 컨테이너 네트워크의 컨트롤 플레인과 관리 플레인 역할이다.

`libnetwork`에서 CNM의 네트워크를 pluggable하게만 구현하고 실제 구현은 드라이버에게 위임한다. 네트워크의 연결과 다른 네트워크와의 격리와 같은 실제 네트워크의 구현은 드라이버에서 한다. 드라이버는 네트워크의 데이터 플레인의 역할이다. 드라이버는 크게 local network과 remote driver로 나뉜다. Local network는 도커 내에 미리 구현된 네이티브 빌트인 네트워크이다. Remote driver는 [Weave Net](https://www.weave.works/oss/net/)과 같은 서드 파티에서 구현한 네트워크이다. 여기선 다음 local network 몇가지만 알아본다:
- 브릿지(단일 호스트)
- Macvlan(외부 네트워크에 연결)
- Overlay(다중 호스트)

## 브릿지 네트워크
브릿지는 802.1d, L2 스위치를 지칭하는 또 다른 말이다. 도커를 설치한 호스트마다 기본으로 같은 이름(bridge)의 브릿지 네트워크가 생성되고 기본으로 여기에 연결된다:
```sh
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
fb3cbeb79136   bridge    bridge    local

$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "fb3cbeb79136ff648e1d3d99ccb1384e655b3827ba8d6db780e9cee3e2628698",
        "Created": "2022-06-28T03:42:53.895128076Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

브릿지는 컨테이너와 함께 새로 생겨난 도커만의 네트워크가 아니라, 리눅스 커널에 원래 존재했던 네트워크 기술이다. 따라서 표준 리눅스 유틸리티로 검사할 수 있다. 위 inspect 결과에서 보이는 브릿지의 이름이 호스트 인터페이스 이름이다(`docker0`):
```sh
$ docker network inspect bridge  | jq -r '.[].Options."com.docker.network.bridge.name"'
docker0

$ ip link show docker0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:ec:e9:fd:d2 brd ff:ff:ff:ff:ff:ff
```

`bridge`는 도커 엔진이, `docker0`는 리눅스 커널, OS 도구가 사용하는 이름이다. 따라서 컨테이너 - bridge - docker0 - 이더넷 인터페이스로 연결된다.

새로운 브릿지 네트워크(`localnet`)를 만들어 같은 호스트의 컨테이너 사이 통신을 해본다:
```sh
$ docker network create -d bridge localnet
30441f25c1f0b9fbaf3689b02b51497988114c9cd74f96bf7d37afb30156a9c5

# apt install bridge-utils
$ brctl show | column -t
bridge           name               bridge  id  STP  enabled  interfaces
br-30441f25c1f0  8000.0242cc33b7da  no
docker0          8000.0242ece9fdd2  no
```

호스트엔 `br-30441f25c1f0`라는 이름으로 추가됐다. 컨테이너 두개를 `locanet` 네트워크에 붙여 연결해보자. [`docker container run` 명령에 `--network` 플래그로 네트워크를 지정할 수 있다](https://docs.docker.com/engine/reference/commandline/container_run/#options):
```sh
$ docker container run -d --name c1 --network localnet alpine sleep 1d
$ docker network inspect localnet  | jq -r '.[].Containers'
{
  "32dc71f35b767c4474df550b6112ed8403eb8cfbd9869a725dcee2a10bdf6855": {
    "Name": "c1",
    "EndpointID": "03e677435fdcaef5ee5ffaa8b467c1b328488b24522545dfe1fb6a2e087da8c8",
    "MacAddress": "02:42:ac:12:00:02",
    "IPv4Address": "172.18.0.2/16",
    "IPv6Address": ""
  }
}

$ docker container run -it --name c2 --network localnet alpine sh
/ # nslookup c1
Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:

Non-authoritative answer:
Name:   c1
Address: 172.18.0.2
```

`c2`에서 `c1`으로 DNS 쿼리가 잘된다. 하지만 기본 브릿지 네트워크인 `bridge`에선 이게 안된다는걸 주의해야 한다:
```sh
# 기본 bridge 네트워크는 DNS 해석이 안된다
$ docker container run -d --name c3 alpine sleep 1d
74243e53e3e92ba02cd3d9215799f574f2db473067a5a1e37d54dd727d8c2d9d
$ docker container run -it --name c4 alpine sh
/ # nslookup c3
Server:         10.0.2.3
Address:        10.0.2.3:53

** server can't find c3: NXDOMAIN

** server can't find c3: NXDOMAIN
```

새로 만든 `localnet` 브릿지 네트워크에선 컨테이너의 로컬 DNS resolver가 도커 내부 DNS 서버에 요청을 포워딩하여 DNS를 해석할 수 있다. 도커 내부 DNS 서버에선, 컨테이너 실행 시 `--name` 또는 `--net-alias` 플래그와 IP를 매핑하여 관리한다:
```sh
```bash
# c2
/ # ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:03
          inet addr:172.18.0.3  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:9 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:726 (726.0 B)  TX bytes:0 (0.0 B)

# 호스트
docker network inspect localnet | jq -r '"\(.[].IPAM.Config) \(.[].Containers)" '
[{"Subnet":"172.18.0.0/16","Gateway":"172.18.0.1"}] {"32dc71f35b767c4474df550b6112ed8403eb8cfbd9869a725dcee2a10bdf6855":{"Name":"c1","EndpointID":"03e677435fdcaef5ee5ffaa8b467c1b328488b24522545dfe1fb6a2e087da8c8","MacAddress":"02:42:ac:12:00:02","IPv4Address":"172.18.0.2/16","IPv6Address":""},"67e4429b6630859488abb82adb1cefbc5adb8f5ffbf50aa1ee8eb89f34f60658":{"Name":"c2","EndpointID":"c14b55997edbe0696166b2faae86384c2908e8c3666546d4482e4259babb91de","MacAddress":"02:42:ac:12:00:03","IPv4Address":"172.18.0.3/16","IPv6Address":""}}
```

컨테이너와 호스트의 포트를 매핑하여 호스트 트래픽을 컨테이너로 향하게 할 수 있다. 다음은 호스트의 5000 포트를 컨테이너의 80 포트와 매핑하여 호스트에서 컨테이너로 요청할 수 있게 된다:
```sh
$ docker container run -d --name web --network localnet --publish 5000:80 nginx
$ docker port web
80/tcp -> 0.0.0.0:5000
80/tcp -> :::5000
$ curl 127.0.0.1:5000
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Macvlan
컨테이너를 외부 시스템이나 물리 네트워크에 연결할 수 있는 네트워크 드라이버이다. 이 네트워크에선 컨테이너와 컨테이너화 안된 외부 앱과 통신을 할 수 있게 된다. 지금은 단순히 호스트에서 연결해본다:
```sh
# enp0s3.100 의 .100은 서브 인터페이스의 태그를 뜻함
$ docker network create -d macvlan --subnet 10.0.2.0/24 --ip-range 10.0.2.0/25 --gateway 10.0.2.1 -o parent=enp0s3.100 macvlan100
$ docker container run -d --name mactainer1 --network macvlan100 alpine sleep 1d
$ docker container inspect mactainer1  | jq -r '.[].NetworkSettings.Networks.macvlan100.IPAddress'
10.0.2.2

$ ping 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=64 time=0.116 ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=64 time=0.116 ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=64 time=0.186 ms
```

Macvlan 생성에 사용한 플래그를 살펴보면, `--subnet`, `--ip-range`, `--gateway`는 직관적이다. 옵션 플래그(`-o`)에서 parent는 호스트에서 사용할 (서브)인터페이스를 뜻한다. 여기선 호스트 인터페이스 `enp0s3`에 `.100` 태그를 붙여 서브인터페이스를 만들었다(VLAN Trunking). Macvlan에선 컨테이너마다 VM이나 물리 서버처럼 MAC + IP 주소를 제공한다. 따라서 포트 매핑이나 브릿지가 필요 없기 때문에 성능이 좋다. 다만 연결하는 인터페이스에 promiscuous 모드가 켜져 있어야 해서 공공 클라우드에선 적합하지 않다.

## Overlay
Overlay 네트워크는 여러 호스트(Multi-host)에 여러 다른 네트워크에 있는 컨테이너들을 하나의 L2 스위치에서 통신하는 네트워킹이다. 여러 호스트에서 사용하는 네트워크이기 때문에 다른 네트워크에 있는 두 개 호스트를 스웜으로 구성하여 실습해본다. node1과 node2는 각각 192.168.1.0/24, 172.31.1.0/24에 대역에 있고 둘은 라우터로 연결했다(10.2)

---

## 참고

