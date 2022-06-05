---
title: "Rubocop Custom Cop"
date: 2022-06-03T15:13:34+09:00
tags:
- ruby
---

나는 [rubocop](https://github.com/rubocop/rubocop)을 정말 좋아한다. Rubocop엔 이미 내장된, lint 규칙의 단위인, cop이 정말 많다. 좀 nit-picking 같은 offense도 많지만, 왜 이런 규칙이 있을까 생각해보는게 재밌다. 예를 들어 메소드 줄 길이를 제한하는 [Metrics/MethodLength](https://docs.rubocop.org/rubocop/cops_metrics.html#metricsmethodlength)는 기본 값이 10이다(내 기억에 한때 5였고, 문서의 예시처럼 여러 줄을 score로 세지 않았던거 같다. [샌디 메츠가 따로 없었다](https://thoughtbot.com/blog/sandi-metz-rules-for-developers)).

이렇다 보니 rubocop을 업그레이드하면 수 많은 새 cop의 offenses가 나온다. 비활성화 또는 파라미터를 바꾸는 정도의 커스텀을 했다. 하지만 rubocop의 내장 cop을 조정하는게 아니라, 우리 코드에 맞는 규칙(cop)을 만들고 싶을 때도 있을 것이다. 코드 리뷰에서 자주 언급하게 되는 패턴이나 어떤 목적을 위해 코드에 강제해야하는 규칙 같은 것이 있을 수 있다. 이번에 그러한 사례가 있어 custom cop을 만들어 봤다.


## Sexp(S 표현식)
Custom cop이 가능하다는 것은, [Railsconf 2020의 한 발표](https://railsconf.com/2020/2020/video/kyle-d-oliveira-communicating-with-cops)를 보고, 이미 알고 있었지만, 노드 표현식이 좀 어렵게 느껴졌었다. 하지만 직접 만들어 보니 꽤 직관적이고 간단한 내용이다.

다른 프로그래밍 언어처럼, 루비 코드의 AST는 [S 표현식](https://en.wikipedia.org/wiki/S-expression)으로 나타낸다. 루비의 구현체는 [Sexp](https://github.com/seattlerb/sexp_processor)이다. 그리고 여기에 패턴 매치 가능한 몇몇 연산자, 변경자 등을 추가한 것이 [RuboCop AST의 노드 패턴](https://docs.rubocop.org/rubocop-ast/node_pattern.html)이다.

간단한 예시의 S표현식에서 Sexp로 확장해보자. 아마, 알고리즘 문제 같은데서 봤을 법한 표현식일 것이다:
```sh
3 + 4 * 5
```

위 수식은 다음 S 표현식로 치환된다:
```sh
(+ (* 4 5) 3)
```

괄호와 요소(element; atom이나 data로 칭하기도 한다)를 사용해 실행 순서를 표현한다. 괄호 안의 표현식은 요소(3,4,5 그리고 \*, + 같은 연산자) 또는 다른 표현식의 중첩(nested)로 이루어진다. 곱셈을 먼저 감싸 연산 우선순위가 높음을 표현했다.

`RuboCop::AST::Node`는 루비의 실제 AST(`Sexp`)와 거의 유사하다. 루비는 [ruby-parse](https://github.com/whitequark/parser)라는 명령으로 표현식을 확인할 수 있다:
```sh
❯ ruby-parse --legacy -e '3 + 4 * 5'
(send
  (int 3) :+
  (send
    (int 4) :*
    (int 5)))
```
- `--legacy`: rubocop 노드 패턴과 비슷하게 확인하기 위해 준 옵션. rubocop이 루비 버전 하위 호환성을 지원하기 위해 그런 것 같다.
- `-e`: 파일 대신 코드를 파싱

루비가 익숙한 사람은 `+, *`이 연산자가 아니라 메소드라서 send(호출) 된 것이라고 쉽게 파악할 수 있을 것이다. 또 3, 4, 5의 값은 단순 요소가 아니라 괄호로 감싸져 하나의 (Int) 노드가 됐다.

## Custom Cop 코드
위처럼 ruby-parse를 이용해 쉽게 `Sexp`를 쉽게 할 수 있다. 내가 만들려는 cop은 레일즈 캐시 사용 시 TTL([kwargs `expires_in`](https://api.rubyonrails.org/classes/ActiveSupport/Cache/Store.html#method-i-fetch))을 제한하고 싶었다. 따라서 다음 명령으로 Sexp를 먼저 만들었다:
```sh
❯ ruby-parse --legacy -e "Rails.cache.fetch('keyr', expires_in: 10.minutes)"
(send
  (send
    (const nil :Rails) :cache) :fetch
  (str "key")
  (hash
    (pair
      (sym :expires_in)
      (send
        (int 10) :minutes))))
```

이를 바탕으로 Custom cop의 노드 매처(matcher) 패턴을 만든다. 이를 포함한 custom cop(`MaxCacheTTL`) 코드를 전체는 다음과 같다. 나머지 코드는 간단하다:
```ruby
class MaxCacheTTL < RuboCop::Cop::Base
  def_node_matcher :max_cache_ttl, <<~PATTERN
    (send
      (send
        (const nil? :Rails) :cache) :fetch
        < (hash
            (pair
              (sym :expires_in) $_ )
          ) ... > )
  PATTERN

  MAX_CACHE_TTL_SEC = 300
  RESTRICT_ON_SEND = [:fetch]
  MSG = "캐시 TTL은 #{MAX_CACHE_TTL_SEC}초가 넘으면 안됨."

  def on_send(node)
    max_cache_ttl(node) do |ttl_node|
      expires_in_seconds = case ttl_node
                           when RuboCop::AST::SendNode
                             method_name = ttl_node.method_name
                             if method_name == :minutes || method_name == :minute
                               ttl_node.receiver.value * 60
                             else # seconds
                               ttl_node.receiver.value
                             end
                           when RuboCop::AST::IntNode
                             ttl_node.value
                           end

      add_offense(ttl_node) if expires_in_seconds > MAX_CACHE_TTL_SEC
    end
  end
end
```

def_node_matcher의 두번째 인자, 노드 매처 패턴을 먼저 설명한다. 위에 ruby-parse로 출력한 노드와 노드 패턴의 눈의 띄는 차이점은 다음과 같다:
- `nil?`: 특수 문자. [메소드 호출자(caller)가 없음을 뜻한다](https://docs.rubocop.org/rubocop-ast/node_pattern.html#nil-or-nil). nil 노드 타입과 반복자 ?의 조합이 아님에 주의해야 한다.
- `< >`: [안의 노드들의 순서 상관 없이 매치한다.](https://docs.rubocop.org/rubocop-ast/node_pattern.html#for-match-in-any-order) 지금처럼 특정 메소드 인자 노드를 매칭할 때, 나머지 인자 노드 매치하는 용도로 자주 사용된다.
- `$_`: [노드를 캡쳐한다](https://docs.rubocop.org/rubocop-ast/node_pattern.html#for-captures).
- `...`: [여러 노드와 매치한다.](https://docs.rubocop.org/rubocop-ast/node_pattern.html#for-several-subsequent-nodes)

나머지 코드는 쉽게 이해할 수 있을 것이다. Offense 조건을 검출하기 위해 [`Numeric#minutes(ActiveSupport)`](https://api.rubyonrails.org/classes/Numeric.html#method-i-minutes)만 검사해 초로 환산했는데, 실제로 코드에서 minutes만 쓰고 있었기 때문이다(`#fetch`만 검사하는 것도 같은 이유이다). 무겁게 active_support를 로드하지 않고 간단하게 검사했다.

Offense에 걸린 결과는 다음과 같다. Custom cop 코드 수정을 반영하려면 캐시를 끄고 rubocop 실행하는 것이 낫다(캐시는 린트하는 파일이 대상이다):
```sh
❯ cat cache_test.rb
Rails.cache.fetch('key', 10.minutes)

❯ rubocop --only MaxCacheTTL --cache false cache_test.rb

Offenses:

cache_test.rb:387:36: C: MaxCacheTTL: 캐시 TTL은 300초가 넘으면 안됨.
Rails.cache.fetch('key', expires_in: 10.minutes)
                                     ^^^^^^^^^^

1 file inspected, 1 offense detected
```

비록 이 코드는 회사 레포에 병합이 안됐지만, rubocop custom cop을 잘 활용하면 지속 통합에서 큰 효과를 볼 수 있을 것이다.

추가로 최신 버전 rubocop에선 custom cop 정의를 모듈(department)로 감쌀 것을 경고한다(Warning: no department given for MaxCacheTTL.). 네임스페이스로 관리하기 위해서이다.


## 정리
- Sexp는 요소와 노드로 표현으로 프로그래밍 언어 AST를 표현하는데 사용된다.
- ruby-parse로 루비 코드 AST Sexp를 알 수 있다.
- Rubocop 노드 패턴을 만들어 custom cop을 만들 수 있다.


---

## 참고
- https://thoughtbot.com/blog/rubocop-custom-cops-for-custom-needs
- https://en.wikipedia.org/wiki/S-expression
- https://docs.rubocop.org/rubocop-ast/index.html
