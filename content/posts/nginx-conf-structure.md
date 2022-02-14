---
title: "웹 서버/리버스 프록시로서 nginx 설정 구조(HTTP)"
date: 2022-02-14T13:12:46+09:00
tags:
- nginx
---

nginx 설정을 바꾸고 재적용(reload) 전 테스트 시 오류를 본적이 있다:
```sh
$ sudo nginx -t -c /path/to/nginx.conf 
nginx: [emerg] "upstream" directive is not allowed here in /path/to/nginx.conf:6
```

upstream 디렉티브는 원래부터 저 위치에 있었고, 다른 컨텍스트 내부를 수정한거라 원인이 뭔지 몰랐다. 과거에 이렇게 잘 몰랐다가 nginx 설정을 파싱할 일이 있어 해보았는데 같은 에러를 만났다. 따라서, 제대로 공부한적 없는, nginx 설정에 대해 한번 정리해본다(디렉티브 및 컨텍스트는 회사에서 쓰고 있는 것만 정리했다).

## 기본 설정 파일
nginx는 설정을 읽어 오는 기본 파일 경로가 있다:
- /etc/nginx/nginx.conf
- /usr/local/nginx/conf/nginx.conf
- /usr/local/etc/nginx/nginx.conf

패키지, 배포판에 따라 조금씩 다르다. 루트 권한으로 apt 패키지 설치한 우분투에선 가장 첫번째 파일이고, brew 로 설치한 로컬 맥에선 가장 마지막 파일이다.

