---
title: "맥에서 kubectl bash 자동 완성 기능 켜기"
date: 2022-02-17T16:06:33+09:00
tags:
- macos
- bash
- kubectl
---

[쿠버네티스 문서 bash 자동완성](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#bash) 안내처럼 `$HOME/.bash_profile`에 코드를 추가했는데, 자동완성이 되질 않는다:
```sh
❯ tail -3 ~/.bash_profile
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
❯ k ap-bash: completion: function `__start_kubectl' not found # k apply 치고 싶어!
```

구글링 결과 [다른 문서](https://kubernetes.io/ko/docs/tasks/tools/included/optional-kubectl-configs-bash-mac/)를 찾게 됐다. 안되는 이유는 맥 시스템의 기본 bash 버전에선 자동완성이 지원되지 않기 때문이다. 

아마 위와 같은 문제를 겪고 있다면 bash 버전은 4.1 미만, 맥 시스템 bash(/bin/bash)를 쓰고 있을 감능성이 높다:
```sh
❯ echo $BASH_VERSION $SHELL
3.2.57(1)-release /bin/bash
```

먼저 최신 bash를 brew를 써서 받는다:
```sh
❯ brew install bash
```

다운 받은 최신 bash를 기본 셸로 사용하기 위해 /etc/shells에 추가해준다:
- 시스템 bash(/bin/bash)와 brew로 받은 bash(/usr/local/bin/bash) 실행 파일은 다르다.
- brew 받은 파일은 저런데에 있다(`brew --prefix bash # /usr/local/opt/bash `)
```sh
❯ cat /etc/shells 
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/dash
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
/usr/local/bin/bash
```

chsh로 기본 셸을 바꾸고 새 bash 접속한다:
```sh
❯ chsh -s /usr/local/bin/bash
Changing shell for hansuk.
Password for hansuk: 

❯ echo $BASH_VERSION $SHELL
5.1.16(1)-release /bin/bash
```

bash-completion v2를 설치하고 caveats을 실행한다:
```sh
❯ brew install bash-completion@2
...
==> Caveats
Add the following line to your ~/.bash_profile:
  [[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
...
```

caveat과 함께 kubectl bash 자동완성 cheatsheet 코드도 $HOME/.bash_profile에 써주면 셸 로그인 시 자동완성이 된다:
```sh
# $HOME/.bash_profile
# Bash completion v2
[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"

# k8s alias and auto complete
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
```

```sh
❯ k ap # <tab> <tab>
api-resources  api-versions   apply 
```


👍

## 참고
- https://apple.stackexchange.com/questions/224511/how-to-use-bash-as-default-shell
