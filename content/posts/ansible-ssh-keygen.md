---
title: "Ansible로 모든 VM을 SSH 연결하기"
date: 2022-03-08T23:27:10+09:00
tags:
- ansible
---

언제나 그렇듯 삽질 후 깔끔하게 정리한다. VM끼리 ssh로 연결하고 싶었다. 보통은 쉘 스크립트로 프로비저닝했다. 그런데 ansible로 해보니 몇가지 어려운 점이 있었다:
- authorized_keys를 만들 때 각 VM에서 만든 공개키를 **'모아야'** 한다.
- SSH key를 **'새로 만든 VM만'** authorized_keys를 갱신해야 한다.

SSH key gen은 [Self-signed Certificate](/posts/self-signed-certificate.md)을 해보며 잘 이해해서 ansible-galaxy에 있는 [community.crypto.openssh_keypair](https://docs.ansible.com/ansible/latest/collections/community/crypto/openssh_keypair_module.html)로 대체했다.

## Special varibables

VM을 up하고 아무도 SSH 키 쌍이 없는 상태에선 모두 키를 만들고 공개키를 모아서 모두의 authorized_keys 파일에 써주어야 한다. 만든 공개키는 `register`를 통해 변수로 받을 수 있었지만, 이를 다른 VM으로 어떻게 넘길지 떠오르지 않았다. SSH가 안되는 마당에 VM끼리 소통할 수 있나?

문제를 추상화하여 [질문](https://stackoverflow.com/questions/71391592/ansible-reduce-to-the-variable-from-registers-of-each-host)을 올렸다. 답은 몇가지 특별한 변수들에 담겨 있었다(마법을 찾고 있었는데 정말로 '마법 변수'였다):
- `hostvars`: 모든 인벤토리 호스트의 변수를 담고 있다.
- `ansible_play_hosts`: 현재 play 중인 호스트명 리스트.

`community.crypto.openssh_keypair` 모듈의 실행 결과를 변수에 등록(register) 했기 때문에, 이 태스크 실행 이후엔 `hostvars` 변수에 담겨 있다. `hostvars`는 호스트명을 키로 하는 해시테이블이기 때문에 [`extract`](https://docs.ansible.com/ansible/devel/user_guide/playbooks_filters.html#selecting-values-from-arrays-or-hashtables)를 통해 매핑한다(Jinja2 builtin이 아닌 ansible의 구현이다):

```yaml
 - name: Generate an OpenSSH keypair with the default values (4096 bits, rsa)
   community.crypto.openssh_keypair:
     state: present
     path: /tmp/id_ssh_rsa
   register: ssh_result

- name: Collect the pubkeys
  set_fact:
    pubkeys: "{{ ansible_play_hosts | map('extract', hostvars, ['ssh_result', 'public_keys']) }}"
```

- [public_key는 community.crypto.openssh_keypair의 반환 값 중 하나이다](https://docs.ansible.com/ansible/latest/collections/community/crypto/openssh_keypair_module.html).
- map의 세번짜 인자처럼 nested attribute에 접근할 수 있다.

뒤에도 또 나오지만, `ansible_play_hosts(_all) | map('extact', hostvars, ...)` 이것이 정말 많이 쓰는 패턴 같다. Ansible은 인벤토리 호스트마다 독립적으로 실행하지만 전체 인벤토리에 대해 변수를 접근하고 싶을 때, 즉 각 호스트 변수들을 모을 때 이 메타데이터를 사용하는것 같다.

## 키 만든 호스트만 거르기
만약 처음 실행하는게 아니라 VM을 새로 추가한다면 새 VM에만 SSH 키 쌍을 만들고 이 공개키만 모든 VM의 authorized_keys에 추가해주어야 한다. 새로 만든 SSH 공개키와 그 호스트만 알면 될 것이다. 호스트로 식별해 authorized_keys 파일에 [blockinfile](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/blockinfile_module.html) 처리하면 된다.

이번에도 호스트 변수를 사용한다. 나는 새로 키를 만들 기준으로 '호스트 $HOME/.ssh/ 에 키 파일이 있는지'로 판단했다. 이 여부 리스트와 ansible_play_hosts를 elementwise AND-ing했다:

```yaml
- name: Check the ssh key exists
  stat:
    path: /home/vagrant/.ssh/id_ssh_rsa
  register: key_stat

- name: Collect the hosts generating the new key
  set_fact:
    new_key_hosts: "{{ ansible_play_hosts | map('extract', hostvars, ['key_stat', 'stat', 'exists']) | zip(ansible_play_hosts) | rejectattr('0') | map(attribute='1') }}"
```

- ansible_play_hosts가 `["vm1", "vm2", "vm3"]` 이고 이 중 vm2에 `/home/vagrant/.ssh/id_ssh_rsa` 가 없다면, 첫번째 파이프(`{{ ansible_play_hosts | map('extract', hostvars, ['key_stat', 'stat', 'exists']) }}`)까지의 결과는 `[true, false, true]`이다.
- 이걸 다시 ansible_play_hosts와 zip하여 list의 list를 만들고 true인 것만 거른다(AND-ing).

## 디버거

위 설명처럼 뚝딱하진 않았다([이 글](https://stackoverflow.com/questions/63619004/ansible-set-fact-for-all-hosts-in-play-when-only-one-host-has-a-met-condition)도 참고했다. Vladimir Botka 짱이다).

맨 처음엔 현재 태스크 when 조건에 만족해서 skip 안하고 실행 중인 호스트만 들고 있는 또 다른 '마법 변수'가 있는지 찾아 보았다(ansible_running_hosts?). 이 과정에서 [디버거](https://docs.ansible.com/ansible/latest/user_guide/playbooks_debugger.html)의 존재를 알게 되었다. 예시는 `debug` 모듈에 붙였지만, 아무 태스크나 `debugger: always`를 추가하면 쉽게 사용할 수 있다:

```yaml
- name: debug
  debug:
    var: pubkeysright
  debugger: always
```

check mode로 실행해도 디버거는 걸린다. 여기선 `task_vars` 변수부터 접근하면 특별한 변수를 포함한 변수들에 접근할 수 있다. "hosts"가 있는 변수 이름을 보았지만 기대한 마법 변수는 없었다(내가 못 찾은걸수도...):
```sh
[cluster1-master1] TASK: ssh : debug (debug)> p list(filter(lambda x: "hosts" in x, task_vars.keys()))
['new_key_hosts',
 'new_pubkey_and_hosts',
 'ansible_play_hosts_all',
 'ansible_play_hosts',
 'play_hosts',
 'ansible_current_hosts',
 'ansible_failed_hosts']
```


## file state=present?

이 작업을 하며 알게 된 또 다른 소소한 사실인데, [file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) 모듈에 [웬만하면 있을거라는 state=present](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#always-mention-the-state)가 없다. 그래서 state=touch로 하면 playbook 실행마다 불필요하게 change가 생겼다. present와 비슷한 동작을 하도록 [access_time과 modification_time에 preserve를 선언해준다](https://stackoverflow.com/questions/63619004/ansible-set-fact-for-all-hosts-in-play-when-only-one-host-has-a-met-condition):

```yaml
- name: Touch the ssh config file
  file:
    state: touch
    path: /home/vagrant/.ssh/config
    mode: "0644"
    access_time: preserve
    modification_time: preserve
```

그리하여 완성한 최종 롤 태스크는 [이렇다](https://github.com/flavono123/kubernetes-the-hard-way/blob/main/provisioning/roles/ssh/tasks/configure.yaml).


---

## 참고
- https://stackoverflow.com/questions/71391592/ansible-reduce-to-the-variable-from-registers-of-each-host
- https://stackoverflow.com/questions/63619004/ansible-set-fact-for-all-hosts-in-play-when-only-one-host-has-a-met-condition
- https://zwischenzugs.com/2021/08/27/five-ansible-techniques-i-wish-id-known-earlier/
- https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html#magic-variables
- https://stackoverflow.com/questions/63619004/ansible-set-fact-for-all-hosts-in-play-when-only-one-host-has-a-met-condition

