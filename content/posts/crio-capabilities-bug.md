---
title: "컨테이너 런타임 CRI-O 사용 시 오류 해결: ping: permission denied (are you root?)"
date: 2022-08-10T10:18:02+09:00
tags:
- kubernetes
- container
---

## 문제
쿠버네티스 컨테이너 런타임으로 containerd 대신 [CRI-O](https://cri-o.io/)를 검토하던 중 ping 테스트에서 다음 오류가 발생했다:
```sh
/ # ping 172.16.40.67
PING 172.16.40.67 (172.16.40.67): 56 data bytes
ping: permission denied (are you root?)
```

CNI는 Calico를 사용했고, [ping 테스트는 Calico 문서](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/test-networking) 것 그대로 했다(IP는 다른 워커 노드의 파드의 것이다).

## 해결
처음 보는 오류 메시지를 검색하니 CRI-O는 [CAP_NET_RAW](https://man7.org/linux/man-pages/man7/capabilities.7.html)라는 ping에 필요한 capability가 기본으로 없다는 것을 알게 됐다:

### default_capabilities

```sh
$ crio config | grep default_capabilities -A 10
INFO[2022-08-10 06:14:49.820468258Z] Starting CRI-O, version: 1.24.2, git: bd548b04f78a30e1e9d7c17162714edd50edd6ca(clean)
INFO Using default capabilities: CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FSETID, CAP_FOWNER, CAP_SETGID, CAP_SETUID, CAP_SETPCAP, CAP_NET_BIND_SERVICE, CAP_KILL, CAP_NET_RAW
default_capabilities = [
        "CHOWN",
        "DAC_OVERRIDE",
        "FSETID",
        "FOWNER",
        "SETGID",
        "SETUID",
        "SETPCAP",
        "NET_BIND_SERVICE",
        "KILL",
]
```

원랜 CAP_NET_RAW와 CAP_SYS_CHROOT도 default_capabilities에 있었으나 [1.18에서 보안 강화를 위해 삭제](https://cri-o.github.io/cri-o/v1.18.0.html)됐다.

따라서 crio config TOML 파일(Ubuntu에선 /etc/crio/crio.conf 또는 /etc/crio/crio.conf.d/ 하위)에 다음을 추가하여 CAP_NET_RAW를 default_capabilities에 추가했다:
```sh
[crio.runtime]
default_capabilities = [
	"CHOWN",
	"DAC_OVERRIDE",
	"FSETID",
	"FOWNER",
	"SETGID",
	"SETUID",
	"SETPCAP",
	"NET_BIND_SERVICE",
	"KILL",
	"NET_RAW",
]
```

crio 서비스를 **재시작**하고 나니 ping이 되기 시작했다.


```sh
# crio 1.24.2
/ # setpriv -d
uid: 0
euid: 0
gid: 0
egid: 0
Supplementary groups: 10
no_new_privs: 0
Inheritable capabilities: [none]
Ambient capabilities: setpriv: prctl: CAP_AMBIENT_IS_SET: Invalid argument

/ # ping 172.16.25.199 -c 5
PING 172.16.25.199 (172.16.25.199): 56 data bytes
64 bytes from 172.16.25.199: seq=0 ttl=62 time=0.662 ms
64 bytes from 172.16.25.199: seq=1 ttl=62 time=0.690 ms
64 bytes from 172.16.25.199: seq=2 ttl=62 time=0.702 ms
64 bytes from 172.16.25.199: seq=3 ttl=62 time=0.570 ms
64 bytes from 172.16.25.199: seq=4 ttl=62 time=0.712 ms

--- 172.16.25.199 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 0.570/0.667/0.712 ms
```

CRI-O가 contaierd에 비해 경량이라는 이유로 PoC를 진행해보고 있었는데, 컨테이너 보안 측면에서도 공격 표면이 더 적다는 것을 체감했다. 또 Linux capabilities도 종류가 너무 많아 어찌 공부할지 감이 안왔는데 NET_RAW 등 몇가지를 알게 됐다.

## 정리
- CRI-O는 containerd에 비해 컨테이너 프로세스에 허용하는 기본 capabilities가 적다.
- 패킷 소켓을 사용하는 ping 같은 명령엔 CAP_NET_RAW capability가 필요하다.


---

## 참고
- https://www.youtube.com/watch?v=ZKJ9oFwjosM
- https://cri-o.io/
- https://github.com/cri-o/cri-o/blob/main/docs/crio.conf.5.md
- https://man7.org/linux/man-pages/man7/capabilities.7.html
- https://busybox.net/
