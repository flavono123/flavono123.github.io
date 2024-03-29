---
title: "CKA 시험 후기와 Security Context - 리눅스 네임스페이스, capabilities"
date: 2022-02-25T18:24:34+09:00
tags:
- cka
- docker
- linux
---

이전까지 쓰던 CKA 강의 정리 글을 중단하고, 남은 섹션 빠르게 공부한 후, 합격했다...

인터넷에 보면 시험 (합격)후기 글이 많다. 시험 팁과 관련된 내용이 주로 있고 나도 그런 글 도움을 많이 받았다. 하지만 나는 조금 다른 이야길 해보려 한다.

우선 강의와 더불어 시험을 보는 과정에서 배우는 것들이 좋았다. 어찌보면 난 컨테이너(도커)는 아주 깔짝, 리눅스도 깔짝 알고 있는 수준이었다는게 드러났다. 예컨데 나는 리눅스 네트워크는 꽤 잘 안다고 착각했다. 이유는 (지금 생각하면 조금 어이 없지만) on-premise 호스트에서 DHCP를 쓰지 않고 정적으로 IP를 할당했기 때문에 그렇다고 생각했다. 하지만 강의에서 컨테이너와 쿠버네티스의 가상 네트워크 구성을 듣고 난 리눅스 네트워크에 대해 거의 아는게 없다고 느껴졌다. 네트워크 2-3 계층을 구현하는 리눅스 명령과 프로그램이 있다는 걸 처음 알았다(몇몇은 알고 있었지만 그런 용도인줄 몰랐다. 이 내용은 꼭 실습하고 정리하려 한다).

네트워크 뿐만 아니라, 컨테이너 오케스트레이터로써 쿠버네티스는, 리눅스 커널과 컨테이너 관련 기술을 알아야 한다. 그리고 강의와 문제에서 이 기술들을 관통한다. 그래서 시험을 준비하며 알게 된 컨테이너, 리눅스 관련 내용을 정리하는 것으로 후기를 대체(?)하려 한다. 아, openssl을 사용해 self-signed certification은, 우연히, 시험 직전에 실습해봤는데 큰 도움이 됐다.

그 중 하나가 K8s의 [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) 관련한 리눅스 네임스페이스와 capabilites 관한 내용이다.