그리고 한 파일에서 다른 설정 파일을 참조할 수 있는 [include 디렉티브](http://nginx.org/en/docs/ngx_core_module.html#include)가 있다. 이것이 내가 만난 문제 에러에 대한 원인이자 스포인데, /path/to/nginx.conf 파일은 단독적으론 불완전한 설정이고 include 디렉티브 컨텍스트 내에 삽입되어야 오류가 없는 설정이 된다(아직 디렉티브, 컨텍스트 등이 무얼 의미하는지 설명하지 않았다. 이를 설명하고 뒤에서 한번 더 정리한다).

## 디렉티브
설정의 기본 구성 요소인 디렉티브는 크게 두가지로 나뉜다.
- 단순 디렉티브(simple, single-line): 한줄에 세미콜론으로 끝난다. 디렉티브와 인수들(arguments) 순서로 쓰임.
- 블록 디렉티브: 디렉티브 뒤 중괄호(`{}`)로 표시된 블록이 있다. 블록은 새로운 컨텍스트이고 컨텍스트마다 쓰일 수 있는 디렉티브가 정해져 있다. 디렉티브 중엔 컨텍스트로 상속되는 것들도 있다. 특정 디렉티브의 블록을 특정 컨텍스트로 지칭한다.

아무것도 없는 설정 파일 최상단은 core 또는 main 컨텍스트이다. 여기엔 전역과 관련한 디렉티브를 정의한다(운영체제 사용자, 워커 개수, pid 파일 등):
```sh
# main(core) context
user nginx;
worker_processes 8;
pid /run/nginx.pid;
...

```

이 main 컨텍스트에만 위치할 수 있는 몇 개의 컨텍스트가 있다.

## 이벤트 컨텍스트
```sh
# main(core) context
events {
	# events context
	use epoll;
  worker_connections 1000;
  multi_accept on;
  ...
}
```
nginx는 이벤트 기반 연결 프로세싱 모델이다. 워커 프로세스의 연결 처리 관련한 디렉티브를 정의한다:
- [`use`](http://nginx.org/en/docs/ngx_core_module.html#use): 연결 방법(methods)
- [`worker_connections`](http://nginx.org/en/docs/ngx_core_module.html#worker_connections): 워커당 연결 수
- [`multi_accept`](http://nginx.org/en/docs/ngx_core_module.html#multi_accept): 연결 처리 중 대기(pending) 연결도 받을 것인지

## HTTP 컨텍스트

모든 HTTP/HTTPS 처리 관련한 디렉티브를 정의하는 컨텍스트. nginx를 주 용도인, 웹 서버나 리버스 프록시로 사용한다면, 가장 주요하게 쓰이는 컨텍스트이다:
```sh
# main context

events {
  # events context
  ...
}

http {
  # http context
  ...
}
```
http 컨텍스트 안엔 각 요청을 처리하는 가상 서버(virtual server)이자, server 컨텍스트가 정의된다. 모든 가상 서버에 적용되는 디렉티브는 여기에 정의하여 상속되도록 한다. 예를 들면:
- 로그 파일 위치
  - [`error_log`](http://nginx.org/en/docs/ngx_core_module.html#error_log)
  - [`access_log`](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log)
  - ...
- TCP keepalive 연결 설정
  - [`keepalive_requests`](https://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_requests)
  - [`keepalive_timeout`](https://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout)
- 압축
  - [`gzip`](https://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip)
- ...

위 디렉티브는 각 가상 서버 내에서 재정의(overwrite) 될 수 있고 기본 값으로 정의된다.

## 서버 컨텍스트
```sh
# main context

http {
    # http context
  server {
    # first server context
  }

  server {
    # second server context
  }
  ...
}
```

http 컨텍스트 내에 나란히 여러 개 정의할 수 있는 컨텍스트이고 가상 서버라고도 한다. 각 가상 서버가 전체 http 요청의 일부를 처리한다. nginx는 다음 두 디렉티브로 요청이 처리될 가상 서버를 고른다:
- [`listen`](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen): IP + 포트
- [`server_name`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name): Host 헤더와 매칭, `listen`에 매칭되는 가상 서버가 여러 개일 때 사용하는 필터. http 컨텍스트에서 상속되는 디렉티브.

서버 컨텍스트 내엔 URI로 분기하는 여러 로케이션 컨텍스트를 정의할 수 있다.

## 로케이션 컨텍스트
서버 컨텍스트 내에 정의되는 컨텍스트. 서버처럼 여러 컨텍스트를 정의할 수 있고 반면에 서버와 달리 중첩(nested) 정의가 가능하다. 서버가 말 그대로 서버(=호스트, 포트도 포함하지만)를  찾는 일이라면, 그 서버 내에서 경로, 요청 URI,를 특정하는 컨텍스트이다. 중첩이 가능하기 때문에 공통 경로 처리에 대해 sumup 할 수 있다:
```sh
location <match_modifier> <uri> | @<name> {
  ...
}


# main context

server {
  # server context

  location /match/criteria {
    # first location context
  }

  location /other/criteria {
    # second location context

    location nested_match {
      # first nested location
    }

    location other_nested {
      # second nested location
    }
  }

}
```

[`location` 디렉티브](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)


match modifier는 안 쓰이거나(optional) 다음 네가지로 매칭한다:
- (none): 접두사(prefix) 매칭
- `=`: 정확한(exact) 매칭
- `~`: case-sensitive 정규표현식 매칭
- `~*`: case-insensitive 정규표현식 매칭
- `^~`: 최적의 비정규표현식에 매칭? (if this block is selected as the best non-regular expression match, regular expression matching will not take place.)

마지막 설명은 이해하지 못했다. 하지만 modifier 외에 로케이션 디렉티브 순서에 따른 우선순위나 nginx의 매칭 알고리즘이 따로 있고 이는 이 포스팅에서 다루려는 범위를 넘어가서 일단 신경 쓰지 않았다(나중에 [Understanding Nginx Server and Location Block Selection Algorithms](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)의 내용도 정리해보자).

또 단순히 `@<name>`만 써서 해당 블록으로 요청을 보낼 수도 있다. 회사에선, 이어서 설명할, 업스트림 컨텍스트의 이름을 `@` prefix로 정의하여 사용하고 있다. 로케이션에서 `@<name>` 사용 시 이는 redirection이 아닌 요청이 되고, 중첩 로케이션에서 쓰거나 이 컨텍스트에 로케이션 컨텍스트를 중첩시킬 수 없다.

## 업스트림 컨텍스트
nginx가 요청을 프록시 할 수 있는 서버 풀을 업스트림이라 하며 이를 정의하는 블록 디렉티브이다. http 컨텍스트 밑에, server 컨텍스트와 나란히 정의되고 서버나 로케이션에서 이름을 참조하여 요청할 수 있다:
```sh
# main context

http {
  # http context

  upstream @<upstream_name> {
    # upstream context
    server <proxy_server1>;
    server <proxy_server2>;
    ...
  }

  server {
    # server context
    location @<upstream_name> {
      # location context
      # @<upstream_name> 업스트림으로 요청을 프록시 함
    }
  }

}
```

## 정리
웹 서버나 리버스 프록시로서 nginx 설정 구조(=컨텍스트 트리)는 다음과 같다:
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
    server proxy_server2;
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

    location /other/criteria {
      # second location context

      location nested_match {
        # first nested location
      }

      location other_nested {
        # second nested location
      }
    }
  }

  server {

  }
}
```

- 최상위 main 컨텍스트엔 event, http 컨텍스트가 정의되어 있다.
- http 컨텍스트엔 upstream과 가상 서버(server) 컨텍스트가 중첩 없이 정의되어 있다.
- 가상 서버 내엔 location 컨텍스트가 중첩 가능하게 정의되어 있다.

include 디렉티브는 다른 파일로 설정을 확장할 수 있다. 나의 경우 http 에서 확장 할 수 있는 upstream 및 가상 서버를 별도 파일에서 정의하고 있었다.

```sh
# /etc/nginx/nginx.conf
# main context
...
events {
  # events context
  ...
}

http {
  # http context
  include /path/to/nginx.conf
  ...
}

# /path/to/nginx.conf
upstream @upstream1 {
  # upstream context
  server proxy_server1;
  server proxy_server2;
  ...
}
...

server {
  # server context
  ...
}
...
```

그래서 설정 테스트 시 main 컨텍스트에 위치할 수 없는 upstream 디렉티브가 있어서 오류가 났다.

event, http, server, location, upstream의 named 디렉티브 중심으로 nginx 설정의 큰 구조를 살펴봤다. 각 컨텍스트가 위치해야하는 곳과 컨텍스트에서 사용 가능한 또는 상속할 수 있는 디렉티브가 미리 설계되어 있다. 하지만 nginx 설정 구조를 더 단순화하면 그저 디렉티브의 트리 구조이다. 블록 디렉티브일 경우 하위 디렉티브들을 포함하는 디렉티브의 트리구조가 루트(main 컨텍스트)부터 계속 이어지는 단순한 구조이다. 이 점을 이용해 nginx 설정을 JSON으로 파싱할 수 있다. 또 이를 이해하고 있으면 파싱한 JSON을 분석하는데 활용할 수 있다.

또 문서를 참고하여 디렉티브를 확인하다 보니 같은 컨텍스트라도 속한 모듈이 다 달랐다(그리고 모듈은 디렉티브의 상속 여부와도 연관 있어 보였다). 다음엔 nginx 모듈과 디렉티브 상속에 대해서도 정리해봐야겠다.

## 참고
- [Understanding the Nginx Configuration File Structure and Configuration Contexts](https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts#apply-directives-in-the-highest-context-available)
- http://nginx.org/en/docs/ngx_core_module.html
- [[NGINX] 꼭 알아야 할 configuration 기초 개념!](https://gonna-be.tistory.com/20)
- https://docs.nginx.com/nginx/admin-guide/basic-functionality/managing-configuration-files/
