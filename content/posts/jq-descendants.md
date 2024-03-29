---
title: "jq descendants"
date: 2022-02-14T12:08:56+09:00
tags:
- jq
---

JSON을 다루다 보면 객체가 중첩(nested), 재귀적인(recursive) 구조가 있다. 객체 안엔 자식 객체를 담는 배열 애트리뷰트가 있고, 그 배열이 비어 있거나 또는 애트리뷰트(= 키 자체)가 없는 경우까지 반복 된다:

```json
{
  "key1": "target",
  "key2": "val2",
  "children": [
    {
      "key1": "target",
      "key2": "val4",
      "children": [
        {
          "key1": "target",
          "key2": "val6",
          "children": [],
          "__comment": "더 이상 children이 없거나"
        },
        {
          "key1": "val7",
          "key2": "val8" 
          "__comment": "아예 children 키가 없다"
        },
      ]
    },
    {
      "key1": "val9",
      "key2": "val10" 
    },
  ]
}
```

이 때 특정 조건을 만족하는 객체만 필터해야 할 때가 있다. [jq의 recursive descdent](https://stedolan.github.io/jq/manual/#RecursiveDescent:..)(`..`)를 사용하면 간단히 할 수 있다. key1의 값이 target인 객체만 필터해보자:
```sh
❯ cat descendants.json   | jq '.. | select(.key1? == "target")'
{
  "key1": "target",
  "key2": "val2",
  "children": [
    {
      "key1": "target",
      "key2": "val4",
      "children": [
        {
          "key1": "target",
          "key2": "val6",
          "children": [],
          "__comment": "더 이상 children이 없거나"
        },
        {
          "key1": "val7",
          "key2": "val8",
          "__comment": "아예 children 키가 없다"
        }
      ]
    },
    {
      "key1": "val9",
      "key2": "val10"
    }
  ]
}
{
  "key1": "target",
  "key2": "val4",
  "children": [
    {
      "key1": "target",
      "key2": "val6",
      "children": [],
      "__comment": "더 이상 children이 없거나"
    },
    {
      "key1": "val7",
      "key2": "val8",
      "__comment": "아예 children 키가 없다"
    }
  ]
}
{
  "key1": "target",
  "key2": "val6",
  "children": [],
  "__comment": "더 이상 children이 없거나"
}

```

## Recursive Descent: `..`

> Recursively descends ., producing every value. This is the same as the zero-argument recurse builtin (see below). This is intended to resemble the XPath // operator. Note that ..a does not work; use ..|.a instead. In the example below we use ..|.a? to find all the values of object keys "a" in any object found "below" ..

`..`는 현재 입력(`.`)의 모든 자손(descendants)의 값을 출력한다. **XPath를 써본 사람이라면, `//` 연산자와 비슷한 역할을 한다는 설명도 도움이 된다**:
```sh
❯ cat descendants.json   | jq '..'
{
  "key1": "target",
  "key2": "val2",
  "children": [
    {
      "key1": "target",
      "key2": "val4",
      "children": [
        {
          "key1": "target",
          "key2": "val6",
          "children": [],
          "__comment": "더 이상 children이 없거나"
        },
        {
          "key1": "val7",
          "key2": "val8",
          "__comment": "아예 children 키가 없다"
        }
      ]
    },
    {
      "key1": "val9",
      "key2": "val10"
    }
  ]
}
"target"
"val2"
[
  {
    "key1": "target",
    "key2": "val4",
    "children": [
      {
        "key1": "target",
        "key2": "val6",
        "children": [],
        "__comment": "더 이상 children이 없거나"
      },
      {
        "key1": "val7",
        "key2": "val8",
        "__comment": "아예 children 키가 없다"
      }
    ]
  },
  {
    "key1": "val9",
    "key2": "val10"
  }
]
{
  "key1": "target",
  "key2": "val4",
  "children": [
    {
      "key1": "target",
      "key2": "val6",
      "children": [],
      "__comment": "더 이상 children이 없거나"
    },
    {
      "key1": "val7",
      "key2": "val8",
      "__comment": "아예 children 키가 없다"
    }
  ]
}
"target"
"val4"
[
  {
    "key1": "target",
    "key2": "val6",
    "children": [],
    "__comment": "더 이상 children이 없거나"
  },
  {
    "key1": "val7",
    "key2": "val8",
    "__comment": "아예 children 키가 없다"
  }
]
{
  "key1": "target",
  "key2": "val6",
  "children": [],
  "__comment": "더 이상 children이 없거나"
}
"target"
"val6"
[]
"더 이상 children이 없거나"
{
  "key1": "val7",
  "key2": "val8",
  "__comment": "아예 children 키가 없다"
}
"val7"
"val8"
"아예 children 키가 없다"
{
  "key1": "val9",
  "key2": "val10"
}
"val9"
"val10"
```

루트 JSON의 모든 값을 순회하여 출력한다. 그래서 객체가 아닌 값(예제에선 문자열)을 만나면 `.key1` 키 인덱스 식별자에 respond 할 수 없어 에러가 난다:

```sh
❯ cat descendants.json   | jq '.. | .key1'
"target"
jq: error (at <stdin>:27): Cannot index string with string "key1"
```

따라서 위와 같은 목적으로 사용할 땐 [옵셔널 인덱스 식별자로 `?`](https://stedolan.github.io/jq/manual/#OptionalObjectIdentifier-Index:.foo?)를 꼭 붙여주는 것이 좋다.

## 정리
nginx 설정 파일을 JSON으로 파싱해 정적 분석할 때 알게 된 내용이다. 이건 따로 정리해서 포스팅할 예정이다.

JSON 그리고 jq는 단순함의 미학이 있는데 막상 잘 사용을 못해서 직접 쓰면 열 받는다(?)(그리고 쓰고 나니, JSON이 '어떤 키에 특정 값 타입이 반복된다' 하는 특정 스키마에 국한된 내용이 아니다. `..` 연산자가 JSON 모든 값을 순회하고 키 식별자를 선택적(`?`)으로 lookup하기 때문이다).

항상 매뉴얼을 켜두고 쓰자. 또 지금처럼 연산자나 필터 함수에 대한 토막 내용을 정리하면 나아지리라 믿는다. 그리고 대충 내가 원하는걸 검색하면 스택오버플로에 비슷한 고민을 하는 사람들이 많다는걸 볼 수 있다. 거기서도 도움을 얻자!

## 참고
- https://stedolan.github.io/jq/manual/#RecursiveDescent:..
- [How to recurse with jq on nested JSON where each object has a name property?](https://stackoverflow.com/questions/43946092/how-to-recurse-with-jq-on-nested-json-where-each-object-has-a-name-property)
