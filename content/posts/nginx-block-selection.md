---
title: "nginx 블록 선택 알고리즘"
date: 2022-02-19T17:07:52+09:00
tags:
- nginx
---

지난 포스트 [웹 서버/리버스 프록시로서 nginx 설정 구조(HTTP)](/posts/nginx-conf-structure)와 [nginx 설정 정적 분석](/posts/static-analysis-nginx-conf)에서 이어지는 내용이다.

nginx 설정은 server와 location 디렉티브를 정의하여 요청이 어떻게 처리할지 정하게 된다. 이 글은 [Understanding Nginx Server and Location Block Selection Algorithms](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)를 읽고 이해한 일부 내용을 정리한다(다 이해하진 못했는데 이 부분은 나중에 설정을 바꿔가며 직접 테스트 해보는걸로 미뤄둔다..). 

## server 블록 선택 알고리즘

server 블록은 요청에서 IP와 포트 그리고 Host 헤더 관련한 부분을 필터하여 가상 서버를 선택한다.

### listen
먼저 [listen](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 디렉티브 인자로 매치할 IP와 포트를 정의한다(소켓 경로도 인자로 받을 수 있으나 여기서 다루진 않는다). IP와 포트는 둘 다 써주거나 둘 중 하나만 써도 되는데, 안쓴것은 기본 값이 채워진다. IP의 기본 값은 `0.0.0.0`으로 호스트의 모든 인터페이스를 의미한다. 포트는 nginx가 루트로 실행중일 경우 80, 아닐 경우 8080이다.

Nginx는 요청의 IP+포트에 대해 다음처럼 server 블록 매치를 한다:
- 먼저 listen 디렉티브의 인자 IP나 포트 중 하나가 없다면 모두 기본 값으로 채운다.
- 요청의 IP+포트가 정확히 매치하는 경우를 찾는다.
  - 정확한 IP 매치 server 블록이 있는 경우 `0.0.0.0` IP의 server 블록은 매치하지 않는다.
- 정확히 매치하는 server 블록이 하나라면 그 가상 서버에서 요청을 처리,
- 아니면 그 중에서 server_name 디렉티브와 매치한다.

### server_name
[server_name](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 요청의 Host 헤더와 인자를 매치한다. server_name 인자로는 다음 값이 가능하다:

1. 도메인의 정확한 값
2. 와일드카드(`*`)로 시작하거나 끝나는 도메인 일부
3. 물결(`~`)로 시작하는 정규표현식

매치 알고리즘의 우선순위와 평가는 다음과 같다:
- 먼저 정확한 값과 매치하여 여러개가 있다면 **첫번째** 블록에서 서빙한다.
  - (e.g. `www.example.com`).
- 없다면 시작하는 와일드 카드와 매치하여 **가장 긴 매치** 블록에서 서빙한다.
  - (e.g. `*.example.com`).
- 없다면 끝나는 와일드 카드와 매치하여 **가장 긴 매치** 블록에서 서빙한다.
  - (e.g. `www.example.*`).
- 없다면 정규표현식 매치(`~`로 시작) 중 **첫번째**로 매치하는 블록에서 서빙한다.
  - (e.g. `~^(www|host1)\.example\.com$`).

원 포스트의 연습문제를 재구성해서, 직접 테스트 할 수 있도록 [쿠버네티스 클러스터](https://github.com/flavono123/nginx-the-hard-way)와 [nginx 설정](https://github.com/flavono123/nginx-the-hard-way/blob/main/conf.d/server_selection.conf) 준비했다(도커 하나면 충분할거 같지만 난 도커는 아직 잘 모르고 쿠버네티스는 할 줄 알아서 오버킬이 됐다):

```

   
root /usr/share/nginx/html/server_selection;

server {
	listen 80;
	server_name www.example.*;
	index index_trailing_wildcard.html;
}

server {
	listen 80;
	server_name *.example.com;
	index index_leading_wildcard.html;
}

server {
	listen 80;
	server_name host1.example.com;
	index index_exact.html;
}

server {
	listen 80;
	server_name ~^(www|host1).*\.example\.org$;
	index index_regex1.html;
}

server {
	listen 80;
	server_name ~^(subdomain|set|www|host1).*\.example\.org$;
	index index_regex2.html;
}
```

`curl -H "Host: <host.to.test>" <expose.service.ip>`처럼 요청해서 어떤 server 블록에서 요청이 처리되는지 테스트 해 볼 수 있다. 다음 호스트에 대해 요청해서 답을 확인하거나(응답이 server_name 인자가 되도록 했다), 설정을 보고 유추하는 식으로 연습해보자. 또는 특정 server 블록에서 처리되도록 호스트를 만드는것도 재밌을 것이다:
- `www.example.com`
- `host1.example.com`
- `www.ex**maple**.com`(오타에 유의, 왜 결과가 그런지 생각해보자)
- `www.example.org`
- `subdomain.example.org`

## location
server 블록 내에는 같은 IP, 포트, 도메인의 요청이 처리된다. URI별로 요청을 달리 처리할 때 [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) 디렉티브를 쓴다. 따라서 location 디렉티브는 URI 즉, 경로와 optional modifier로 이루어져 있다(`@`으로 시작하는 프록시 이름은 선택 알고리즘과 관련이 없어 생략한다).

Modifier는 없거나(none) 다음 네가지일 경우 뒤따라 오는 경로에 대해 각각 달리 처리한다:
- (none): 접두사(prefix) 매치
- `=`: 정확한(exact) 매치
- `~`: case-sensitive 정규표현식 매치
- `~*`: case-insensitive 정규표현식 매치
- `^~`: 정규표현식이 **아닌** 매치로 매치될 경우 다른 location과 매치하지 않고 바로 처리한다.

마지막 `^~` modifier 설명은 location 선택 우선순위와 평가 방법 그리고 location을 "jump" 하는 동작과 관련 있다.

location 선택은 다음 순서로 진행된다:
- 정규표현식을 제외한 prefix-base(exact 포함) 매치부터 한다.
  - exact 매치
  - 없으면 exact가 아닌 prefix 매치 중 **가장 긴** 매치를 찾는다.
    - `^~` modifier가 있으면 나머지 정규표현식 location을 더 찾지 않고 요청을 여기서 서빙한다.
    - `^~` modifier가 없으면 이 prefix 매치 location을 잠시 저장하고 나머지 정규표현식 location을 매치해본다.
      - 정규표현식(cs,ci 둘다) location과 매치하여 첫번째 매치하는 블록에서 서빙한다.
      - 없다면 저장해둔 prefix 매치 location에서 서빙한다.


기본적으로 nginx는 정규표현식 매치를 prefix 매치보다 우선한다(prefix 매치가 됐는데도 잠시 저장하고 정규표현식 매치를 찾으니 말이다). prefix 매치는 가장 긴 매치를 우선하지만, 정규표현식은 가장 첫번째 매치를 우선한다. prefix 매치는 정규표현식 매치보다 먼저 될 수 있지만, 다른 정규표현식 매치를 우선하느라 "jump the line" 즉, 매치된 location 블록을 벗어나 다른 location에서 처리될 수 있다. 이러한 특이한 동작과 디렉티브의 컨텍스트 상속 때문에 의도치 않은 요청 처리가 될 수 있다.

글로는 겨우 이해했다. 하지만 아직 테스트 해보진 못했다. 예제를 준비해서 [nginx-the-hard-way](https://github.com/flavono123/nginx-the-hard-way)에 업데이트 하겠다. 일단은 계속해서 위에서 언급한 "jump" 현상을 유발하는 디렉티브에 대해 알아보자.

### 다른 블록으로 점프시키는 location 내의 디렉티브

요청이 처리되는 컨텍스트 내의 디렉티브와 상속 받은 디렉티브가 요청 처리에 관여한다. 요청을 처리하던 location 컨텍스트에서 다른 컨텍스트로 점프한다면 의도치 않은 동작을 할 수 있다.

무슨 이야기인지 예시를 통해 알아보자. 먼저 이러한 점프를 트리거하는 디렉티브가 몇개 있다:
- [index](http://nginx.org/en/docs/http/ngx_http_index_module.html#index)
- [try_files](http://nginx.org/en/docs/http/ngx_http_core_module.html#try_files)
- [rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite)
- [error_page](http://nginx.org/en/docs/http/ngx_http_core_module.html#error_page)


try_files를 예로 들면:
```
root /var/www/main;
location / {
    try_files $uri $uri.html $uri/ /fallback/index.html;
}

location /fallback {
    root /var/www/another;
}
```

위 설정에서 `/blahblah` 라는 요청의 처리는:
- 첫번째 블록에서 /var/www/main/blahblah를 먼저 찾고(`$uri`)
- /var/www/main/blahblah.html을 찾고(`$uri.html`)
- /var/www/main/blahblah/ 디렉토리를 들여다보고(`$uri`)
- 마지막으로 /fallback/index.html 을 서빙한다.
- try_files가 location 탐색을 트리거하여 두번째 블록에서 처리되어 결국 /var/www/another/fallback/index.html 파일을 서빙한다.

`/blahblah`라는 파일이 /var/www/main 과 /var/www/another 아무데도 없을 경우 응답은 항상 /var/www/another/fallback/index.html 로 하게 된다. 이렇게 동작을 의도할 수도 있지만, 한눈에 봐도 설정 설계 오류처럼 보인다.

## 정리
server 블록 선택 알고리즘:
- listen 디렉티브로 가능한 모든 매치를 찾는다.
  - IP와 포트 중 없는 것을 기본 값으로 채운다.
    - IP는 `0.0.0.0`, 포트는 80(루트) 또는 8080(일반 사용자)
  - IP + 포트의 정확한 매치를 한다.
    - 정확한 매치가 여러개이면 server_name 선택으로 넘어간다.
    - `0.0.0.0` 는 IP 매치에서 후순위가 된다.
- server_name 인자와 요청의 Host 헤더 값 매치한다.
  1. 정확한 값 매치의 가장 첫번째 블록
  2. leading wildcard 매치의 가장 긴 매치 블록
  3. trailiing wildcard 매치의 가장 긴 매치 블록
  4. 정규표현식 매치(leading tilde(`~`))의 가장 첫번째 블록
  5. 기본 가상 서버(default_server) 블록

location 블록 선택 알고리즘:
- modifier 인자 문법
  - (none): prefix 매치
  - `=`: exact 매치
  - `~`: case-sensitive 정규표현식 매치
  - `~*`: case-insensitive 정규표현식 매치
  - `^~`: 비정규표현식(non-regular expression) prefix 매치, 정규표현식 매치로 넘어가지 않고 바로 요청 처리
- 블록 점프시키는 디렉티브
  - index
  - try_files
  - rewrite
  - error_page

location 선택 알고리즘에 대해, 특히 prefix 매치 후 정규표현식 매치 블록으로 점프에 대해, 이해 못한채로 글을 쓰기 시작했는데, 쓰다보니 이해가 됐다. 이건 테스트 해보고 다시 정리해야겠다. 또 이것과 관련하여 설정에서 검출할 수 있도록 이번에도 `crossplane` + `jq` 명령도 준비했는데, 오류가 있을거 같아 다시 정리할 때 한번에 올려야겠다.

---

## 참고
- http://nginx.org/en/docs/dirindex.html
- http://nginx.org/en/docs/http/request_processing.html
