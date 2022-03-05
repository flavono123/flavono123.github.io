---
title: "ip: 리눅스 네트워크 장치"
date: 2022-03-06T00:03:41+09:00
tags:
- linux
- network
---

리눅스에서 [ip](https://linux.die.net/man/8/ip) 명령을 통해 네트워크 장치를 확인하고 구성할 수 있다. 그런데 이더넷 카드 이름이 언젠가부터(?) eth\*가 아닌 enp\*s\*로 나오고 있다. '언젠가'는 리눅스를 VMWare에 처음 설치해봤던 학생 때이고 지금은 시간이 많이 지났다. eth는 이더넷이란 이름에 찰싹 달라 붙어 받아 들였지만, eno\* 또는 enp\*s\* 역시 이더넷 카드를 뜻하는건지 궁금해졌다. 이번에도 알게 된 점 그리고 모르기로 한 부분을 나눠 정리한다.

## Predictable Network Interface Names

위 같은 변화는 systemd가 나오며 시작됐다(정확하게는 [v197](https://lwn.net/Articles/531850/)라는 배포판에서 이다). 과거엔 간단하게 eth 뒤에 인덱스를 붙여도 드라이버가 찾을 수 있었지만, 오늘 날의 네트워크 구성 기술에선 이름을 쉽게 바꿀 수 있고 같은 이름으로도 여러 곳에서 사용할 수 있어 적합하지 않다고 한다.

그래서 예측 가능한(predictable) 이름을 사용하기로 했다. 여러 개정 끝에 나온 룰은 다음과 같다.
> 1. Names incorporating Firmware/BIOS provided index numbers for on-board devices (example: eno1)
> 2. Names incorporating Firmware/BIOS provided PCI Express hotplug slot index numbers (example: ens1)
> 3. Names incorporating physical/geographical location of the connector of the hardware (example: enp2s0)
> 4. Names incorporating the interfaces's MAC address (example: enx78e7d1ea46da)
> 5. Classic, unpredictable kernel-native ethX naming (example: eth0)


... 보드, PCI등 하드웨어 관련한것은 바꿔 꼽으면서 디버깅 할 수도 없기 때문에 그러려니 했다(구글에서 PCI, PCI Express, 슬롯, 버스, 보드 등 이미지를 보고 뭔지 잘 모르겠지만 으응~했다 ㅎㅎ).  그렇다하더라도 내가 궁금한 경우에 해당하는 세번째의 설명이 추상적이다. p와 s 뒤의 인덱스는 각각 PCI의 버스와 슬롯 위치를 뜻한다.
이 [답변](https://unix.stackexchange.com/questions/134483/why-is-my-ethernet-interface-called-enp0s10-instead-of-eth0)에서 보기 쉽게 설명하고 [매뉴얼](https://www.freedesktop.org/software/systemd/man/systemd.net-naming-scheme.html)도 자세히 보면 나온다. 내가 보기엔 더 알아볼 수 없는 이름이지만 드라이버가 참 찾기 쉽겠구나.. 싶었다.

난 버추얼박스에 우분투를 띄워서 확인했는데, 그럼 가상 머신은 어떻게 PCI를 구현한걸까? 인터페이스 이름과 대조하여 검증할 수 있을까? [버추얼막스 매뉴얼](https://www.virtualbox.org/manual/ch06.html) PCI 이야기가 쓰여 있지만 어질어질해서 그냥 이 부분은 몰라도 되겠다 하고 말았다(...).

## ip 

하드웨어는 최대한 추상화해서 이해하자. enp\*s\* 같은 인터페이스 이름 뿐만 아니라 ip 명령의 결과도 대충 느낌적으로 알고 있기 때문에 이번에 정리해본다.

ip addr 명령을 ubuntu focal에서 실행해봤다:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.box_version = "20220215.1.0"
  end
end
```

```sh
vagrant@ubuntu-focal:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:10:bc:02:65:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 79730sec preferred_lft 79730sec
    inet6 fe80::10:bcff:fe02:654d/64 scope link 
       valid_lft forever preferred_lft forever
```

- 첫째 줄 가장 앞, 콜론 앞에 있는 숫자는 인터페이스 인덱스(interface index; ifindex)
- 다음은 인터페이스 네임
- 플래그 요약(있는 것만 설명)
  - UP: 켜짐, 패킷을 받아 커널어 넣을 수 있는 상태
  - LOOPBACK: 다른 호스트와 통신하지 않음(보내는 모든 패킷이 되돌아옴)
  - BROADCAST: 같은 링크(스위치)의 호스트에게 모두 패킷을 보냄
  - MULTICAST: 같은 링크의 부분 그룹 호스트에게 패킷을 보냄, BROADCAST는 MULTICAST의 특수한 경우이다
- MTU(Max transmission unit)를 바이트로 나타냄(65536, 1500)
- qdisc(queueing discpline)
- 그룹
- 큐 길이(qlen 1000), 이 이상의 큐의 패킷은 드랍됨
- 둘째 줄 link/ 뒤는(loopback, ether) 하드웨어 종류, 맥(MAC) 주소와 브로드캐스트 주소
- inet과 inet6는 각각 IP(v4), IPv6 link와 같이 인터페이스 주소와 브로드캐스트 주소
- scope
  - global: the address is globally valid.
  - site: (IPv6 only) the address is site local, i.e. it is valid inside this site.
  - link: the address is link local, i.e. it is valid only on this device.
  - host: the address is valid only inside this host.
- 플래그
  - dynamic: DHCP에 의해 동적 할당됨, 유효 시간(valid_lft)이 있음
- lft은 lifetime을 뜻하고 동적 할당된 경우에 값이 있다. 같게 설정되어 있어서 invalid 후에 바로 handoff 될 것 같다

로컬에 VM으로 k8s 클러스터 구성하려다 여기까지 와 버렸다. 그만 알아보자.


---

## 참고
- https://www.freedesktop.org/wiki/Software/systemd/
- https://askubuntu.com/questions/704035/no-eth0-listed-in-ifconfig-a-only-enp0s3-and-lo
- https://major.io/2015/08/21/understanding-systemds-predictable-network-device-names/
- https://unix.stackexchange.com/questions/134483/why-is-my-ethernet-interface-called-enp0s10-instead-of-eth0
- https://www.freedesktop.org/software/systemd/man/systemd.net-naming-scheme.html
- https://unix.stackexchange.com/questions/465563/how-to-understand-ifconfig-or-ip-addr-show
- http://linux-ip.net/gl/ip-cref/ip-cref.html
