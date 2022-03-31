---
title: "jq 커스텀 함수 사용(Elasticsearch 프로퍼티 매핑 JSON 만들기)"
date: 2022-03-30T08:07:45+09:00
tags:
- jq
- elasticsearch

---

[Elasticsearch의 인덱스된 문서 필드에 데이터 타입이 동적으로 매핑이 된다](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html). 하지만 인덱스 크기를 제한하기 위해 회사에선 [명시적 매핑](https://www.elastic.co/guide/en/elasticsearch/reference/current/explicit-mapping.html)을 사용하고 있다.

그리고 회사에서 ES는 대부분 로그를 검색하는 용도로 쓰기 위해 사용하고 있다. 즉, ES 문서가 로그이다. 로그는 대부분 JSON이다. 그리고 매핑의 바디 역시 JSON이다. 따라서 내가 만들려는 것은 다음 같은 JSON 입출력의 jq 필터이다:

```sh
# 입력
❯ cat input.json | jq
{
  "a": 11,
  "b": 0.1
}

# 출력
❯ cat output.json | jq
{
  "a": {
    "type": "integer"
  },
  "b": {
    "type": "float"
  }
}
```

ES 매핑의 문법을 자세히 모르더라도 이해할 수 있도록 예시 JSON을 구성했다. [ES 데이터 타입](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)이나 [매핑 파라미터](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html)를 모르더라도 위 입출력을 이해하는데 어렵지 않다.

앞으로 글에 나올 필터는 내가 실제로 사용하는 필터이다. 하지만 모든 ES 매핑 문법을 커버하지 않는다. 즉, JSON 문서로부터 특수한 ES 매핑을 만드는 필터이다.  따라서 ES 문서로 JSON을 쓰고 있다해서 이 필터를 그대로 복붙해 쓴다면 원하는 매핑 바디가 안 나올 수 있다는 이야기이다.

또 글은 jq 자체 기능에 대해서도 다룬다. 커스텀 함수 정의와 사용은, 예제가 되는 ES JSON 문서와 매핑 바디라는 경우와 무관하게, JSON 입출력으로 하는 필터를 만드는 것에 대한 일반적인 이야기이다.


## 숫자형 처리

예시로 쓸 입력 JSON은 다음과 같다:
```sh
❯ cat input.json | jq
{
  "a": 11,
  "b": 0.1,
  "c": "flavono123",
  "d": "2022-03-28T05:01:37.768Z",
  "e": {
    "f": "darkestclimber",
    "g": "2022-03-01T04:38:14+09:00"
  }
}
```

먼저 "a",  "b" 키에 대해서 앞선 출력처럼 만들려고 한다. jq의 데이터 타입과 ES의것은 서로 비슷한 점이 있다. 그래서 jq `type` 빌트인 함수를 활용한다. 단순하게 모든 값에 대해 필터만 하더라도:
```sh
❯ cat input.json | jq 'with_entries(.value |= type)'
{
  "a": "number",
  "b": "number",
  "c": "string",
  "d": "string",
  "e": "object"
}
```

그럴싸한 결과가 나온다. `|=` update assignment로써 오른쪽 절로 값(`.value`)를 대체한다.

우린 "a", "b"가 같은 number가 아닌 integer와 float으로 구분하고 싶다. 앞서 말한듯 [ES의 숫자형 필드 타입은 훨씬 많지만](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html), 간단한 경우만 다룬다. 따라서 구두점(`.`)의 존재 여부로 float과 integer를 나눌 것이다:

```sh
❯ cat input.json | jq 'with_entries(.value |= if tostring | test("\\.") then "float" else "integer" end)'
{
  "a": "integer",
  "b": "float",
  "c": "integer",
  "d": "float",
  "e": "integer"
}
```

- `tostring`: 먼저 숫자 값을 문자열로 바꿔준다.
- `test("\\.")`: 바꾼 문자열이 구두점을 포함하는지 확인한다.
  - 백슬래시를 이스케이프 해주어야 한다(=두개를 써주었다). 이는 쉘에서 [PCRE](https://www.pcre.org/) 동작과 관련 있을듯 한데 나중에 알아봐야겠다.

"a"와 "b"는 잘 구분됐지만, 원치 않는 "c", "d", "e"도 숫자형 타입이 됐다. jq 타입이 number인것에 대해서만 처리하자:
```sh
❯ cat input.json | jq 'with_entries(.value |= if type=="number" then if tostring | test("\\.") then "float" else "integer" end else type end)'
{
  "a": "integer",
  "b": "float",
  "c": "string",
  "d": "string",
  "e": "object"
}

```

중첩 if 문이 되었다. 너무 복잡하고 너무 길다. 쉘 명령줄에서 작업하는것보다 이제 파일을 쓰는게 낫다:

```sh
❯ cat script
with_entries(.value |= if type=="number" then if tostring | test("\\.") then "float" else "integer" end else type end)

❯ cat input.json | jq -f script
{
  "a": "integer",
  "b": "float",
  "c": "string",
  "d": "string",
  "e": "object"
}
```

그래도 이중 if문의 복잡함은 해결되지 않았다. 단계적으로 개행을 하고 함수로 분리한다. 매 수정마다 `jq -f`로 테스트 해본다(이 과정은 생략했다):
```sh
# 개행
❯ cat script
with_entries(.value |=
  if type=="number" then
    if tostring | test("\\.") then
      "float"
    else
      "integer"
    end
  else
    type
  end)

# 이중 if문 전체 함수로 분리
❯ cat script
def estype:
  if type=="number" then
    if tostring | test("\\.") then
      "float"
    else
      "integer"
    end
  else
    type
  end;

with_entries(.value |= estype)

# 한번 더 함수로 분리
❯ cat script
def floatorinteger:
  if tostring | test("\\.") then
    "float"
  else
    "integer"
  end;

def estype:
  if type=="number" then
    floatorinteger
  else
    type
  end;

with_entries(.value |= estype)
```

jq는 여러 공백과 개행 모두 하나의 delimeter로 파싱한다. 따라서 한줄에 길게 썼던 필터 공백마다 개행을 하거나 탭으로 가독성을 높일 수 있다.

jq 함수 정의는 `def <function> ... end;`와 같이 한다. 여기서 정의한것처럼 arity가 0인 함수는 입력이 있다고 가정하고 그 입력을 처리한다.

## 문자형 처리
"c", "d" 그리고 "e.f", "e.g" 는 jq의 string 타입이다. 하지만 "d"와 "e.f" 모양을 보면 타임스탬프이다:
```sh
❯ cat input.json | jq '[.d, .e.g][]'
"2022-03-28T05:01:37.768Z"
"2022-03-01T04:38:14+09:00"
```

 ES에선 이를 [date 타입](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html)으로 저장하면 검색, 연산, 집계에 타임스탬프로써 편하게 쓸 수 있다. 따라서 이러한 "date"로 만들어 주자:
```sh
❯ cat script
def floatorinteger:
  if tostring | test("\\.") then
    "float"
  else
    "integer"
  end;

def dateorstring:
  if test("\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}") then
    "date"
  else
    type
  end;

def estype:
  if type=="number" then
    floatorinteger
  elif type=="string" then
    dateorstring
  else
    type
  end;

with_entries(.value |= estype)

❯ cat input.json | jq -f script
{
  "a": "integer",
  "b": "float",
  "c": "string",
  "d": "date",
  "e": "object"
}
```

- `dateorstring`: if문이 생길 곳을 미리 함수로 분리했다.
- `test("\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}")`: 문자열이 'yyyy-mm-ddThh:MM:ss' 형식과 매치하는지 검사한다. 모든 [ISO 8601](https://ko.wikipedia.org/wiki/ISO_8601) 정규식과 매치하지 않았다. 타임존도 고려하지 않는다.


ES에서 문자열은 타입 이름은 [text](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html)이다. ES는 전문 검색을 지원하기 때문에 text는 단순한 문자열이 아니다. ES가 전문(full text)을 처리하는 것에 대한 설명은 글 주제에서 벗어나 생략한다.

이 text 필드 전체를 문자열(string)처럼 쓰기 위해 아래에 [keyword](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html) 필드를 정의한다. 전문이 아닌 로그의 한 필드 문자열로써 괜찮은 처리이다. 따라서 text 필드는 `{"type": "float"}`처럼 한 객체로 필드 매핑 정의하지 않고 다음처럼 보다 길다:

```json
{
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword",
      "ignore_above": 256
    }
  }
}
```

`dateorstring`에서 date가 아닌 경우 이 객체를 반환해주자. 이름도 `dateortext`로 바꾼다. 이에 맞춰 `floatorinteger`나 `estype`에서 타입 문자열을 "type" 키의 값에 넣은 객체를 반환한다:
```sh
❯ cat input.json | jq -f script
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

with_entries(.value |= estype)

❯ cat input.json | jq -f script
{
  "a": {
    "type": "integer"
  },
  "b": {
    "type": "float"
  },
  "c": {
    "type": "text",
    "fields": {
      "keyword": {
        "type": "keyword",
        "ignore_above": 256
      }
    }
  },
  "d": {
    "type": "date"
  },
  "e": {
    "type": "object"
  }
}
```

## 객체 처리
값이 객체인 "e"는 단순히 object라고 출력하고 있다. 회사에선 이런 하위 필드도 접근할 수 있도록 각각의 프로퍼티로 매핑하고 있다(반대로 지금처럼 객체 자체로써 매핑하고 싶다면 [nested](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html) 타입을 매핑해야할 것이다).

이번에 할 일은 값의 타입이 객체(object)일 경우 "properites" 키 아래에 각 객체 타입을 매핑한 2중 JSON을 반환하는 것이다. 위의 결과는 중복되기 때문에 "e" 키만 결과를 출력한다:
```sh
❯ cat script
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

with_entries(.value |= esprop) | .e


❯ cat input.json | jq -f script
{
  "properties": {
    "f": {
      "type": "text",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    },
    "g": {
      "type": "date"
    }
  }
}
```


jq에서 커스텀 함수를 정의하고 파일에 필터를 가져와 ES 매핑 JSON을 만들었다. 다음 포스팅에선 정의한 커스텀 함수를 라이브러리로 사용하는 법을 소개할 것이다. 원랜 같이 하려고 했지만 분량 조절에 실패했다 😹.


## 정리
- jq 커스텀 함수는 필터에 `def <function>[(arg1, [...])] ... end;` 로 정의한다.
- jq 필터에서 개행을 포함한 공백들은 하나의 delimeter로 파싱한다.
  - 따라서 `-f` 옵션과 함께 파일에 필터를 쓰면 가독성 좋게 포매팅 할 수 있다.
  - 합성 필터가 길어지거나, 특히 커스텀 함수 정의가 길면 파일을 쓰는것이 좋다.


---

## 참고
- https://stedolan.github.io/jq/manual/v1.6/
