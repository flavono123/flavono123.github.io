---
title: "Vim 줄에서 N번째 패턴 검색"
date: 2022-09-25T10:26:36+09:00
tags:
- vim
- regexp
---

Vim에서 줄의 n번째 패턴을 검색하는 법을 [찾아보고](https://vi.stackexchange.com/questions/8621/substitute-second-occurence-on-line) 배운 점을 정리한다. 줄의 세번째 'Fab'이라는 단어(패턴)를 찾는 방법이다:
```vim
Fab1 Fab2 **Fab**3
    Fab1 Fab2 **Fab**3 Fab4
~
~
~
/\v(.{-}\zsFab){3}
```

먼저 `\v`은 [very magic modifier](https://learnbyexample.github.io/vim_reference/Regular-Expressions.html#very-magic)로 메타 문자를 escape 없이 쓸 수 있게 도와준다. 만약 `\v`를 쓰지 않는다면 위 vim 정규표현식은 다음과 같다. `\(.\{-}\zsFab\)\{3}`(n번째 패턴을 찾는 것과는 거리가 있는, vim 정규표현식을 보기 편하게 하기 위함이다).


`{-}`은 vim의 [non-greedy](https://vimhelp.org/pattern.txt.html#non-greedy) 매칭이다.

`\zs`는 매칭 시작 패턴을 지정하는 것인데, 지금의 경우 PCRE 정규표현식에서 그룹 매치와 정확히 일치한다. 따라서 위 정규표현식을 PCRE로 바꾸면 다음과 같다:

https://regex101.com/r/GCFV1l/1

Quantifier와 non-greedy를 통해 찾고 싶은 문자로 끝나는 반복하는 패턴의 찾고 싶은 등장번째(n)로 검색하는 것이다. 따라서 2번째 패턴을 찾으려고 하면 두번째 줄에선 네번째 패턴까지 찾아 주게 된다.

```vim
Fab1 **Fab**2 Fab3
    Fab1 **Fab**2 Fab3 **Fab**4
~
~
~
/\v(.{-}\zsFab){2}
```


내가 실제로 해결하려 했던 문제는 **'YAML 값에서 ":" 찾기'** 이다. YAML 키 끝에 ":"이 있기 때문에 'YAML 파일에서 줄의 두번째 ":"를 찾기'와 같은 문제다.


Vim과 PCRE 정규표현식이 같지 않다는 것은 느끼고 있었지만, 이번에 제대로 알게 됐고 이번처럼 활용할 일은 지금까지 없었다. 그리고 아직 vim의 시작 매치(`\zs`)와 PCRE의 그룹 캡쳐(`(<...>)`)가 왜 대구를 이루는진 모르겠다. Vim의 그룹 캡쳐가 더 종류가 많아 보인다.

그래서 n번째 패턴 찾기라는 사례로 정리한다. Vim도 정규표현식 엔진의 과정, 결과를 시각적으로 풀어주는 도구가 있으면 좋을거 같다.


## 정리
- Vim에서 n번째 패턴을 찾을 때 `/\v(.{-}\zs<패턴>){<n>}`을 사용하자.


---

## 참고
- https://vi.stackexchange.com/questions/8621/substitute-second-occurence-on-line
- https://learnbyexample.github.io/vim_reference/Regular-Expressions.html
- https://vimhelp.org/
