---
title: "여러 호스트에 간단한 명령하기(Ansible vs. PSSH)"
date: 2022-03-24T11:46:14+09:00
tags:
- ansible
- pssh
---

이 글은 프로비저닝 툴로써 Ansible과 PSSH를 비교하는 글이 아니다. 간단한 명령을 할 때 Ansible과 PSSH 둘 다 사용해보고 비교하는 글이다.

나는 프로비저닝 툴로썬 Ansible을 이미 쓰고 있는데, 막상 간단한 명령을 하려니 "PSSH를 쓰는게 더 쉬운가?" 라는 생각이 들었다. 상황은 간단하게 각 호스트의 공인 IP를 알아내는 것이다. 각 호스트에서 `curl ifconfig.me` 를 실행하고 출력을 확인해야 한다.

## PSSH

```sh
❯ pssh --version
2.3.1
```

#### 호스트 인자로 인벤토리 파일 활용

앞서 말한 듯 Ansible을 사용하고 있는 상황이기 때문에 PSSH의 호스트 인자로 Ansible의 인벤토리 INI 파일을 쓸 것이다. 인벤토리 파일을 정의하는 것 나름이겠지만, 어쨌든 ssh 접속할 수 있는 호스트명을 포함하고 있을 것이다(어쩌면 호스트변수 yaml 파일에 있을 것이다). 나의 경우는 INI 파일 컨벤션이 다음과 같다:

```
<type>_host1 ansible_host=host1 [...]
...

[<type>_servers]
<type>_host1
...
```

- `<type>_host1`: Ansible 호스트명(hosts)을 용도에 따른 시맨틱 이름인, 서버 타입(`<type>`)과 실제 ssh 접속에 사용하는 hostname을 `_` 로 잇는다
- `[<type>_servers`]: 그룹명은 간단하게 타입 뒤에 `_servers`를 붙인다(e.g. widget_servers)
- `ansible_host=host1 [...]`: `ansible_`로 시작하는 몇몇 호스트 변수는 INI 파일에 그대로 쓰고 있다(왜 이건 yaml에 안썼을까 싶다 .. 🤔)

이런 조건에서 나는 `<type>_host1` 에서 호스트명을 파싱하며 PSSH 호스트 인자를 만들기로 했다.

```sh
❯ cat <invetory_file> | grep -vE '^\[' |\
  awk 'NF {sub("<type>_", "", $1); print $1}' |\
  sort | uniq |\
  xargs -I{} echo -H {} |\
  tr '\n' ' '
-H host1 -H host2 -H host3
```

- `grep -vE '^\['`: 그룹 정의 라인을 제외한다
- `awk 'NF {sub("<type>_", "", $1); print $1}'`: 공백 줄을 제거하고, Ansible 호스트명에서 `<type>_`을 제거하여 SSH 호스트명만 남긴다
- `sort | uniq`: 중복을 제거한다(위에 호스트 변수 정의 때문에 중복이 있음)
- `xargs -I{} echo -H {}`: 호스트명을 "-H <host>"에 매핑하고
- `tr '\n' ' '`: 각 라인을 공백으로 합친다

`awk`나 map join(`xargs`, `tr`)를 더 잘 쓰면 명령을 줄일 수 있을거 같다.

또는 `ansible_host` 가 호스트 변수 yaml 파일에 정의되어 있다면, yaml만 파싱하면 되기 때문에 더욱 쉬울 것이다. `<type>`을 widget이라 하고 인벤토리 파일 전체를 yaml로 쓰면 다음과 같을 것이고:
```yaml
widget_servers:
  hosts:
    widget_host1:
      ansible_host: host1
    widget_host2:
      ansible_host: host2
    widget_host3:
      ansible_host: host3
```
yq로 파싱하여 모든 ansible_host를 읽는다.

```sh
❯ cat host.yaml | yq '.widget_servers.hosts["*"].ansible_host' |\
   xargs -I{} echo -H {} |\
   tr '\n' ' '
-H host1 -H host2 -H host3
```

#### 명령하고 출력 다루기
이 명령 결과를 인자로 주어 PSSH 명령을 해본다(나는 다시 INI 파일의 경우로)

```sh
❯ pssh $(cat <inventory_file> grep -vE '^\[' | awk 'NF {sub("<type>", "", $1); print $1}' | sort | uniq | xargs -I{} echo -H {} | tr '\n' ' ')\
   curl ifconfig.me
[1] 12:47:14 [SUCCESS] host3
[2] 12:47:14 [SUCCESS] host1
[3] 12:47:14 [SUCCESS] host2
```