[실습 환경](https://github.com/flavono123/docker-the-hard-way)을 올려놨다.

## 리눅스 네임스페이스
강의에선 K8s Security Context를 이해하기 위해 도커의것부터 설명한다. 도커 즉, 컨테이너는 호스트와 커널을 공유하면서도 분리된 환경에서 실행된다. 네임스페이스를 통해 분리한다. [man 페이지](https://man7.org/linux/man-pages/man7/namespaces.7.html)도 있으나, [이 글](https://www.44bits.io/ko/keyword/linux-namespace)과 같이 보면 더 이해가 잘 된다.

실제로 출력해보면 다음과 같다:
```sh
$ sudo ls -l /proc/1/ns | awk '{print $11}'

cgroup:[4026531835]
ipc:[4026531839]
mnt:[4026531840]
net:[4026531992]
pid:[4026531836]
pid:[4026531836]
user:[4026531837]
uts:[4026531838]
```

컨테이너를 하나 실행해서 비교해보자:
```sh
$ docker run -d busybox sleep 3000
$ docker exec <container_id> ls -l /proc/1/ns | awk '{print $11}'

cgroup:[4026531835]
ipc:[4026532184]
mnt:[4026532182]
net:[4026532187]
pid:[4026532185]
pid:[4026532185]
user:[4026531837]
uts:[4026532183]

$ comm <(docker exec dbadb729e1d4 ls -l /proc/1/ns | awk '{print $11}' | sort) \
> <(sudo ls -l /proc/1/ns | awk '{print $11}' | sort) \
> --output-delimiter='                    '
                                        
                                        cgroup:[4026531835]
                    ipc:[4026531839]
ipc:[4026532184]
                    mnt:[4026531840]
mnt:[4026532182]
                    net:[4026531992]
net:[4026532187]
                    pid:[4026531836]
                    pid:[4026531836]
pid:[4026532185]
pid:[4026532185]
                                        user:[4026531837]
                    uts:[4026531838]
uts:[4026532183]

# diff
$ comm <(docker exec dbadb729e1d4 ls -l /proc/1/ns | awk '{print $11}' | sort) \
> <(sudo ls -l /proc/1/ns | awk '{print $11}' | sort) \
> --output-delimiter='                    ' -12

cgroup:[4026531835]
user:[4026531837]

# 컨테이너 것
$ comm <(docker exec dbadb729e1d4 ls -l /proc/1/ns | awk '{print $11}' | sort) \
> <(sudo ls -l /proc/1/ns | awk '{print $11}' | sort) \
> --output-delimiter='                    ' -23
ipc:[4026532184]
mnt:[4026532182]
net:[4026532187]
pid:[4026532185]
pid:[4026532185]
uts:[4026532183]

# 호스트 것
$ comm <(docker exec dbadb729e1d4 ls -l /proc/1/ns | awk '{print $11}' | sort) \
> <(sudo ls -l /proc/1/ns | awk '{print $11}' | sort) \
> --output-delimiter='                    ' -13
ipc:[4026531839]
mnt:[4026531840]
net:[4026531992]
pid:[4026531836]
pid:[4026531836]
uts:[4026531838]
```

컨테이너와 호스트는 cgroup과 user를 제외한 네임스페이스가 다르다. pid가 다른 것은, 컨테이너가 실행하는 프로세스가 컨테이너 내부에서 바라 볼 때 1인 것과 호스트에서 볼 때 다른 것으로 쉽게 이해할 수 있다:

```sh
$ docker exec dbadb729e1d4 ps aux | grep [s]leep
    1 root      0:00 sleep 3000
$ ps aux | grep [s]leep
root        6162  0.0  0.0   1308     4 ?        Ss   09:57   0:00 sleep 3000
```

네트워크 스택을 분리하는 net과 uts 네임스페이스 역시 그러하다(나중에 컨테이너와 쿠버네티스 네트워크를 공부할 때 다시 볼 것).

[cgroup](https://docs.docker.com/engine/security/#control-groups) 호스트의 리소스를 컨테이너에게 안전하게 할당하는 리눅스 컨테이너 컴포넌트이다. 호스트의 전체 리소스를 공유하며 관리 받기 위해 컨테이너도 같은 네임스페이스일 수 밖에 없을거 같다. user 역시 같다. 컨테이너는 호스트의 사용자의 권한으로 프로세스를 실행시키는 꼴이다. 기본은 root 이고 UID 인자를 주어 변경할 수 있다(`--user`). UID 1000인 vagrant 사용자로 실행해도 네임스페이스 분리된 모습은 같다:

```sh
$ docker run -d --user 1000 busybox sleep 3000

# diff 만 비교
$ diff -urN <(docker exec ad7a0ba892ae ls -l /proc/1/ns | awk '{print $11}' | sort) \
> <(sudo ls -l /proc/1/ns | awk '{print $11}' | sort)
--- /dev/fd/63	2022-02-25 10:25:49.879155366 +0000
+++ /dev/fd/62	2022-02-25 10:25:49.879155366 +0000
@@ -1,9 +1,9 @@
 
 cgroup:[4026531835]
-ipc:[4026532248]
-mnt:[4026532246]
-net:[4026532251]
-pid:[4026532249]
-pid:[4026532249]
+ipc:[4026531839]
+mnt:[4026531840]
+net:[4026531992]
+pid:[4026531836]
+pid:[4026531836]
 user:[4026531837]
-uts:[4026532247]
+uts:[4026531838]
```

## Capabilities
컨테이너는 실행될 때의 호스트 사용자 권한(previlege, capability)을 그대로 받기도 하지만, 특정 권한을 추가하거나 없애서 제어할 수도 있다. 이건 실제로 프로세스 실행 시 동작과 같고, 호스트는 컨테이너를 프로세스로 바라보기에 가능한것 같다.

이 때 권한들에 해당하는게 capabliities이다. 역시 [man 페이지](https://man7.org/linux/man-pages/man7/capabilities.7.html)가 있다.

`CAP_` 으로 시작하는것들이 프로세스가 할 수 있는 리눅스 동작에 대한 권한이다. [커널 코드에 매크로로 정의](https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h)되어 있다. 민망해서 주절주절 늘어 놓는데, 리눅스를 쓰며 처음 본 내용이다...

다음 글로 설명을 대체하며 쓸 일이 있을까도 싶다(물론 웹 서버를 root 권한으로 실행하고 있지 않다! 실무에선 웹 서버 외에 DB 같은 프로세스를 root로 실행해야 하는지 또는 별도의 리눅스 사용자로 실행해야 하는지 아는게 더 중요한거 같다)([출처](https://docs.docker.com/engine/security/#linux-kernel-capabilities)):
> Capabilities turn the binary “root/non-root” dichotomy into a fine-grained access control system. Processes (like web servers) that just need to bind on a port below 1024 do not need to run as root: they can just be granted the net_bind_service capability instead. And there are many other capabilities, for almost all the specific areas where root privileges are usually needed.

프로세스의 capablities는 proc 파일 시스템 status에 인코딩 되어 있다. 각 인코딩 값에 대한 설명은 [링크](https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities)로 대체한다:
- CapInh(Inherited from the parent processes)
- CapPrm(Permitted to the threads)	
- CapEff(Effective)
- CapBnd(Bounding)
- CapAmb(Ambient)

`capsh` 명령을 통해 디코딩하면 값을 쉽게 확인할 수 있다. root와 vagrant 사용자로 실행한 컨테이너의 프로세스를 비교하면 capabilities 차이를 확인할 수 있다:

```sh
# root
$ docker exec 2561c489efdc cat /proc/1/status | grep -i cap | awk '{print $2}' | xargs -I{} capsh --decode={}
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x0000000000000000=

# vagrant
$ docker exec b01846125ba6 cat /proc/1/status | grep -i cap | awk '{print $2}' | xargs -I{} capsh --decode={}
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x0000000000000000=
0x0000000000000000=
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x0000000000000000=
```

root로 실행한 컨테이너는 CapAmb를 제외한 모든 권한이 꽉꽉 차 있지만, vagrant로 실행한 것은 CapPrm, CapBnd이 비어 있다. [`--cap-drop` 옵션을 주어 컨테이너를 실행하면 특정 권한을 제거할 수 있다](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities):
```sh
$ docker run -d --user 1000 --cap-drop cap_chown busybox sleep 2000
f0b5e2569f6b2d4ec3283ef0f7761e82584c9caa879d5fd9c7573f0efc5435d6
$ docker exec f0b5e2569f6b cat /proc/1/status | grep -i cap | awk '{print $2}' | xargs -I{} capsh --decode={}
0x00000000a80425fa=cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x0000000000000000=
0x0000000000000000=
0x00000000a80425fa=cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
0x0000000000000000=

```

반대로 `--cap-add`로는 특정 권한을 추가할 수 있다.

## Security Context

위 내용을 이해하면 K8s에서 파드의 리눅스 사용자와 컨테이너의 권한을 설정은 쉽게 이해할 수 있다.

사용자는 파드 단위([`spec.securityContext.runAsUser`](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)) 또는 컨테이너 단위([`spec.containers[].securityContext.runAsUser`](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-container))로 설정할 수 있다.

Capabilities는 컨테이너 단위로만 설정할 수 있다([`spec.containers[].securityContext.capablities`](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container))

명령 실행은 도커 것과 결과가 거의 비슷해 생략한다.

---

## 참고
- https://docs.docker.com/engine/security/#linux-kernel-capabilities
- https://tech.ssut.me/what-even-is-a-container/
- https://www.44bits.io/ko/keyword/linux-namespace
- https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities
- https://man7.org/linux/man-pages/man1/capsh.1.html
- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
- https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/
