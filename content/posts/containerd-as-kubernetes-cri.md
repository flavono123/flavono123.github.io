---
title: "K8s CRI와 containerd 설치"
date: 2022-05-08T16:45:29+09:00
tags:
- kubernetes
- docker
---

kubeadm으로 쿠버네티스 클러스터를 구성하던 중 [컨테이너 런타임 설치 부분](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime)에서 혼란이 있었다. 전부터 containerd, 런타임, 쿠버네티스의 도커 지원 중단 같은 이야기를 들었지만 명확히 알진 않았다.

따라서 컨테이너 런타임이 무엇인지, 또 구성하게 될 쿠버네티스 클러스터에선 containerd가 어떤 역할을 할지 정리한다.

## 도커 역사와 구조 변화
이 내용은 [Docker Deep Dive](https://www.amazon.com/Docker-Deep-Dive-Nigel-Poulton/dp/1521822808/ref=tmm_pap_swatch_0?_encoding=UTF8&qid=1611059631&sr=8-1)의 파트1 2장 Docker와 파트2 5장 The Docker Engine을 참고했다. 그리고 책에서 말하듯, 도커의 구조는 도커와 OCI의 역사 관련한 설명을 빼고 설명하기가 어렵다. 특히 역사, 과거의 사건, 관련해선 저자의 말을 빌려 간단하게 압축하여 설명한다. 또 "쿠버네티스에서 containerd의 역할"을 설명하는데 그리 중요하지 않은 부분들은 설명을 간단히 한다.

먼저 과거 도커는, 현재의 모습과 다르게, 모놀로딕 구조(도커 대몬)에 [lxc](https://linuxcontainers.org/)가 합쳐진 구조였다. 이 구조는 크게 두가지 이유에서 현재처럼 개선된다:
1. OS 이식성: lxc는 리눅스 한정적
2. 컨테이너 생태계: 모놀로딕 도커 대몬은 서드 파티(e.g. 쿠버네티스)가 필요한 부분만 사용하기에 무겁다

따라서 lxc는 OS 중립적인(platform-agnostic) [libcontainer](https://pkg.go.dev/github.com/opencontainers/runc/libcontainer)의 구현으로 대체되었고, 모놀리딕 도커 대몬은 그 크기가 작아지며 여러 레이어로 나뉘게 된다. 아래 그림은 순서대로 더 추상적인 묘사와 구체적인 것의 현재의 도커 구조이다:

![1.png](/images/containerd-as-kubernetes-cri/1.png)
![2.png](/images/containerd-as-kubernetes-cri/2.png)

우리가 관심 있는 containerd를 포함해 여러 컴포넌트가 레이어 별로 있다. containerd를 이해하기 위해 나머지 컴포넌트도 간단히 설명해본다(두번째 그림 기준):
- Docker client(CLI): Docker daemon에게 명령한다.
- Docker daemon
  - CLI의 API 명령을 받는 부분의 구현
  - containerd에 명령
- containerd
  - 주된 임무는 컨테이너의 생명주기를 운영한다(start, stop, pause, rm, ...)
  - 그외 추가적인 기능도 한다(컨테이너 이미지 받기, 볼륨, 네트워킹, ...)
  - 도커 이미지를 OCI 번들로 바꾸어 runc에 전달
- shim
  - runc가 종료되어도 컨테이너 부모로 남아 입출력을 유지하고 나중에 종료 상태를 대몬에 보고
- runc
  - libcontainer의 래퍼(wrapper)이자 컨테이너 생성(create)을 담당하는 바이너리
  - 커널에 필요한 컨테이너 구성 요소(네임스페이스, cgroup, ...)을 모아 컨테이너 생성
  - OCI 런타임 명세의 구현체

기존엔 각 레이어와 컴포넌트 구분 없이 하나의 대몬으로 동작하던 도커가 이처럼 모듈로 나뉘었다. 여기서 [OCI](https://opencontainers.org/)라는 용어가 나온다.

간단히 설명하면 도커가 시장점유율이 아주 높던 시기에 [경쟁사](https://github.com/coreos/)가 등장하여 [새로운 표준](https://github.com/appc/spec/)을 제안하였는데, 이로 인해 혼란이 생기는 것을 막기 위해 합의점을 본 것이 OCI라는 컨테이너 관련 표준을 지정하는 기구를 만든 것이다.

실질적으로 기구 자체보다는 표준을 지칭하는 용어로 더 많이 쓰인다. OCI가 중요하게 관리하는 표준은 크게 두가지이다(containerd의 구현까지 볼게 아니니 "있다" 정도로 알면 되겠다):
1. [이미지 명세](https://github.com/opencontainers/image-spec)
2. [런타임 명세](https://github.com/opencontainers/runtime-spec)

앞서 말한 구식 도커 모놀리딕 대몬의 모듈화와 OCI의 탄생을 비슷한 시기에 진행됐다(책에선 약간의 타임라인도 알려주지만... 그저 비슷한 시기에 발생했다고만 이해했다). 그리고 서로에게 영향을 준거 같다. 실제로 런타임 명세를 따르는 구현인 runc는 도커 직원들이 개발에 많이 참여했고, containerd도 이 명세를 따라 개발됐다.

도커의 자세한 구조와 여러 컴포넌트를 설명했지만, 쿠버네티스에서 containerd란 무엇인지 알기 위해선 다음의 한 문장으로 앞선 내용을 요약한다:
- containerd는 도커에서 OCI를 준수하여 만든 컨테이너 런타임이다.


## K8s: 컨테이너 런타임 인터페이스(CRI)
쿠버네티스는 containerd 뿐만 아니라 컨테이너 런타임으로써 인터페이스를 [명세](https://github.com/kubernetes/cri-api)해두었다. 이것을 컨테이너 런타임 인터페이스, CRI라고 한다. containerd는 이 CRI를 따르기 때문에 쿠버네티스의 컨테이너 런타임으로써 사용할 수 있다.

하지만 과거 쿠버네티스는 CRI에 맞는 컨테이너 런타임이 아니라, dockeshim이라는, 도커 전체를 감싸는 인터페이스를 사용했다. 이와 관련해선 정확한 역사를 찾아보진 못했다. 추측하기론 CRI라는 명세를 만들면서 많이 쓰이고 있는 컨테이너 도구인 도커를 그에 맞춘 dockershim을 만들어 쓴거 같다.

최근 쿠버네티스는 dockershim의 CRI 지원을 중단했다. 이와 관련한 블로그 글이나, [불안을 줄이기 위한 문답 글](https://kubernetes.io/ko/blog/2020/12/02/dont-panic-kubernetes-and-docker/)등을 많이 볼 수 있고 여기에서도 (OCI 입장에선 서드 파티인)쿠버네티스와 도커 그리고 (쿠버네티스 입장에선 CRI 구현체 중 하나인)containerd의 관계를 알 수 있다. 특히 [이 글](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-removal-affects-you/#role-of-dockershim)의 그림을 보면 dockershim, containerd 그리고 도커의 위치가 깔끔하게 정리된다:

![3.png](/images/containerd-as-kubernetes-cri/3.png)

기존에 쿠버네티스를 사용하지 않던 입장에서, 도커(라는 이름이 들어가는 것들 e.g. dockershim) 지원 중단이 어떤 의미인지 정확히 알지 못했다. 하지만 컨테이너 런타임으로 containerd를 쓴다면, 도커 CLI와 대몬이 없더라도 도커 이미지의 컨테이너가 잘 동작할 수 있겠다는 확신이 생겼다. 하지만 [기존에 dockershim을 사용했던 쿠버네티스의 런타임을 containerd로 바꿀 때엔 문제점도 있는 것으로 보인다](https://ikcoo.tistory.com/189).

## containerd 설치
kubeadm으로 구성하는 쿠버네티스 클러스터에서 사용하기 위한 , Ubuntu 20.04 기준, containerd 설치 Ansible 코드를 공유한다. [공식 문서의 지시사항](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)을 따라 만든 것이라 크게 설명할 내용은 없다. containerd의 TOML 설정 파일(/etc/containerd/config.toml)을 만들기 위해 [지난 포스트](/posts/ansible-filter-plugins-from-to-toml)에 공유한 Ansible `from_toml`, `to_toml` 커스텀 필터를 사용했다:

```yaml
- name: Create the module file for containerd
  become: yes
  file:
    state: touch
    path: /etc/modules-load.d/containerd.conf
    access_time: preserve
    modification_time: preserve

- name: Configure containerd module
  become: yes
  blockinfile:
    insertafter: EOF
    path: /etc/modules-load.d/containerd.conf
    block: |
      overlay
      br_netfilter
- name: Add modules overlay and br_netfilter
  become: yes
  command: "modprobe {{ item }}"
  with_items:
  - overlay
  - br_netfilter

- name: Setup required sysctl net params
  become: yes
  sysctl:
    state: present
    name: "net.{{ item }}"
    value: 1
    sysctl_file: /etc/sysctl.d/99-kubernetes-cri.conf
  with_items:
  - bridge.bridge-nf-call-iptables
  - ipv4.ip_forward
  - bridge.bridge-nf-call-ip6tables

- name: Add Docker GPG key
  become: yes
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Configure Docker's upstream APT repository
  become: yes
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    update_cache: true

- name: Install containerd
  become: yes
  apt:
    state: present
    update_cache: yes
    name:
    - containerd.io

- name: Get containerd default configurations
  become: yes
  command: containerd config default
  register: containerd_default_config
  run_once: yes

- name: Set containerd config
  set_fact:
    containerd_config: "{{ containerd_default_config.stdout | from_toml | combine(containerd_systemd_cgroup_driver, recursive=True) }}"

- name: Template the containerd configuration
  become: yes
  template:
    src: containerd_config.toml
    dest: /etc/containerd/config.toml
    mode: '0644'
    owner: root
    group: root
  notify: restart containerd
  vars:
    config: "{{ containerd_config }}"
```

container.io 패키지 설치와 더불어, overlay와 br_netfilter 모듈 추가와 IP forward, iptables 제어 관련한 sysctl 변수 변경, 그리고 cgroup driver 사용을 위한 설정 변경까지 해준다. 설치 후 [containerd의 기본 소켓은 kubelet이 잘 찾도록 되어 있어서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node) 따로 설정이 필요하진 않았다.


## 정리
containerd가 무엇이고 어떻게 개발되어 Dockershim이 CRI로써 지원 중단되는 이 시기에 쿠버네티스에서 어떻게 사용되는지 알아봤다. containerd는 dockershim 같은 꼴이 나지 않을까?에 대한 두려움은 [CNCF 졸업 프로젝트](https://www.cncf.io/projects/containerd/)라는 점이 해소해준다. 앞으로도 Dockerfile이나 도커는 쿠버네티스와 함께 쓰일 것으로 예상되기에 이 역사를 아는 것은 꽤 유효할 거 같다.

- containerd는, 도커에도 있지만 독립적인 컴포넌트, 그리고 CRI를 준수하는 컨테이너 런타임으로써 kubelet이 사용할 수 있다.

---

## 참고
- https://www.oreilly.com/library/view/docker-deep-dive/9781800565135/
- https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-removal-affects-you/
- https://kubernetes.io/ko/blog/2020/12/02/dont-panic-kubernetes-and-docker/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://ikcoo.tistory.com/189
