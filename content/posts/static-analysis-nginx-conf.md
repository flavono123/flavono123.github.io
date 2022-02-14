---
title: "nginx 설정 정적 분석"
date: 2022-02-14T19:13:48+09:00
tags:
- nginx
- jq
---

회사에선 nginx를 웹 서버이자 리버스 프록시로 여러군데서 쓰고 있다. 설정 파일을 레일즈 배포 시 ERB로 템플릿하고 있다. 그래서 설정 내용이 레일즈 앱에 종속적이기도 하고(배포 변수가 레일즈 앱에 연관이 있을 시), 엄청 길다.

이걸 간편하게 바꿀 임무가 주어졌다. 2000줄이 넘는 설정 파일이라 단순히 눈으로 보며 고치긴 어려울거 같아, 구조를 파싱한 후 정적으로 분석해보기로 했다.

## crossplane
먼저 nginx 설정 파일을 구조를 파싱할 수 있는 도구가 있을거 같아 찾아봤다. [crossplane](https://github.com/nginxinc/crossplane)이라는 NGINX, Inc.에서 파이썬으로 만든 도구가 있다. [아쉽게도 현재는 개발이 중단된걸로 보인다](https://github.com/nginxinc/crossplane/pull/95#issuecomment-986565551)(더 아쉽게 k8s로 인프라를 관리하는듯?한 [동명이프로젝트](https://www.cncf.io/projects/crossplane/)가 잘 나가고 있는걸로 보인다). 하지만 파싱할 때 이슈 리포팅된 엣지 케이스에 해당하진 않았다. 또 [NGINX Amplify](https://www.nginx.com/products/nginx-amplify/)라는 nginx 모니터 도구 내부에서 쓰이는 모태라고 하니, 꽤 쓸만한거 같다.

처음엔 무턱태고 upstream과 가상 서버만 정의된 단일 파일을 파싱했다가 실패했는데, 극복하고 nginx 설정의 대략적인 구조를 파악한 것을 [전 포스트](/posts/nginx-conf-structure)에 정리해놨다.

> nginx 설정 구조를 더 단순화하면 그저 디렉티브의 트리 구조이다. 블록 디렉티브일 경우 하위 디렉티브들을 포함하는 디렉티브의 트리구조가 루트(main 컨텍스트)부터 계속 이어지는 단순한 구조이다.

crossplane이 JSON으로 짠하고 파싱할 수 있는 이유이다. [README의 Schema](https://github.com/nginxinc/crossplane#schema) 중 디렉티브 부분을 보면 다음과 같다:

```json
{
    "directive": String, // the name of the directive
    "line": Number,      // integer line number the directive started on
    "args": Array,       // Array of String arguments
    "includes": Array,   // Array of integers (included iff this is an include directive)
    "block": Array       // Array of Directive Objects (included iff this is a block)
}
```

main 컨텍스트에 해당하는 몇몇 루트 디렉티브**들**이 있고 optional prop인 `block`에 위 디렉티브 객체 배열이 있는 트리 구조이다.

간단히 쓴 nginx 설정에 대응하자면:
```sh
# main context
...
events {
  # events context
  ...
}

http {
  # http context
  ...
  upstream @upstream1 {
    # upstream context
    server proxy_server1;
    ...
  }

  server {
    # server context
    location @upstream1 {
      # location context
    }

    location /match/criteria {
      # first location context
    }
  }

  server {
    ...
  }
}
```

이러한 nginx 설정은:
```json
// $ crossplane parse nginx.conf | jq -r '.config[2].parsed[]'
[
  {
    "directive": "events",
    "block": [...]
  },
  {
    "directive": "http",
    "block": [
      {
        "directive": "upstream",
        "args": ["@upstream1"],
        "block": [
          {
            "directive": "server",
            "args": ["proxy_server1"]
          },
          ...
        ]
      },
      {
        "directive": "server",
        "block": [
          {
            "directive": "location",
            "args": ["@upstream1"]
          },
          {
            "directive": "location",
            "args": ["/match/criteria"]
          },
          ...
        ]
      }
    ]
  },
  ...
]
```

이런 JSON으로 출력된다는 이야기이다
- `line`은 항상 있는 prop이지만 편의상 생략했다.
- `crossplane parse nginx.conf`의 결과는 파일마다 성공, 실패한 결과가 담겨 있는 `config[*]` 배열에 결과 객체가 있다(위 README Schema 참고). 세번째를 선택한건(`[2]`) 분석하려는 대상 파일이기 때문이다. `config[*].parsed` 키에 main 컨텍스트의 디렉티브가 있는데, 위에서 말한 듯 **여러 개**이다.
  - 글에선 편의상 jq 구문이 아닌 JSONPATH를 사용함

## find_all(recursive: true)
최상위엔 events와 http, 그리고 http 바로 아래 깊이엔 upstream과 server들, server 안엔 location 디렉티브가 있을 것이다. location 디렉티브는 중첩이 가능하므로 2레벨보다 더 깊을 수도 있다. 이렇게 계층 위치가 설계된 디렉티브가 아니라면 더 다양한 트리 레벨에 위치할 수 있다. 기본 값을 상속하고 하위 컨텍스트에서 재정의를 할 수도 있다. 트리 전역에 있는 모든 디렉티브를 어떻게 찾을 수 있을까?

JSON의 반복되는 계층 트리 구조에서 모든 오브젝트에 대해 출력하는 방법은 [전 포스트](/posts/jq-descendants)에서 정리했다. Recursive descendants(`..`) 식별자를 이용해 필터하자:

```sh
# 모든 server 디렉티브 찾기
$ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "server")'
# 모든 location 디렉티브 찾기
$ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "location")'
# 모든 if 디렉티브 찾기
$ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "if")'
```

## 무엇이 설정을 복잡하게 할까?
단순히 길어서, 줄이 많아서 nginx 설정이 복잡한건 아닐것이다. 특히 템플릿으로 range 순회를 한다면 줄은 선형적으로 늘어날테니.

첫번째로 location 중첩이 많으면 복잡할 것이라고 생각했다. location은 nginx 기본 구조를 정리하면서 알게된 유일하게 중첩이 가능한 블록 디렉티브였고, 위 설정 파일에서도 많이 쓰이고 있었다(137). 또 미리부터 과도한 sumup은 오버엔지니어링이 아닌가. 그리고 실행되는 코드가 아닌, 설정은 평평한(flat)한 편이 낫다고 생각한다.

먼저, 모든 server 디렉티브의 1 depth location 디렉티브를 출력한다:
```sh
❯ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "server") | (.block // [])[] | select(.directive? == "location").lines' | wc -l
     137
```
- `(.block // [])[]`: server보다 1 depth 아래의 location 디렉티브만 선택하기 위해 `.block`의 값들을 출력한다. `.block`이 없는 server 디렉티브가 있을 수 있기 때문에(= null 일 수 있기 때문에), 기본 값으로 `[]`을 준다. 여기서 `..`를 사용하면 모든 하위 depth의 location을 반환해서 쓰지 않는다.
- `select(.directive? == "location").line`: location 디렉티브만 찾아 객체에 항상 있고 값이 하나인 `.line`을 출력하여 개수를 센다.

이제 2 depth location을 찾아본다. 1 depth location 디렉티브에서 위 방법을 반복한다:
```sh
❯ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "server") | (.block // [])[] | select(.directive? == "location") | (.block // [])[] | select(.directive? == "location").line' | wc -l
       0
```

2 depth의 location이 없다. 중첩 location이 하나도 없다!(위에 전체 location 디렉티브 개수와 1 depth의 개수가 같은데에서 눈치챌 수 있다. TDD 같네...). 이건 꽤 좋은 신호라고 판단했다.

두번째론 if가 괜찮게 쓰이는가 걱정됐다. 내가 파악하는 nginx 설정엔 일단 if가 많은 편이다(90). 

언젠가 봤던 [If is Evil... when used in location context](https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/)이라는, (무려 공식) 포스트가 생각났고 다시 읽어봤다. if가 location 컨텍스트에서 왜 나쁜지, 어떤 오작동을 하는지는 직접 테스트 해보고 나중에 다른 포스트에 정리하기로 했다. 어쨌든 location 내의 if가 나쁘다면, 잠재적으로 위험한 if를 찾아보기로 했다:
```sh
❯ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "location") | .. | select(.directive? == "if").line' | wc -l
      45
```
전체 if 중 절반은 위험할 수도 있는 if였다(여기서부터 "템플릿의 range 순회 코드로 반복되고 있구나"가 느껴지긴했다).

그렇다면 이 45개의 if 디렉티브는 정말 위험할까? if is evil 포스트에서 [location 내 if 사용의 모범 사례(What to do instead)](https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/#what-to-do-instead)를 보면 패턴이 있다. error_page를 정의하고 if 컨텍스트에서 해당 페이지로 바로 return 하는 것이다. 과연 이 패턴을 따르고 있을까 확인해봤다:
```sh
❯ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "location") | select(.block[].directive == "if").block[].directive'
error_page
error_page
if
return
...
```
- `select(.block[].directive == "if").block[].directive`: 앞 입력 location 컨텍스트에 if 디렉티브가 있다면 모든 컨텍스트 디렉티브를 출력한다.

"error_page\nerror_page\nif\nreturn"가 45번 반복되고 있었다. 즉 모두 같은 패턴이었는데, error_page와 return의 args도 같았다(출력은 생략):
```sh
❯ crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "location") | select(.block[].directive == "if").block[] | [.directive, (.args | join(" "))] | join(" ")'
```

다행히 모범 사례를 잘 따르고 있었다. 커밋을 추적해보니 참고한 블로그 링크가 있는데, 지금은 404로 나온다..

## 경로 분기 집계
server 컨텍스트에서 server_name(Host 헤더)과 중첩된 location의 args(match pattern)을 모두 이어 붙이면 경로 분기가 잘 보일거 같아 집계해봤다:

```sh
❯ crossplane parse nginx.conf | jq '.config[2].parsed[] | .. | select(.directive? == "server") as $servers | .. | select(.directive? == "server_name").args as $server_name | $servers | .. | select(.directive? == "location").args as $location | [$server_name, $location] | flatten | join("")'
```
- `select(.directive? == "server") as $servers`: sever 디렉티브를 찾아 `$server` 레퍼런스에 저장해둔다.
- `| .. | select(.directive? == "server_name").args as $server_name`: server 컨텍스트의 모든 server_name의 args를 `$server_name` 레퍼런스에 저장한다.
- `$servers | .. | select(.directive? == "location").args as $location`: 다시 server 컨텍스트에서 location 디렉티브의 args를 `$location` 레퍼런스에 저장한다. 여기선 파싱한 설정에 1 depth 이상의 location이 없어서 간단하게 `..`을 썼지만, 그 이상 있다면 server 또는 location 컨텍스트(`.block`)를 알고 접근해야할 거 같다(재귀적으로 짤 수 있을까..?)
- `[$server_name, $location] | flatten | join("")`: 입력은 버리고 앞서 저장한 두 레퍼런스를 concat하여 출력한다. server_name도 location도 배열이기 때문에 flatten 한다.

이건 생각보다 결과가 좋지 않았다. 왜냐하면 server_name이나 location의 args 모두 정규표현식이 가능해서 단순히 URI 보단 훨씬 보기 어려운 출력이 나왔다:
```sh
<host>~^<subdomain1>\\..*~^<subdomain2>\\..*~/<path>/
```

일부러 정규표현식 그리고 location은 modifier가 보이도록 시맨틱만 안보이게 처리했다. 저 사이의 백슬래시, 점, 물결 때문에 URI처럼 보이진 않았다. 하지만 대략 한 도메인(호스트)에서 경로 분기를 파악하기엔 괜찮은 목록이라 더 정교한 작업을 하진 않았다.

## 정리
사실 정적 분석이라 했지만 그렇게 대단한건 없다. 또 더 자세한 분석은 컨텍스트의 디렉티브를 일일이 확인할 필요가 있다(동적 분석?). 

그럼에도 여러 줄의 설정 파일을 less로 읽는 것보다, JSON으로 구조를 파싱하고 필터, 집계한 것이 큰 그림을 파악하기에 좋았다. if is evil 같은 경우엔 안티 패턴에 대한 린터를 만드는 것도 가능할거 같다.

위에서 쓴 명령을 일반화 해보았다.

- 파싱한 특정 설정 파일 내 모든 디렉티브 찾기 
```sh
$ crossplane parse <nginx.conf> | jq -r '.config[0].parsed[] | .. | select(.directive? == "<directive>")'
```

- 특정 컨텍스트 내의 특정 디렉티브 찾기
```sh
$ crossplane parse <nginx.conf> | jq -r '.config[0].parsed[] | .. | select(.directive? == "<context>") | .. | select(.directive? == "<directive>")'
```

- 특정 컨텍스트 내에 특정 디렉티브가 있을 경우 컨텍스트 내의 모든 디렉티브 출력(e.g. if is evil의 안티 패턴을 찾기 위해)
```sh
crossplane parse nginx.conf | jq -r '.config[2].parsed[] | .. | select(.directive? == "<context>") | select(.block[].directive == "<directive>").block[].directive'
```

- server 컨텍스트의 server_name + location (1 depth만 가능)
```sh
$ crossplane parse nginx.conf | jq '.config[<n>].parsed[] | .. | select(.directive? == "server") as $servers | .. | select(.directive? == "server_name").args as $server_name | $servers | .. | select(.directive? == "location").args as $location | [$server_name, $location] | flatten | join("")'
```

## 참고
- https://github.com/nginxinc/crossplane
- https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/
