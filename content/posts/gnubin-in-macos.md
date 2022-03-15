---
title: "맥에서 GNU 프로그램 사용"
date: 2022-03-15T23:53:09+09:00
---

맥에 기본으로 설치된 CLI 프로그램은 BSD의 것이다. 그런데 그런 프로그램들 사용법을 알기 위해 검색하면 대부분 Linux에 설치된 GNU 프로그램 기준으로 설명하는 경우가 많다. 심지어 `ls` 도 옵션 사용이 조금 다르다(e.g. `-w` 옵션의 의미가 다르다):
- https://www.freebsd.org/cgi/man.cgi?ls
- https://man7.org/linux/man-pages/man1/ls.1.html

이 때문에 맥에서 참고한 같은 명령을 치고 결과가 다르거나 오류가 발생하는 경우가 꽤 있다. 따라서 맥에서도 GNU 프로그램을 사용하는게 속편한것 같다.

GNU 프로그램은 brew를 통해 받을 수 있다:
```sh
brew install autoconf bash binutils coreutils diffutils ed findutils flex gawk \
    gnu-indent gnu-sed gnu-tar gnu-which gpatch grep gzip less m4 make nano \
    screen watch wdiff wget
```

이를 기본 프로그램으로 사용하기 위해 `PATH` 변수 앞에 붙여 준다. 매뉴얼도 GNU 것이 나오도록 한다:
```sh
if type brew &>/dev/null; then
  brew_prefix=$(brew --prefix)
  for d in ${brew_prefix}/opt/*/libexec/gnubin; do export PATH=$d:$PATH; done
  for d in ${brew_prefix}/opt/*/libexec/gnuman; do export MANPATH=$d:$MANPATH; done
fi
```

---

## 참고
- https://gist.github.com/skyzyx/3438280b18e4f7c490db8a2a2ca0b9da
