---
title: "nginx location 점프"
date: 2022-02-25T12:33:29+09:00
tags:
- nginx
---

지난 포스트, [nginx 블록 선택 알고리즘](/posts/nginx-block-selection), 에서 테스트와 설명에 자신감이 없던 부분에 대한 보충 설명이다. [Understanding Nginx Server and Location Block Selection Algorithms](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)를 참고하여 location 컨텍스트 내에서 다시 location 탐색을 유발하는 디렉티브 네개를 소개했다:
- index
- try_files
- rewrite
- error_page

이 중 세개는 테스트해봤다. 첫번째인 index 디렉티브는 검증하지 못했다. 디지털 오션 참고 글엔 location 컨텍스트가 비어 있어, index 디렉티브를 상속 받는게 아니라 각각 써주었다:
```
...
  location = /exact {
    index index.html;
  }

  location / {
    index index.html;
  }
...
```

하지만 의도한대로 동작하지 않았다. /exact/ 로 요청하면, /exact/index.html 이 서빙됐다. [nginx 문서](http://nginx.org/en/docs/http/ngx_http_index_module.html#index)의 설명도 점프에 대해 이야기 하지만, 예제는 더 모호한 것이라 시도하진 않았다.


try_files는 성공했다. [문서 설명](http://nginx.org/en/docs/http/ngx_http_core_module.html#try_files)처럼 찾는 파일이 없으면 마지막 인자 uri로 internal redirect를 한다. Digital ocean 글에서 언급이 됐던, internal redirect는 점프를 의미한다. 이게 좀 더 공식적인 용어 같다.


rewrite는 마지막 flag 인자가 last일 경우, internal redirect가 발생할 수 있다:
> stops processing the current set of ngx_http_rewrite_module directives and starts a search for a new location matching the changed URI;

- [rewrite 문서](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite)


error_page 역시 마지막 인자 uri에 의해서 internal redirect가 발생할 수 있다:
> This causes an internal redirect to the specified uri with the client request method changed to “GET” (for all methods other than “GET” and “HEAD”).

- [error_page 문서](http://nginx.org/en/docs/http/ngx_http_core_module.html#error_page)

Digital ocean의 예제를 참고해 테스트해볼 수 있도록 lab을 구성했다([repo](https://github.com/flavono123/nginx-the-hard-way)). README에 try_files, rewrite, error_page 각각 커밋이 있는데, 여기로 checkout 하면 된다(같은 nginx 설정 파일에 커밋을 쌓아서 이렇게 했다. lab 재현 환경 문서화가 아주 깔끔한건 아니지만... 혼자서 이정도면 충분한듯)

여기서 몇가지 느낀점이 있다:
- try_files 인자 처리가 불합리(?)하다.
  - 마지막 인자를 포함해서 모두 root 인자 + location 인자 하위에서 파일을 찾는다.
  - 그런데 마지막 인자는 `/`로 시작해서 절대 경로로 보인다.
  - e.g. `try_files $uri $uri.html $uri/ /fallback/index.html;`
  - 마지막 인자 `/fallback/index.html`만 `/`로 시작하지만, 앞선 탐색과 마찬가지로, 현재 컨텍스트 디렉토리 내에서 찾는다.
- 그런데 [try_files](http://nginx.org/en/docs/http/ngx_http_core_module.html#try_files)와 [error_page 문서](http://nginx.org/en/docs/http/ngx_http_core_module.html#error_page)를 보면, 이 부분은 `uri`라는 인자로 구분되어 있다.
  - error_page는 비교할 인자가 없지만, try_files는 앞 인자인 `file`과 비교가 된다.
  - 그리고 없으면 다음으로 넘어간다는 점과, location 탐색(internal redirect)를 한다는 점도 구분된다.
- 문서의 `uri`라는 인자는 그런 특성을 갖는걸까?

이번에도 모든게 명확히 보이진 않는다. 다시 C를 공부해서 nginx 코드를 보는게 빠를까 하는 생각이 또 든다(그러나 아니란걸 알고 있다). 계속 파고 들다 보면 끝이 없으므로 일단 factsheet를 하나 정리한다:

- try_files의 마지막 인자(uri)로 fallback하면 internal redirect 할 가능성이 있다.
- rewrite를 `last` flag와 쓰면 패턴에 따라 internal redirect 할 가능성이 있다.
- error_page 조건에 맞아 uri로 fallback하면 internal redirect 할 가능성이 있다.

이번에도 파싱해서 정적으로 추적해보자:
```sh
# try_files 때문에 internal redirect 위험이 있는 location
❯ crossplane parse nginx.conf  | jq -r '.config[<n>].parsed[] | .. | select(.directive?=="location") | select(.block[].directive=="try_files")'
# rewrite ... last 때문에 internal redirect 위험이 있는 location
❯ crossplane parse nginx.conf  | jq -r '.config[<n>].parsed[] | .. | select(.directive?=="loccation") | select(.block[].directive == "rewrite" and .block[].args[-1] == "last")'
# error_page 때문에 internal redirect 위험이 있는 location
❯ crossplane parse nginx.conf  | jq -r '.config[1].parsed[] | .. | select(.directive?=="location") | select(.block[].directive=="error_page")'
```

---

nginx 왜 이리 요청을 복잡하게 처리하나 싶지만... 재밌기도 하다. 또 k8s의 공식 ingress controller라서 배워두는게 아깝진 않을거 같다(그쪽에선 요청과 서비스 매치 처리가 훨씬 간단해보였다). 아직 알게 많아서 재밌기도하다. 디렉티브 상속과 모듈에 관한 내용도 나중에 정리해봐야겠다.
