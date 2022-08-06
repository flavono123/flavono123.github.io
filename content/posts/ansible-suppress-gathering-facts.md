---
title: "Ansible Gathering Facts 하지 않기"
date: 2022-08-06T10:42:32+09:00
tags:
- ansible
---

Ansible playbook을 실행하면 항상 [Gathering Facts](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/gather_facts_module.html)를 한다:
```sh
❯ ansible-playbook playbook.yml [options...]
PLAY [어쩔 플레이북] ***************************************************************************

task [Gathering Facts] *************************************************************************
ok: [host_a]

TASK [task1 : 어쩔 태스크] *********************************************************************
ok: [host_a]

TASK [task1 : 저쩔 태스크] *********************************************************************
ok: [host_a]
...

```

하지만 수집한 facts를 쓰지 않아 불필요할 때가 많다. 또 playbook이, 여러 roles를 로드하는 식으로, 여러 play를 실행한다면 Gathering Facts가 각각 실행한다. 이 때 특정 태스크만 태그로 필터하여 play하고 싶다면 모든 play에서의 Gathering Facts는 확실히 불필요하다:

```sh
❯ ansible-playbook playbook.yml [options...] --tags="some_tasks"
...
PLAY [어쩔 롤] *********************************************************************************

TASK [Gathering Facts] *************************************************************************
ok: [host_a]

PLAY [저쩔 롤] *********************************************************************************

TASK [Gathering Facts] *************************************************************************
ok: [host_a]

PLAY [태그 롤] *********************************************************************************

TASK [Gathering Facts] *************************************************************************
ok: [host_a]

TASK [tagged_role : Run a Tagged Task] *********************************************************
ok: [host_a]
...
```

불필요한 Gathering Facts는 시간도 소모하고 결과 출력도 한눈에 보기 어렵게 한다. 그래서 꼭 하고 싶지 않은 Gathering Facts를 안하는 방법이 있다:
```sh
❯ ANSIBLE_GATHERING=explicit ansible-playbook playbook.yml [options...] --tags="some_tasks"
...
PLAY [어쩔 롤] *********************************************************************************

PLAY [저쩔 롤] *********************************************************************************

PLAY [태그 롤] *********************************************************************************

TASK [tagged_role : Run a Tagged Task] *********************************************************
ok: [host_a]
...
```

**ANSIBLE_GATHERING 환경 변수에 `explicit`**을 써주자. [이 환경 변수는 Gathering Facts에 대한 정책을 구성할 수 있다.](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#envvar-ANSIBLE_GATHERING) 구성 옵션 [DEFAULT_GATHERING](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-gathering)을 보면 Gathering Facts에 대한 정책을 확인할 수 있다:
- `implicit`(default): play에 `gather_facts: False`로 되어 있지 않으면 facts 수집
- `explicit`: implicit과 반대로 명시하지 않으면(`gather_facts: True`) facts를 수집하지 않는다
- `smart`: play에서 중복되는 호스트의 facts를 최초 한번만 수집한다

나는 현재 facts를 잘 활용하고 있지 않지만, 사용한다면 smart 정책도 좋은 옵션이다:
```sh
❯ ANSIBLE_GATHERING=smart ansible-playbook playbook.yml [options...] --tags="some_tasks"
...
PLAY [어쩔 롤] *********************************************************************************

TASK [Gathering Facts] *************************************************************************
ok: [host_a]

PLAY [저쩔 롤] *********************************************************************************

PLAY [태그 롤] *********************************************************************************

TASK [tagged_role : Run a Tagged Task] *********************************************************
ok: [host_a]
...
```

`smart` 정책은 모든 play에서 호스트다마 중복 없이 facts를 한번씩만 수집한다.

## 정리
- ANSIBLE_GATHERING 환경변수는 Gathering Facts 태스크 실행 정책을 제어한다.
  - Playbook에서 facts를 사용하지 않는다면 `explicit`으로 하자(default는 `implicit`).
  - 사용한다면 중복을 줄이기 위해 `smart`를 사용해보자.

---

## 참고
- https://stackoverflow.com/questions/72094835/disable-ansible-gather-facts-on-the-command-line
- https://docs.ansible.com/ansible/latest/reference_appendices/config.html#
