---
title: "Stow로 dofiles 관리하기"
date: 2022-03-26T00:10:53+09:00
---

과거엔 dotfiles를 Git bare repositoy로 관리했다. 아니 관리에 실패했다. 노트북을 바꾸진 않았지만 어떤 이유로(기억이 안난다) 싹 초기화 했는데 다시 bare repo를 만들고 사용하질 않았다(개발에 관심을 덜 가졌던 시기라서도 그렇다).

최근 사용하는 쉘, 에디터, 도구들을 손보면서 dotfiles도 자주 수정하게 됐다. 싱크가 안된, 아카이브와도 같은 dotfiles 레포와 실제 dotfiles를 관리하는게 점점 부담으로 다가왔다.

다시 bare repo 사용법을 익히려 검색을 하던 중 [stow](https://www.gnu.org/software/stow/manual/stow.html)라는 프로그램을 사용하면 다른 방법으로, 쉽게 dotfiles를 관리할 수 있다는걸 알게 됐다.


## stow
stow는 기본 인자인, 패키지를 목적 디렉토리에 심링크한다. 패키지는 하나의 루트 디렉토리에서 시작하는 파일과 디렉토리 집합이다. stow로 dotfiles를 관리한다는 것은, dotfiles를 git repo에 패키지로 만들어 홈 디렉토리에 심링크하여 설치, 관리하는 것이다.

dotfiles 패키지의 예시로 vim을 살펴보자. 나의 vim dotfiles는 구조는 다음과 같다:

```sh
❯ pwd
/Users/hansuk

❯ tree -a .vim*
.vim
├── .netrwhist
├── autoload
│   └── plug.vim
├── plugged
│   └── fzf.vim
│       ├── .git
# ... 생략
└── plugin           # 이거
    ├── fzf.vim
    └── vagrant.vim
.viminfo
.viminfo.tmp
.vimrc                 # 이거
```

홈 디렉토리에서 `.vim*`에 해당하는 파일과 디렉토리 트리 구조를 출력했다. 내가 설정하는 dotfiles는 `.vimrc`와 `.vim/plug` 아래의 vim 파일들이다. 나머지는 히스토리나 플러그인 설치 시 자동으로 쓰이는, 관심 바깥의 파일들이다.

먼저 dotfiles 레포에 vim 이라는 패키지(디렉토리)를 만들어 준다. 그리고 패키지에 포함시키고 싶은 파일들을 그 아래 복사한다.
```sh
❯ cd /path/to/dotfiles
❯ mkdir vim
❯ cp ~/.vimrc vim
❯ cp -r ~/.vim/plugin vim
```

나의 vim dotfiles 패키지를 만들었다. stow를 사용하여 심링크를 거는 시뮬레이션을 해본다:

```sh
❯ stow -nv -S vim -t ~ --adopt
MV: .vim/plugin/fzf.vim -> dotfiles/vim/.vim/plugin/fzf.vim
LINK: .vim/plugin/fzf.vim => ../../dotfiles/vim/.vim/plugin/fzf.vim
MV: .vim/plugin/vagrant.vim -> dotfiles/vim/.vim/plugin/vagrant.vim
LINK: .vim/plugin/vagrant.vim => ../../dotfiles/vim/.vim/plugin/vagrant.vim
MV: .vimrc -> dotfiles/vim/.vimrc
LINK: .vimrc => dotfiles/vim/.vimrc
WARNING: in simulation mode so not modifying filesystem.

```
- `-n(--no)`: dry-run, simulate 한다(마지막 줄에서 설명한다).
- `-v(--verbose)`: 어떤 명령을 하는지 자세히 출력
- `-S(--stow)`: 패키지
- `-t(--target)`: 목적 디렉토리
- `--adopt`: 목적 디렉토리의 패키지 파일이 링크나 디렉토리가 아니더라도 덮어 씌운다. 이미 있는 파일을 심링크로 바꾸기 위해 필요

`-n`을 지우고 실행하면 stow, 즉 vim 설정 패키지를 내 홈 디렉토리에 설치(심링크)한다. `-v` 역시 옵션이다:

```sh
❯ stow -S vim -t ~ --adopt
```

이제 다시 홈 디렉토리를 살펴보면 대상 파일들이 심링크로 바뀐 것을 볼 수 있다:
```sh
❯ ll ~/.vimrc
lrwxr-xr-x 1 hansuk 19  3 26 01:44 /Users/hansuk/.vimrc -> dotfiles/vim/.vimrc

❯ ll ~/.vim/plugin/
total 0
lrwxr-xr-x 1 hansuk 38  3 26 01:44 fzf.vim -> ../../dotfiles/vim/.vim/plugin/fzf.vim
lrwxr-xr-x 1 hansuk 42  3 26 01:44 vagrant.vim -> ../../dotfiles/vim/.vim/plugin/vagrant.vim
```

새로운 파일을 패키지에 추가하려면 dotfiles 레포에 추가 후 stow하거나(이땐 `--adopt` 옵션이 필요 없을 것이다), 실제로 홈 디렉토리에 추가 후 위와 같은 방법으로 dotfiles와 동기를 맞추면 된다. dotfiles를 적용하다가 또는 프로그램이 설치하는 템플릿 같은게 있을테니 후자를 더 많이 쓰지 않을까 싶다.

또는 새 노트북에 환경을 프로비저닝(?)한다던가 과거의 나처럼 어떤 이유로 날려 먹으면, dotfiles 레포를 받아 stow로 **패키지 설치**하면 된다. 그 땐 dotfiles 레포 루트에서 모든 패키지(`*`)를 설치하면 될 것이다:

```sh
❯ stow -S * -t ~
```


실제로 stow는 심링크를 이용해 패키지를 설치하기 위해 만들어진 것이라고 한다. 요즘엔 이렇게 dotfiles를 관리하는데 많이 이용하는것 같다.

언젠가 있을지 모르는 로컬 dotfiles를 새로운 곳에 설치할 일 그리고 사라질 일을 대비해서도 dotfiles를 git repo로 관리하는게 좋다. 하지만 형상 관리가 된다는게 더 실질적인 이득이다.

Bare repo 방식과 비교하면, stow가 홈 디렉토리와 git repo를 연결(심링크)하는 과정이 더 명시적이기 때문에 장점이 있다고 느껴진다. 특히 아예 새로 설치할 때 (새 파일을 추가할 때마다) 자주 해보던 일이라 당황하지 않을것 같다.

---

## 참고
- https://gruby.medium.com/dotfile-how-to-manage-and-sync-with-git-gnu-stow-6beada1529ea
- https://jeonwh.com/stow/
- https://linux.die.net/man/8/stow

