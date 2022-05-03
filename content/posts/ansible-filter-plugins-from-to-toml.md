---
title: "Ansible 커스텀 필터 from_toml, to_toml"
date: 2022-05-03T09:10:13+09:00
tags:
- ansible
---

[TOML](https://toml.io/en/)은 YAML/JSON 보단 빈도는 적지만 설정 파일 포맷으로 쓰이는 곳이 있다. 그래서인지 [Ansible에 `from_yaml`, `to_yaml`, `from_json`, `to_json` 등의 Jinja2 필터는 있지만](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#formatting-data-yaml-and-json), TOML용 필터는 없다.

쓸 일이 있기도 해서 Ansible에서 사용할 수 있는 `from_toml`, `to_toml` Jinja2 필터를 만들어 본다. 또 나는 파이썬을 잘 모르지만 필터를 작성하는데엔 무리가 없었다. 다만 같은 이유로 이 글에서 파이썬과 관련한 자세한 설명은 생략한다.


## 준비
빈 경로에 다음과 같은 파일을 써준다:
```sh
❯ tree .
.
├── localhost
└── var.yaml

0 directories, 2 files

❯ tail -n 10 *
==> localhost <==
[local]
localhost

==> var.yaml <==
var:
  a:
    b:
      c: string
    d: 1

```

- `localhost`: Ansible ad-hoc 명령을 로컬에 하기 위한 호스트 INI 파일
- `var.yaml`: `to_toml`의 입력으로 사용할 변수

변수는 최소한의 검증하고 싶은 부분만 커버하도록 만들었다:
- 중첩 키에 대해 동작하는지
- 값 타입
  - 정수(숫자)
  - 문자열

## 필터 작성
Ansible이 파이썬 필터를 로드하는 기본 위치는 다음 명령에서 확인할 수 있다:
```sh
❯ ansible-config list | grep filter_plugins -B7 -A3
DEFAULT_FILTER_PLUGIN_PATH:
  default: ~/.ansible/plugins/filter:/usr/share/ansible/plugins/filter
  description: Colon separated paths in which Ansible will search for Jinja2 Filter
    Plugins.
  env:
  - name: ANSIBLE_FILTER_PLUGINS
  ini:
  - key: filter_plugins
    section: defaults
  name: Jinja2 Filter Plugins Path
  type: pathspec

```

이 실습에선 default 경로에 파일을 만들지 않고 별도 경로에 만든 후 설정(`ANSIBLE_FILTER_PLUGINS`)하여 로드한다:
```sh
❯ mkdir filters
```
```python
# filters/toml.py
import toml

def from_toml(toml_string):
    return toml.loads(toml_string)


def to_toml(yaml_variable):
    return toml.dumps(dict(yaml_variable))

class FilterModule(object):

    def filters(self):
        return {
            'from_toml': from_toml,
            'to_toml': to_toml
        }


```

먼저 `FilterModule` 클래스를 선언한 것은 Ansible [lib/ansible/plugins/filter/core.py](https://github1s.com/ansible/ansible/blob/devel/lib/ansible/plugins/filter/core.py#L560-L561)와 [인터넷에서 찾아볼 수 있는 필터 플러그인 예제](https://blog.oddbit.com/post/2019-04-25-writing-ansible-filter-plugins/)를 참고했다([문서](https://docs.ansible.com/ansible/latest/plugins/filter.html)에서 상세히 설명하지 않는다). `filters()` 메소드가 overwrite 되지 않는 모양인데 이것이 파이썬 특징인지, Ansible 필터 플러그인의 구현이 그런지는 확인하지 않았다.

새로 정의한 `from_toml`, `to_toml` 필터 동작은 [파이썬 패키지 toml](https://github.com/uiri/toml#api-reference)에 의존한다. 엣지 케이스(?)에 대한 처리는 하지 않고 변수명에서 `from_toml` TOML 문자열 입력만 받을 것을, `to_toml`은 YAML 변수 입력만 받을 것을 암시한다. 정의한 메소드를 같은 이름의 키의 사전으로 `FilterModule.filters()`에서 반환한다.

## to_toml

Ad-hoc 명령으로 `to_toml` 테스트 해보면 문자열에 대해 잘 동작하지 않는다([Ad-hoc 명령에 대한 참고](/posts/ansible-pssh/#ansible)):
```sh
❯ ANSIBLE_FILTER_PLUGINS=filters ansible local -i localhost \
  -m debug \
  -a "msg={{ var | to_toml }}" \
  -e "@var.yaml"
localhost | SUCCESS => {
    "msg": "[a]\nd = 2\n\n[a.b]\nc = [ \"s\", \"t\", \"r\", \"i\", \"n\", \"g\",]\n"
}

~/tmp/toml on ☁️  (ap-northeast-2) on ☁️ flavono123@gmail.com
❯ echo -e "[a]\nd = 2\n\n[a.b]\nc = [ \"s\", \"t\", \"r\", \"i\", \"n\", \"g\",]\n"
[a]
d = 2

[a.b]
c = [ "s", "t", "r", "i", "n", "g",]
```

[Ansible에서 YAML 문자열 객체를 `<class 'ansible.parsing.YAML.objects.AnsibleUnicode'>`로 다루어 일어나는 문제](https://www.iops.tech/blog/generate-toml-using-ansible-template/)라고 한다. 이 문제는 YAML 문자열을 홑 또는 쌍따옴표로 감싸도 발생한다. TOML은 문자열을 쌍따옴표(escaped) 또는 홑따옴표(literal)로 감싸는데 반해 YAML은 그러지 않아도 동작하니 파이썬에서 다루는 것이 다를 것이라 생각했다. 역시 이 문제에 대해 파이썬 코드 레벨까지 보진 않았다. 아무튼 **Ansible YAML 문자열 객체를 별다른 처리하지 않으면 파이썬 TOML에선 각 문자를 원소로 하는 배열로 반환**해버린다.

이 문제를 해결하기 위해, 위 블로그 글에서 소개하는 트릭을 사용했다. YAML변수를 JSON 객체로 덤프하여 Ansible YAML 문자열 객체를 파이썬 문자열로 바꾼 후 다시 JSON -> TOML로 로드/덤프하는 것이다:
```python
import json

def to_toml(yaml_variable):
    j = json.dumps(dict(yaml_variable))
    d = json.loads(j)
    return toml.dumps(d)
```

filters/toml.py를 수정 후 다시 테스트하면 문자열에 대해 잘 동작한다:
```sh
❯ ANSIBLE_FILTER_PLUGINS=filters ansible local -i localhost \
  -m debug \
  -a "msg={{ var | to_toml }}" \
  -e "@var.yaml"
localhost | SUCCESS => {
    "msg": "[a]\nd = 2\n\n[a.b]\nc = \"string\"\n"
}
```

## from_toml
위 `to_toml` 테스트의 결과를 다시 입력 변수로 `from_toml`은 간단하게 테스트 할 수 있다. 변수 전체를 escaped한 쌍따옴표로 감싸는 것에 주의한다:
```sh
❯ ANSIBLE_FILTER_PLUGINS=filters ansible local -i localhost \
  -m debug \
  -a "msg={{ var | from_toml }}" \
  -e "var=\"[a]\nd = 2\n\n[a.b]\nc = \"string\"\n\""
localhost | SUCCESS => {
    "msg": {
        "a": {
            "b": {
                "c": "string"
            },
            "d": 2
        }
    }
}

```

## 정리
만든 필터는 [containerd의 설정](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md)을 템플릿할 때 쓰게 될 것이다.

필터 자체는 아주 간단하지만, Ansible 필터 플러그인을 만들기 위해 필요한 설정과 파일 형식을 알 수 있었다:
- 필터 파일 설정 경로
  - 환경변수: `ANSIBLE_FILTER_PLUGINS`
  - ansible.cfg(INI): `[defaults]` 섹션, `filter_plugins` 키
  - 기본 값: ~/.ansible/plugins/filter:/usr/share/ansible/plugins/filter
- 필터 파이썬 파일 포맷
  - 필터로 동작할 메소드 정의
  - 필터 이름 키와 정의한 메소드 값 쌍의 사전을 `FilterModule.filters()`에서 반환


---

## 참고
- https://www.iops.tech/blog/generate-toml-using-ansible-template/
- https://toml.io/en/


