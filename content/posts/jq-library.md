---
title: "jq 커스텀 함수 라이브러리화"
date: 2022-03-31T17:57:16+09:00
tags:
- jq
---


이전 글 ["jq 커스텀 함수 사용(Elasticsearch 프로퍼티 매핑 JSON 만들기)"](/posts/jq-custom-function/)에서 좀 불편한 점이 있다. `-f` 옵션을 단 하나만 쓰기 때문에 인자인 필터 파일 하나에 아주 작은 수정도 입력해야 한다. 실제로 "e" 경로를 보기 위해 `| .e`를 **파일에** 써 주었다. 길고 복잡해진 필터를 파일에 썼지만, 명령줄에서 파이프(합성)하여 쓰는 이점이 사라졌다.

이런 문제점을 해결하기 위해jq는 라이브러리(모듈) 기능을 제공한다.

["jq 필터로 JSON 정렬하기"](/posts/jq-sort-json/)에서 예로 든 JSON 정렬 필터를 예시로 들어 본다. 먼저 작업 디렉토리에 jqlib 라는 디렉토리를 만들고 아래 ./jqlib/script.jq 라는 파일에 함수로 정의했다:
```sh
❯ cat jqlib/script.jq
def jsort:
  walk(
    if type=="object" then
      to_entries | sort_by(.key) | from_entries
    else
      .
    end
  );

❯ echo '{"b": {"d": {"g": 5, "f": 4, "e": 3},"c": 2},"a":1}' | \
 jq -L jqlib 'include "script"; jsort'
{
  "a": 1,
  "b": {
    "c": 2,
    "d": {
      "e": 3,
      "f": 4,
      "g": 5
    }
  }
}

```

- 커스텀 함수를 정의하는 라이브러리 파일은 .jq 확장자로 저장한다.
- `-L` 인자는 디렉토리를 주어야 한다. 아래의 .jq 파일을 읽을 수 있다.
- `include "script"`: 라이브러리 경로 아래의, 확장자를 제외한 파일명을 주면 정의된 함수를 로드한다.


["jq 커스텀 함수 사용(Elasticsearch 프로퍼티 매핑 JSON 만들기)"](/posts/jq-custom-function/)에서 정의한 함수는 범용적이지 않다. 이를 위해 네임스페이스를 주는 모듈 기능도 제공한다. 이번엔 같은 라이브러리 디렉토리 밑에 es.jq라는 이름의 파일에ES 프로퍼티 매핑 관련 함수를 썼다:
```sh
❯ cat jqlib/es.jq
def floatorinteger:
  if tostring | test("\\.") then
    { "type": "float" }
  else
    { "type": "integer" }
  end;

def dateorstring:
  if test("\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}") then
    { "type": "date" }
  else
    {
      "type": "text",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    }
  end;

def estype:
  if type=="number" then
    floatorinteger
  elif type=="string" then
    dateorstring
  else
    { "type": type }
  end;

def esprop:
  if type=="object" then
    {"properties": with_entries(.value |= estype)}
  else
    estype
  end;

❯ echo '{"a": 1}' | \
 jq -L jqlib 'import "es" as es; with_entries(.value |= es::esprop)'
{
  "a": {
    "type": "integer"
  }
}
```

- `import "es" as es`: 라이브러리 경로 아래의, 확장자를 제외한 파일명을 module 네임스페이스에 정의된 함수를 로드한다.
- `es::esprop`: 사용할 땐 `<module>::<function>`으로 호출한다.

또 jq는 $HOME/.jq를 기본 라이브러리 경로로 로드한다. 따라서 [dotfiles에 자주 사용하는 jq 필터를 만들어 두고](https://github.com/flavono123/dotfiles/tree/master/jq/.jq) 쓰면 된다.


## 정리
- 커스텀 함수는 .jq 확장자 파일에 저장하고 jq 실행 시 라이브러리로 불러올 수 있다.
- `-L`로 라이브러리 경로를 추가하거나 $HOME/.jq에서 기본으로 불러올 수 있다.
- `include "<file>"`은 전역에 함수를 로드한다.
- `import "<file>" as <module>`로 module 네임스페이스에 함수를 로드한다.

---

## 참고
- https://stedolan.github.io/jq/manual/v1.6/#Modules
- https://github.com/stedolan/jq/issues/461
- https://stackoverflow.com/questions/71660127/jq-file-format-as-filter-f-vs-library-l?noredirect=1#comment126649007_71660127