PSSH는 실행하는 곳에선 각 호스트 성공 실패 여부만 출력하기 때문에 보낸 명령, `curl ifconfig.me`, 의 출력을 볼 수 없다. `-i` 옵션을 주면 보낸 명령의 표준 출력을 출력하지만, 에러도 같이 출력한다.

따라서 표준 출력만 출력하는 `--inline-stdout`을 쓰거나 `-i` 옵션과 `curl -s ifconfig.me`를 같이 사용하여 curl에서 표준 에러를 없애는 법이 있다:

```sh
# 호스트 인자 만드는 명령을 $(ansible_inventory_to_pssh_host) 로 대체
❯ pssh $(ansible_inventory_to_pssh_host) -i curl -s ifconfig.me
[1] 13:11:32 [SUCCESS] host2
223.xxx.yyy.zzz[2] 13:11:32 [SUCCESS] host3
223.xxx.yyy.zzz[3] 13:11:32 [SUCCESS] host1
223.xxx.yyy.zzz

❯ pssh $(ansible_inventory_to_pssh_host) -i curl -s ifconfig.me
[1] 13:11:34 [SUCCESS] host3
223.xxx.yyy.zzz[2] 13:11:34 [SUCCESS] host1
223.xxx.yyy.zzz[3] 13:11:34 [SUCCESS] host2
223.xxx.yyy.zzz
```

출력이 조금 이상하다. curl의 출력인 IP가 개행을 포함하지 않고 넘어와 다음 호스트 성공에 대한 메세지와 같은 줄에 출력된다. 예시의 경우는 아주 간단해서 한 눈에 결과를 파악하는데 어려움은 없다(실제로 나는 공인 IP가 모두 같은 호스트에 명령했다). 하지만 호스트가 훨씬 많거나 명령의 결과나 양식이 조금 더 복잡해지면 PSSH가 뭉쳐진 결과를 다루는데 어려움이 있을 것이다.


## Ansible

PSSH는 명령을 보낼 호스트명 인자를 만드는데에 공수가 많이 들어갔다. 하지만 Ansible은 인벤토리 파일을 그대로 쓸 것이기 때문에 문제가 없다. 또 실질적(?)으로 유효한 점도 있다. Ansible을 쓰고 있다면 형상관리가 되고 있을 것이라 호스트명과 그룹 등을 원하는대로 다루기 쉬울 것이다. 반면 PSSH는 자주 쓰는 명령이 아니라, 위에서 소개한 방법으로 Ansible 인벤토리 코드가 바뀔 때 호스트 파일을 만들어야 할 것이다.

```sh
❯ ansible --version
ansible [core 2.11.6]
  config file = /etc/ansible/ansible.cfg
  ...(opaque)...
  python version = 3.8.8 (default, Nov 23 2021, 14:07:49) [Clang 13.0.0 (clang-1300.0.29.3)]
  jinja version = 3.0.3
  libyaml = True
```
Ansible ad-hoc 명령에 대해 짧게 설명한다. 많은 사람들이 ansible-playbook 명령보다 오히려 ansible 명령이 더 어색할 수도 있다. 이미 있는 인프라를 구조를 갖추어 코드로 옮기면 playbook을 바로 사용했을 것이고 나도 그렇다:

```sh
❯ ansible <group> -i <inventory_file> -m <module> -a <module_args>
```

Playbook이 아닌 모듈과 모듈의 인자를 지정하여 마치 단일 태스크처럼 실행할 수 있다. 지금 하고 있는 예시로 명령한다면:

```sh
❯ ansible <group> -i <invenotry_file> -m command -a "/usr/bin/curl -s ifconfig.me"
host2 | CHANGED | rc=0 >>
223.xxx.yyy.zzz
host3 | CHANGED | rc=0 >>
223.xxx.yyy.zzz
host1 | CHANGED | rc=0 >>
223.xxx.yyy.zzz
```

command 모듈에 실행하려는 명령을 인자로 주었다. PSSH 출력보다 실행 호스트와 출력이 바로 다음 줄에 나와 조금 더 정돈된 모습을 보여준다.

#### 모듈 출력을 포함해 JSON으로 처리하기
단순히 쉘에 명령을 날리는게 아니라 Ansible을 쓰는 장점을 이용하기 위해 curl에 해당하는 [uri](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html) 빌트인 모듈을 사용해봤다.

```sh
❯ ansible <group> -i <inventory_file> -m uri -a "
    url=https://ifconfig.me
    return_content=yes
    http_agent=curl/7.64.1"
```

- `url`: curl과 달리 프로토콜을 명시해주어야 한다.
- `return_content=yes`: 요청의 응답(`content` 필드)이 포함되게 한다.
- `http_agent=curl/7.64.1`: ifconfig.me는 agent에 따라 출력을 달리 준다(브라우저에서 접속하여 확인해보자). 따라서 IP 주소만 응답 받도록 curl 명령할 때와 같게 해주었다.

출력은 다음과 같다:
```sh
type_host1 | SUCCESS => {
    "access_control_allow_origin": "*",
    "alt_svc": "clear",
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "connection": "close",
    "content": "223.xxx.yyy.zzz",
    "content_length": "15",
    "content_type": "text/plain; charset=utf-8",
    ...(생략)
}
type_host2 | SUCCESS => {
    "access_control_allow_origin": "*",
    "alt_svc": "clear",
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "connection": "close",
    "content": "223.xxx.yyy.zzz",
    "content_length": "15",
    "content_type": "text/plain; charset=utf-8",
    ...(생략)
}
type_host3 | SUCCESS => {
    "access_control_allow_origin": "*",
    "alt_svc": "clear",
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "connection": "close",
    "content": "223.xxx.yyy.zzz",
    "content_length": "15",
    "content_type": "text/plain; charset=utf-8",
    ...(생략)
}
```
"Ansible host | 성공 실패 여부 =>" 뒤에 모듈 결과인 JSON 있고 여기엔 curl의 출력(content)를 포함해서 HTTP 응답 헤더나, ansible facts 같은 메타데이터들이 있다. 이걸 좀 다듬으면 Ad-hoc 명령의 결과를 JSON으로 쉽게 다룰 수 있어 보인다:

```sh
❯ <ansible_ad_hoc_command> | sed -E 's/(.*) \| FAILED\! \=> \{/\{"ansible_host": "\1", "success": false,/;
                                     s/(.*) \| SUCCESS \=> \{/\{"ansible_host": "\1", "success": true,/'
```

ansible_host와 성공 실패 여부 필드(success)를 포함한 JSON으로 바꿔준다. 정렬은 이상하지만, jq로 이쁘게 출력하거나 파싱할 수 있다:

```sh
❯ ansible <group> -i <inventory_file> -m uri -a "url=https://ifconfig.me
                                                 return_content=yes
                                                 http_agent=curl/7.64.1" |\
sed -E 's/(.*) \| FAILED\! \=> \{/\{"ansible_host": "\1", "success": false,/;
        s/(.*) \| SUCCESS \=> \{/\{"ansible_host": "\1", "success": true,/' |\
jq '{"host": .ansible_host, "content": .content}'
{
  "host": "type_host2",
  "content": "223.xxx.yyy.zzz"
}
{
  "host": "type_host1",
  "content": "223.xxx.yyy.zzz"
}
{
  "host": "type_host3",
  "content": "223.xxx.yyy.zzz"
}
```

나는 이런 간단한 명령을 여러 호스트에 날리는 상황에선, PSSH 보단 Ansible ad-hoc으로 작업을 하려고 한다. 인벤토리를 그대로 쓸 수 있고 인벤토리 파일은 형상관리가 잘 될 수 있다는 점이 제일 큰 이유이다. 결과를 JSON으로 처리할 수 있는 것도 장점이다. 또 Ansible 모듈의 장점(e.g. 멱등성 보장)과 모듈도 이것저것 써보게 되는게 낫다고 생각해서 그렇다.


## 정리
- PSSH
  - 형상관리 되고 있는 Ansible 인벤토리 파일을 조작해서 호스트 인자로 쓸 수 있다.
  - 명령 결과가 plain text라 조작(reduce등)이 어렵다.
- Ansible
  - [결과를 조금 바꾸면](https://gist.github.com/flavono123/65522ea36a1c957c74daac0dd99c8180) JSON으로 처리할 수 있다.


---

## 참고
- https://linux.die.net/man/1/pssh
- https://ansible-tips-and-tricks.readthedocs.io/en/latest/ansible/commands/
- https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html

