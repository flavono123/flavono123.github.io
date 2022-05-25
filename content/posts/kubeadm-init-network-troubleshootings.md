---
title: "kubeadm으로 쿠버네티스 클러스터 설치와 문제 해결"
date: 2022-05-24T17:24:19+09:00
tags:
- kubernetes
- network
---

쿠버네티스 클러스터를 turn up하고 그 과정에서 생긴 문제점과 해결 과정을 정리한다.
코드는 이 [레포](https://github.com/flavono123/kubernetes-the-hard-way)에 있고 글을 쓰는 시점에 마지막 커밋은 [0188346](https://github.com/flavono123/kubernetes-the-hard-way/commit/0188346)이다.

노드엔 Vagrant, 클러스터 구성 도구는 kubeadm 그리고 CNI는 [Calico](https://projectcalico.docs.tigera.io/about/about-calico)를 사용했다. Vagrant 컴퓨팅 리소스를 보면 알 수 있듯, 로컬에서 연습용으로 사용할 클러스터이다.

provisioning 디렉토리 아래의 Ansible 코드 위주로 turnup 과정을 먼저 설명한다. 그리고 대부분 네트워크 문제였던 문제 해결 과정을 후술한다.

### Prerequisuite

클러스터는 총 3개 노드 모두 ubuntu 20.04이고, 각각 controlplane 1개와 worker2 개이다. CK\* 시험 볼 때나(worker가 1개인 경우도 있다), 시험 준비하며 연습하던 기본적인 구성으로 했다.

내부망(192.168.1.0/24)을 만들어 정적 IP를 할당하고 [DNS 등록](https://github.com/flavono123/kubernetes-the-hard-way/blob/main/provisioning/roles/dns/tasks/configure.yaml)과 [SSH 연결](/posts/ansible-ssh-keygen/)이 되도록 해주었다.

## Cluster turn-up

앞의 기본적인 프로비저닝을 제외하고, kubeadm로 쿠버네티스 클러스터 설치하는 것에 집중하여 설명한다. 또 CNI까지 설치해야 동작이 가능하므로 그것도 포함한다. 코드에선 Ansible [kubernetes role(provisioning/roles/kubernetes)](https://github.com/flavono123/kubernetes-the-hard-way/tree/main/provisioning/roles/kubernetes)이다.

기본적으론 [쿠버네티스 문서 가이드](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)를 따라했고, [CKS 강의하신 김선생님](https://wuestkamp.com/)의 [코드](https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/cluster-setup/latest/install_master.sh)도 참고했다.

컨테이너 런타임인 containerd 설치는 [이전 포스트](http://localhost:1313/posts/containerd-as-kubernetes-cri/)로 대체한다.

kubelet이 잘 동작하기 위해 시스템의 swap을 비활성화해야 한다:
```yaml
# tasks/swap.yaml
- name: Disable swap
  become: yes
  command: swapoff -a

- name: Remove swapfile from /etc/fstab
  become: yes
  mount:
    state: absent
    name: swap
    fstype: swp
```

kubeadm으로 설치하기 위해 kubeadm, kubectl, kubelet 세개의 패키지를 받는다:
```yaml
# tasks/kube_install.yaml
- name: Add Google Cloud GPG key
  become: yes
  apt_key:
    state: present
    keyring: /usr/share/keyrings/kubernetes-archive-keyring.gpg
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg

- name: Add the Kubernetes APT repository
  become: yes
  apt_repository:
    state: present
    update_cache: yes
    filename: kubernetes.list
    repo: "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main"

- name: Install kubeadm, kubelet and kubectl
  become: yes
  apt:
    state: present
    name:
    - "kubeadm={{ kube_version }}-00"
    - "kubelet={{ kube_version }}-00"
    - "kubectl={{ kube_version }}-00"
```

여기서 의아한 부분은 패키지 배포판 이름이 focal(20.04)가 아닌 **xenial**(16.04)라는 점이다. 이는 가이드 문서에도 그렇게 나와 있고, 실제로 [구글의 패키지 배포판 경로](https://packages.cloud.google.com/apt/dists)를 보아도 그렇다. 히스토리까지 찾진 못했지만, 아마 xenial 이후의 버전은 다 호환이 되도록 하나에서 관리하는 것 같다(나는 focal로 바꾸어 레포를 등록하려다가 안됐었다).


kubeadm으로 클러스터를 구성하는 것은:
- controlplane에서 옵션과 함께 초기화
- worker 노드를 join(controlplane에서 join 토큰 발급)

순서로 진행된다. kubeadm.yaml 태스크는 크게 set/reset 블록으로 나누었다. Set 부분을 먼저 설명한다:
```yaml
# tasks/kubeadm.yaml
- name: Set kubeadm cluster
  become: yes
  block:
  - name: Collect kuebadm init options
    set_fact:
      kubeadm_init_option: "--{{ item.key }}={{ item.value }}"
    with_items: "{{ kubeadm_init_options | dict2items }}"
    register: kubeadm_init_option_list
    when: "'controlplane' in group_names"

  - name: Reduce kubeadm init options to a string
    set_fact:
      kubeadm_init_option_str: "{{ kubeadm_init_option_list.results | map(attribute='ansible_facts.kubeadm_init_option') | join(' ') }}"
    when: "'controlplane' in group_names"

  - name: Initialize kubeadm in controlplane
    command: "kubeadm init {{ kubeadm_init_option_str }}"
    register: kubeadm_init_result
    until: kubeadm_init_result.stdout.find("Your Kubernetes control-plane has initialized successfully!")
    delay: 120
    retries: 1
    ignore_errors: "{{ ansible_check_mode }}"
    when: "'controlplane' in group_names"

  - name: Make directory for kube-config
    file:
      state: directory
      path: $HOME/.kube

  - name: Copy admin kube-config for root
    command: cp /etc/kubernetes/admin.conf $HOME/.kube/config
    when: "'controlplane' in group_names"

  - name: Create token for nodes
    command: "kubeadm token create --print-join-command --ttl 0"
    register: join_command
    delegate_to: "{{ groups.controlplane[0] }}"
    when: "'nodes' in group_names"

  - name: Join nodes to the cluster
    command: "{{ hostvars[inventory_hostname].join_command.stdout | trim }}"
    when:
    - "'nodes' in group_names"
    - join_command.stdout != ""
  when:
  - kubeadm_reset is not defined
  tags:
  - kubernetes/kubeadm/set
```

실제 Ansible tasks엔 Ansible 변수를 kubeadm 명령 옵션으로 만들기 위해 문자열 조작하거나, 노드의 root 사용자가 쿠버네티스 어드민 자격으로 kubectl을 실행할 수 있게 설정을 복사하는 것(Copy admin kube-config for root)이 추가로 있다.

controlplane에서 `kubeadm init` 시 옵션이 중요한데 그에 해당하는 변수인 `kubeadm_init_options`를 가져와 설명한다:
```yaml
# defaults/main.yaml
controlplane_node_hostname: "{{ groups['controlplane'][0] }}"
controlplane_node_ip: "{{ hostvars[controlplane_node_hostname]['ansible_facts']['enp0s8']['ipv4']['address'] }}"

kube_apiserver_port: 6443

kubeadm_init_options:
  apiserver-advertise-address: "{{ controlplane_node_ip }}"
  control-plane-endpoint: "{{ controlplane_node_ip }}:{{ kube_apiserver_port }}"
  apiserver-cert-extra-sans: "{{ controlplane_node_ip }}"
  pod-network-cidr: 172.16.0.0/16
```

`controlplane_node_ip`는 [Ansible magic variables](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)를 이용해 얻어왔는데, 결국 192.168.1.2 즉, controlplane 노드의 내부망 IP이다.

[가이드의 kubeadm init 옵션 설명 부분](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node)과 함께 보면,
- `--control-plane-endpoint` - controlplane HA(multi-controlplane) 구성은 아니지만, 하나 있는 controlplane의 노드의 IP 주소를 적음
- `--apiserver-cert-extra-sans` - Apiserver(=controlplane 노드)의 인증을 위한 SAN
- `--pod-network-cidr` - 필수로 써주어야 CNI 애드온이 정상적으로 설치된다. **노드(192.168.1.0/24)나 쿠버네티스 서비스(``--service-cidr` default 10.96.0.0/12) 네트워크와 겹치지 않도록 해주어야 한다.** 서브넷의 크기도 중요한데 이는 후술한다.
- `--apiserver-advertise-address` - 가이드엔 선택 사항으로 쓰여 있지만 이 설치에선 **반드시 적어야 한다. Vagrant로 ubuntu VM을 만들면 호스트와 연결하기 위한 네트워크 인터페이스(enp0s3)가 있기 때문에, 구성한 내부망 IP를 지정해 주어야 한다**

노드에서 네트워크 인터페이스를 확인해보면 기본으로 enp0s3와 클러스터 노드 라우팅을 위한 enp0s8(192.168.1.0/24)을 확인할 수 있다:
```sh
root@cluster1-master1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:10:bc:02:65:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 80280sec preferred_lft 80280sec
    inet6 fe80::10:bcff:fe02:654d/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:5d:be:98 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.2/24 brd 192.168.1.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5d:be98/64 scope link
       valid_lft forever preferred_lft forever

```

반복적으로 클러스터를 turn up 해보기 위해 reset하는 tasks도 만들었다:
```yaml
# tasks/kubeadm.yaml
- name: Reset kubeadm cluster
  become: yes
  block:
  - name: Drain nodes
    command: "kubectl drain {{ item }} --delete-emptydir-data --force --ignore-daemonsets"
    with_items: "{{ groups.nodes }}"
    ignore_errors: yes
    when: "'controlplane' in group_names"

  - name: Reset nodes
    command: "kubeadm reset -f"
    when: "'nodes' in group_names"

  - name: Reset iptables rules
    shell: "iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X"

  - name: Delete nodes
    command: "kubectl delete node {{ item }}"
    with_items: "{{ groups.nodes }}"
    ignore_errors: yes
    when: "'controlplane' in group_names"

  - name: Reset controlplane
    command: "kubeadm reset -f"
    when:
    - "'controlplane' in group_names"
  when:
  - kubeadm_reset is defined
  - kubeadm_reset
  tags:
  - kubernetes/kubeadm/reset
```

Reset은 다음 순서로 진행된다:
- worker 노드 drain
- worker 노드 reset
- 모든 노드의 iptables 초기화
- worker 노드 삭제(delete)
- controlplane 노드 reset

처음엔 노드, Ansible 호스트, 의 상태에 따라 reset, set을 실행하고 싶었으나 적합하지 않은 것 같아 `kubeadm_reset`이란 플래그 변수와 태그로 reset/set tasks를 실행할지 분기했다. 이는 파드 네트워크 애드온인 Calico 설치할 때도 사용했다.

여기까지 설치를 마치면 coredns 파드가 pending인 상태일 것이다. Calico를 설치해준다.

여러 Calico 설치 방법 중에서 [Self-managed on-premise](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)를 따라했다:

```yaml
# tasks/calico.yaml
- name: Set the Calico CNI
  become: yes
  block:
  - name: Create the directory for Calico
    become: yes
    file:
      state: directory
      path: $HOME/calico

  - name: Get the Calico operator definition
    become: yes
    get_url:
      url: https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
      dest: $HOME/calico/tigera-operator.yaml

  - name: Get the Calico custom resource definition
    become: yes
    uri:
      url: https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml
      return_content: yes
    register: calico_custom_resource_response

  - name: Copy the Calico custom resource definition
    become: yes
    copy:
      content: "{{ calico_custom_resource_response.content | regex_replace('192\\.168\\.0\\.0\\/16', '172.16.0.0/16') | to_yaml(indent=2, width=1337) | replace('\\n', '\n') | replace('\"', '') }}"
      dest: $HOME/calico/custom-resources.yaml
    ignore_errors: "{{ ansible_check_mode }}"

  - name: Create the Calico operaters and resoureces
    become: yes
    command: "kubectl apply -f $HOME/calico/{{ item }}"
    with_items:
    - tigera-operator.yaml
    - custom-resources.yaml

  - name: Get the calicoctl
    become: yes
    get_url:
      url: https://github.com/projectcalico/calico/releases/download/v3.23.1/calicoctl-linux-amd64
      dest: /usr/local/bin/calicoctl
      mode: '0755'
  tags:
  - kubernetes/calico/set
  when:
  - "'controlplane' is in group_names"
  - calico_reset is not defined
```
가이드 설명대로:
- operator를 설치하고
- custom resources(Installation, APIServer)를 커스터마이즈한 후 설치한다
  - Installation의 `spec.calicoNetwork.ipPools[0].cidir`를 kubeadm init 시 `pod-network-cidr`와 같게 172.16.0.0/16으로 한다

추가로 Calico 제어를 위한 calicoctl도 설치한다. CNI가 잘 설치되었는지 확인하는 몇가지 명령을 실행해본다(설명과 공부는 다음으로 미룬다. DOIK라는 쿠버네티스 데이터베이스 오퍼레이터 스터디에서 명령을 알게 되었다):
```sh
root@cluster1-master1:~# calicoctl get ippool -owide
NAME                  CIDR            NAT    IPIPMODE   VXLANMODE     DISABLED   DISABLEBGPEXPORT   SELECTOR
default-ipv4-ippool   172.16.0.0/16   true   Never      CrossSubnet   false      false              all()

root@cluster1-master1:~# calicoctl ipam show --show-blocks
+----------+------------------+-----------+------------+--------------+
| GROUPING |       CIDR       | IPS TOTAL | IPS IN USE |   IPS FREE   |
+----------+------------------+-----------+------------+--------------+
| IP Pool  | 172.16.0.0/16    |     65536 | 8 (0%)     | 65528 (100%) |
| Block    | 172.16.222.64/26 |        64 | 4 (6%)     | 60 (94%)     |
| Block    | 172.16.25.192/26 |        64 | 2 (3%)     | 62 (97%)     |
| Block    | 172.16.40.64/26  |        64 | 2 (3%)     | 62 (97%)     |
+----------+------------------+-----------+------------+--------------+
root@cluster1-master1:~# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.1.4  | node-to-node mesh | up    | 09:20:29 | Established |
| 192.168.1.3  | node-to-node mesh | up    | 09:20:33 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

```

worker 노드의 연결 상태나 파드용 서브넷 할당이 잘 된걸 볼 수 있다.

Reset은 이미 설치되어 있단 가정하에 위 리소스들을 삭제해준다:
```yaml
# tasks/calico.yaml
- name: Delete the Calico operators and resources
  become: yes
  command: "kubectl delete -f $HOME/calico/{{ item }}"
  with_items:
  - custom-resources.yaml
  - tigera-operator.yaml
  tags:
  - kubernetes/calico/reset
  when:
  - "'controlplane' is in group_names"
  - calico_reset is defined
  - calico_reset
```

이로써 쿠버네티스 구성은 끝났다. 추가로 [root 유저로 bash 사용을 쉽게 하기 위해 설정 몇개를 추가해줬다.](https://github.com/flavono123/kubernetes-the-hard-way/blob/main/provisioning/roles/kubernetes/tasks/bash_kubectl.yaml)

## Troubleshooting

실제로 위 설명처럼 클러스터 turn up을 한번에 뚝딱하진 못했다. 중간 중간 나사빠진 부분들이 있었는데, 그 때 마다 만난 오류와 해결방법을 설명한다.

**client: unable to create k8s client: Get \"https://10.96.0.1:443/api/v1/namespaces/kube-system\": dial tcp 10.96.0.1:443: connect: connection refused" subsys=daemon**

처음으로 애드온을 설치하고 나서, operator가 계속 재시작하여 보니 위와 같은 오류 로그가 있었다.

원인은 크게 두가지였다.

첫번째는 내부망 IP를 잘못 지정해서였다. 위에 쓴 IP와 달리 맨 처음엔 192.168.1.2, 192.168.1.3, 192.168.1.4가 아니라 **192.168.1.1**, 192.168.1.2, 192.168.1.3으로 했다. 첫번째 IP는 라우터가 사용하는 IP이기 때문에 잘 동작하지 않았던거 같다:
```sh
==> cluster1-master1: You assigned a static IP ending in ".1" to this machine.
==> cluster1-master1: This is very often used by the router and can cause the
==> cluster1-master1: network to not work properly. If the network doesn't work
==> cluster1-master1: properly, try changing this IP.
```
하지만 192.168.1.1를 내가 라우터용 IP로 지정한 적이 없기 때문에, 확실히 이 IP를 라우터가 사용하는지 모르겠다. 또 VM에서 192.168.1.0/24의 라우터 IP를 직접 보지도 못했다. 학교에서 배웠던거 같은 지식과 몇몇 그렇게 사용하는 사례([AWS VPC](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)), 그리고 Vagrant가 띄워주는 경고 문구를 보고 유추했다. 조금 찜찜하지만 192.168.1.1가 아닌 IP를 사용하여 문제는 해결된다.

두번째 이유는 kube-apiserver의 IP 주소가 잘못 되어서 그렇다. 앞서 말한듯 Vagrant로 시작한 VM은 호스트 머신에서 접속할 수 있도록 네트워크 인터페이스가 생긴다. 옵션이 없다면 kube-apiserver는 이 기본 인터페이스의 IP 주소를 다른 worker 노드에서 접근할 수 있도록 자신의 IP 주소로 사용하게 된다. 옵션 이름은 `advertise-address`이고 이는 kubeadm init 시 `apiserver-advertise-address`로 초기화 가능하다.

즉, kubeadm init 시 `--apiserver-advertise-address=192.168.1.2`를 추가함으로 해결할 수 있었다. 잘 적용됐는지 확인은 kube-apiserver 매니페스토 파일과 생성된 kubernetes 서비스/엔드포인트로 확인할 수 있다:
```sh
root@cluster1-master1:~# grep advertise -C3 /etc/kubernetes/manifests/kube-apiserver.yaml
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.1.2:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
--
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.1.2
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
root@cluster1-master1:~# k get svc,ep
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   16h

NAME                   ENDPOINTS          AGE
endpoints/kubernetes   192.168.1.2:6443   16h
```


또 worker 노드에서, 인증이 필요 없는 버전 경로로, kube-apiserver에 요청하여 연결이 잘 되는지 확인할 수 있다:
```sh
root@cluster1-worker1:~# wget --no-check-certificate https://192.168.1.2:6443/version -O- -q
{
  "major": "1",
  "minor": "23",
  "gitVersion": "v1.23.6",
  "gitCommit": "ad3338546da947756e8a88aa6822e9c11e7eac22",
  "gitTreeState": "clean",
  "buildDate": "2022-04-14T08:43:11Z",
  "goVersion": "go1.17.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

**client: unable to create k8s client: Get \"https://10.96.0.1:443/api/v1/namespaces/kube-system\": dial tcp 10.96.0.1:443: i/o timeout" subsys=daemon**

문제를 해결하니 다른 문제가 생겼다. 여전히 operator는 kube-apiserver 연결이 실패하는데 이번엔 이유가 i/o timeout이다.

이 문제는 해결과정이 좀 독특했다. 먼저 calico i/o timeout 로 구글링 하여 [이 스레드](https://discuss.projectcalico.tigera.io/t/getting-the-dial-tcp-10-96-0-1-i-o-timeout-issues/38)를 발견했다. 답변에 파드 CIDR를 잘 설정해 해결했다는 것을 보고, '난 파드 CIDR 지정한적이 없는데?' 싶었다. 그렇다. kubeadm init 시 필수로 주어야 하는 옵션 `pod-network-cidr`를 쓰지 않았다.

노드, 서비스 그리고 파드 각 네트워크 CIDR가 겹치지 않게 설정하라는 조언을 참고해서, `pod-network-cidr`를 처음엔 172.16.0.0/24로, 정말 겹치지 않게만, 설정했다. Calico custom resource Installation의 `spec.calicoNetwork.ipPools[0].blockSize|cidr`도 각각 24와 172.16.0.0/24로 맞추어 주었다.

이럴 경우 모든 worker 노드에서 하나의 서브넷(172.16.0.0/24)를 공유하게 된다. [kubelet엔 worker 노드에서 생성 할 수 있는 파드의 최대 개수를 110개로 hard restriction 하고 있다(max-pods 옵션)](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/). [GKE 문서엔, 이에 두배에 IP가 노드 당 할당할 수 있다](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr?hl=ko)고 하여(아마 파드마다 서비스를 노출하는 경우를 생각한거 같다), 이 때 노드당 서브넷을 24 즉 256개 IP를 가용할 수 있게 설정한다.

그런데 pod-network-cidr도 24의 서브넷으로 주었다. woker 노드마다 서브네팅 할 순 없을 것이다. 하지만 적은 개수의 파드 생성이나 통신하는데엔 문제가 없었다. 이는 Calico의 ippool 설정을 Cross-Subnet(default)으로 했기 때문에 그런것 같다.
```sh
root@cluster1-master1:~# calicoctl ipam show --show-blocks
+----------+---------------+-----------+------------+-----------+
| GROUPING |     CIDR      | IPS TOTAL | IPS IN USE | IPS FREE  |
+----------+---------------+-----------+------------+-----------+
| IP Pool  | 172.16.0.0/24 |       256 | 8 (3%)     | 248 (97%) |
| Block    | 172.16.0.0/24 |       256 | 8 (3%)     | 248 (97%) |
+----------+---------------+-----------+------------+-----------+
```

IPAM 블록이 하나로 되어 있다. 일반적인 설정이 아닌거 같고 나중에 문제가 있을 수도 있겠다 판단했다. 이 부분은 Calico를 더 스터디하며 알아 보아야겠다 생각하고 파드 cidr는 더 넉넉하게 16, 블록 cidr는 기본 값인 26으로 하여 블록이 노드 당 서브네팅 될 수 있도록 설정을 바꾸었다.


## 여담
원랜 CNI를 Calico가 아닌 [Cilium](https://cilium.io/)으로 진행하고 있었다. Cilium의 [네트워크 정책 편집기](https://editor.cilium.io/)가 반짝반짝 이뻐 보여서 둘러보니, 네트워크 흐름 CLI 디버깅 툴이나 GUI인 [Hubble](https://github.com/cilium/hubble)의 기능이 좋아 보여 이걸로 설치하기로 정했다. 반면에 Calico는 언뜻 봤을 때 홈페이지가 그닥 세련돼 보이지 않아(?) 선택하지 않았다. 그러다 위 문제 부분들에서 막혀 진행이 안 되고 있었다.

그러다 Cilium 보단 Calico가 더 참고하기 좋은 리소스가 많다는 것을 알게 됐다([Certifications](https://www.tigera.io/lp/calico-certification/), [블로그](https://gasidaseo.notion.site/gasidaseo/CloudNet-Blog-c9dfa44a27ff431dafdd2edacc8a1863)). 결정적으론 위에 언급한 disquss 스레드가 문제를 해결하고 앞으로 나아가는데 도움이 되어 Calico를 선택하게 됐다.


## 정리
클러스터 turn up 중 있던 문제와 해결 위주로 정리한다:

- kube\*의 Apt(ubuntu) 패키지 저장소 배포판 이름은 xenial 이후엔 모두 **xenial**이다(focal, bionic은 없다)
- Vagrant에선 기본 네트워크 인터페이스가 이미 있으므로, 쿠버네티스 노드를 구성할 때 내부망 IP를 찾을 수 있도록 옵션을 주어야 한다
  - e.g. kub-apiserver의 `advertise-address`(kubeadm init 시 `apiserver-advertise-address`)
- CNI operator는 kube-apiserver 즉, kubernetes 서비스와 통신 가능해야 한다.
  - 노드 레벨에서 연결 가능한지 worker 노드에서 controlplane으로 직접 /version 요청을 해볼 것(TLS 인증 없이 가능)
- kubeadm init 시 **`pod-network-cidr`** 는 꼭 지정해주어야 하며, 노드나 서비스 CIDR와 겹치지 않도록 해야 한다.


---

## 참고
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/cluster-setup/latest/install_master.sh
- https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises
