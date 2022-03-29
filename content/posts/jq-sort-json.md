---
title: "jq 필터로 JSON 정렬하기"
date: 2022-03-29T20:16:40+09:00
tags:
- jq
---

```sh
❯ jq --version
jq-1.6
```

jq를 써 JSON을 정렬해보자. 정렬은 키를 기준으로 한다.

쿠버네티스 정의 파일을 편집하면 키 순서대로 정렬되어 있는것을 볼 수 있다(보통은 JSON이 아닌 YAML로 보긴한다). 긴 파일에서 키가 정렬되어 있는 것은 한눈에 보기에 편하다. 그래서인지 언제부턴가 다음 같은 JSON은 보기 불편하다. 예시로 쓸 JSON은 짧긴 하다:

```sh
❯ echo '{"b":2,"a":1}' | jq
{
  "b": 2,
  "a": 1
}
```

## 중첩 없는 JSON 정렬하기

키를 기준으로 정렬할 것이다. jq엔 객체(JSON)의 키를 출력하는 `keys`와 배열을 정렬하는 `sort`라는 빌트인 함수가 있다:

```sh
❯ echo '{"b":2,"a":1}' | jq 'keys | sort'
[
  "a",
  "b"
]

```

하지만 정렬된 키 배열이 아니라 키 순서로 정렬된 원래의 JSON이 필요하다. 먼저 `to_entries` 라는 빌트인 함수로 JSON의 키와 값을 하나의 객체로 하는 배열을 만들어 준다:
```sh
❯ echo '{"b":2,"a":1}' | jq 'to_entries'
[
  {
    "key": "b",
    "value": 2
  },
  {
    "key": "a",
    "value": 1
  }
]
```

`sort_by` 빌트인 함수 인자에 키 경로를 주어 이 배열을 정렬한다:
```sh
❯ echo '{"b":2,"a":1}' | jq 'to_entries | sort_by(.key)'
[
  {
    "key": "a",
    "value": 1
  },
  {
    "key": "b",
    "value": 2
  }
]

```

그리고 키 값 객체의 배열 즉, 엔트리 배열을 `from_entries`로 다시 객체로 만들 수 있다:
```sh
❯ echo '{"b":2,"a":1}' | jq 'to_entries | sort_by(.key) | from_entries'
{
  "a": 1,
  "b": 2
}

```

중첩이 없는, 1 계층의 JSON의 정렬은 아주 간단하다. 하지만 값에 객체가 있는 JSON의 경우 모든 JSON이 정렬되지 않는다:

```sh
❯ echo '{"b": {"d": 3,"c": 2},"a":1}' | jq 'to_entries | sort_by(.key) | from_entries'
{
  "a": 1,
  "b": {
    "d": 3,
    "c": 2
  }
}
```

## 중첩 JSON 정렬하기

맨 위에서 볼 수 있는 `keys` 뿐만 아니라, 중첩된 객체의 키도 순서대로 정렬하고 싶다. 엔트리를 순회하다가 값(`.value`)이 객체일 경우 위에서 한 정렬과 같은 동작을 재귀적으로 하고 싶다. 이를 위한 빌트인 함수 `walk`가 있다. 일단 결과를 보자:

```sh
✦ ❯ echo '{"b": {"d": 3,"c": 2},"a":1}' | \
 jq 'walk(if type=="object" then
   to_entries | sort_by(.key) | from_entries
 else
   .
 end)'
{
  "a": 1,
  "b": {
    "c": 2,
    "d": 3
  }
}
```

if-else 문이 추가되긴 했지만, if 에서 실행하는 합성 함수는 위에서 쓴 JSON 정렬과 같다. if 문은 잠시 잊고, `walk` 가 하는 일을 보자:

>  **walk(f)**

>  The walk(f) function applies f recursively to every component of the input entity. When an array is encoun-
       tered, f is first applied to its elements and then to the array itself; when an object is encountered, f is
       first applied to all the values and then to the object. In practice, f will usually test the  type  of  its
       input,  as illustrated in the following examples. The first example highlights the usefulness of processing
       the elements of an array of arrays before processing the array itself. The second example shows how all the
       keys of all the objects within the input can be considered for alteration.

           jq 'walk(if type == "array" then sort else . end)'
              [[4, 1, 7], [8, 5, 2], [3, 6, 9]]
           => [[1,4,7],[2,5,8],[3,6,9]]

           jq 'walk( if type == "object" then with_entries( .key |= sub( "^_+"; "") ) else . end )'
              [ { "_a": { "__b": 2 } } ]
           => [{"a":{"b":2}}]

`walk`는 입력의 모든 컴포넌트에 **재귀적**으로 실행한다. 좀 더 자세히 설명하면, 배열의 경우 배열의 원소에 모두 함수 `f`를 실행한 후 배열 자체에 함수 `f`를 실행한다. 객체의 경우 객체 모든 값에 함수 `f`를 실행한 후 객체 자체에 함수 `f`를 실행한다.

아래의 두 예제 중 첫번째를 보면, 바깥 배열의 원소인 안쪽 배열이 각각 정렬되어 있고 바깥 배열(itself)도 안 배열의 가장 앞 원소에 따라 정렬되어 있다. 바깥 배열의 정렬은 순서를 조금 바꿔주면 명확하게 볼 수 있다:
```sh
❯ echo '[[4, 1, 7], [3, 6, 9], [8, 5, 2]]' | jq -c 'walk(if type == "array" then sort else . end)'
[[1,4,7],[2,5,8],[3,6,9]]
```

즉 배열 원소에 대한 `sort`의 정렬 방식은 다음과 같다:
```sh
❯ echo '[[2], [0], [1]]' | jq -c 'walk(if type == "array" then sort else . end)'
[[0],[1],[2]]

```

또 이어서 설명하듯 보통 `f` 함수는 입력의 타입을 검사하는걸로 시작하게 된다. 우리의 예제는 JSON을 정렬하는 것이라, 배열과 객체 중 객체에 대해서만 정렬(`to_entries | sort_by(.key) | from_entries`)을 할 것이라 객체 타입을 검사하는 if문을 썼다.

그런데 `walk`는 아주 최신 버전인, 1.6 버전의 빌트인 함수이다. 1.5 버전엔 없다. [데비안을 비롯한 리눅스 패키지와 윈도우 패키지에선 1.5 이하가 최신 버전이다(맥이라서 1.6버전을 쓸 수 있었다)](https://stedolan.github.io/jq/download/). 따라서 바이너리를 받아 1.6 버전을 쓰지 않는다면 `walk`를 구현해야 한다.

`walk`의 구현은 jq의 자체의 함수 선언에서 가능하다. 다음은 [실제 1.6 버전의 빌트인 `walk`의 구현](https://github.com/stedolan/jq/blob/master/src/builtin.jq#L273-L280)이다:
```jq
def walk(f):
  . as $in
  | if type == "object" then
      reduce keys_unsorted[] as $key
        ( {}; . + { ($key):  ($in[$key] | walk(f)) } ) | f
  elif type == "array" then map( walk(f) ) | f
  else f
  end;
```

앞선 매뉴얼의 설명과 일치하고 꽤 간결하다. 단순히 2 계층에 대해서가 아니라 완전히 재귀적으로 동작하는가 확인해본다:
```sh
❯ echo '{"b": {"d": {"g": 5, "f": 4, "e": 3},"c": 2},"a":1}' | \
 jq 'walk(if type=="object" then
   to_entries | sort_by(.key) | from_entries
 else
   .
 end)'
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

## 옵션 `--sort-keys`

뿌듯하게 이 글을 마무리하는 중, 매뉴얼을 읽다가 간단한 옵션을 발견했다:
```sh
❯ echo '{"b": {"d": {"g": 5, "f": 4, "e": 3},"c": 2},"a":1}' | \
 jq --sort-keys
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

그렇다. 나의 구구절절 필터는 `--sort-keys` 라는 옵션 한방으로 해결 가능하다😹...

'JSON 정렬하기'라는 목적만 보면, 앞선 필터를 만들어 내는것은 비효율적이다. 단순히 글자 수만 비교해도 그렇다. 하지만 `walk`의 동작을 이해한 것, jq 필터를 프로그래매틱하게 사용한 것이라는 수확이 있었다. jq 필터를 좀 더 잘 다룰 수 있게 됐다.


## 정리
- `walk`는 인자 함수 `f`를 재귀적으로 호출한다
  - 배열은 각 원소에 `f` 실행 후 배열 자신에 `f` 실행하고
  - 객체는 각 값에 `f` 실행 후 객체 자신에 `f`를 실행한다
  - 나머지 타입은 `f` 실행만 한다
- JSON 키 정렬은 `--sort-keys`로 간단하게 할 수 있다
  - [필터를 활용하여 프로그래매틱하게 할 수도 있다](https://gist.github.com/flavono123/f71903270ab6b3d9bb823ebddac702f0)


---

## 참고
- https://stedolan.github.io/jq/manual/v1.6/
