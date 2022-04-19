---
title: "Dockerfile Best Practices"
date: 2022-04-19T10:07:58+09:00
tags:
- docker
---

https://docs.docker.com/develop/develop-images/dockerfile_best-practices/

위 가이드는 CKS를 공부하던 중 multi-stage build라는 키워드로 유입됐다. 회사에서도 컨테이너라이즈를 하는 중이라 한번 읽고 정리하며 몇가지 실험을 해봤다(그리고 후술하지만, 이 문서를 포함해 도커 문서는 최신화가 덜 되어 있다. 다른 레퍼런스나 질문도 그리고 직접 실험하는 것도 활용해보자)

Dockerfile은 도커 이미지를 만드는 명령을 써 놓은 파일이다. Dockerfile을 잘 쓰는 것은 '어떻게 **좋은 도커 이미지**를 만들 것인가'에 대한 답이다. 좋은 도커 이미지란 가능한 **작은** 크기의 이미지를 말한다. 작은 이미지는 다음과 같은 이점이 있다:
- 빌드, 배포 속도가 빠름
- 보안(Reduce attack surface)

다른 사람들도 빌드 시간이 오래 걸려서 힘들어 하고 있는걸 어렵지 않게 찾아 볼 수 있다. 빌드 시간은 곧 배포 속도로 직결되기 때문에 실질적으로 중요한 문제이다. 특히 레일즈는 앱의 바이너리를 빌드하지 않고 코드 전체를 올려야 하기 때문에, 컨테이너라이즈의 이점을 잘 살릴 방법을 고민해야 할 것 같다.

이미지에서 실행할 앱과 무관한 모든 것은 공격 대상이 될 수 있다. CKS에선 앱과 필요 없는 패키지, 쉘을 모두 삭제하라고 말한다. 쉘이 없다면 디버깅이나 트러블 슈팅이 어려울 수도 있다. 보안과 미래에 있을 불편함 사이에서 타협점을 잘 보아야 할거 같다. 아직까진 보안 문제를 자발적으로 염두에 두고 작업한 적은 없다. 따라서 이미지 크기를 줄이다 보면 보안적인 이점은 따라 오는거라고 생각해도 될거 같다.

그럼 다시 문제는, '**어떻게** 작은 이미지를 만들 것인가'이다. 이를 위해 도커 이미지 레이어와 캐시에 대한 이해가 필요하다.

### 레이어

도커 이미지는 여러 레이어로 이루어져 있다. 레이어는 말 그대로 층층이 쌓인(stack) 구조이고, 이전(부모) 레이어의 delta가 이미지 레이어로 계속 쌓여서 전체 이미지를 만든다. Git 커밋이 diff patch인것과 비슷하게 생각하면 된다.

모든 Dockerfile instruction은 레이어를 만들지만, 실제 이미지를 만드는 것은 `RUN`, `ADD`, `COPY` 세가지이고 나머진 임시 레이어를 만들 뿐 결과 이미지 레이어엔 포함되지 않는다. 이를 확인하기 위해 모든 instruction이 들어간 도커 이미지를 하나 만들었는데 결론부터 말하면 완전히 맞는 말이 아니었다.


다음 실험은 https://github.com/flavono123/dockerstudy/tree/main/layer 에 코드와 방법이 있다.

'`RUN`, `ADD`, `COPY`만 실제 이미지 레이어를 만든다'를 증명해보려 모든 instruction을 포함한, 딱히 합리적이지 않은 이미지를 만들었다:
```Dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.15.4
LABEL name=layer
EXPOSE 3456
ENV APP=layer
ADD add.tar.gz /
COPY copy /copy
ENTRYPOINT ["date"]
RUN rm /bin/arch
CMD ["--help"]
VOLUME /log
USER root
ARG workdir
WORKDIR $workdir
ONBUILD RUN echo 'b'
STOPSIGNAL SIGTERM
HEALTHCHECK CMD which date
SHELL ["/bin/sh", "-c"]
```

참고로 Dockerfile instructions는 https://docs.docker.com/engine/reference/builder/ 문서에서 대문자인 h2를 모았다(내가 못 찾은건지.. 도커 문서는 친절하지 않은거 같다):
```js
Array.prototype.map.call(
  document.querySelectorAll('h2'), n => n.innerText)
  .filter(t => t == t.toUpperCase())
```

빌드한 이미지의 레이어를 보면 5개가 있다:
```sh
❯ docker build -t layer .
(...생략...)

❯ docker inspect layer | jq -r '.[0].RootFS.Layers[]'
5

```

[history](https://docs.docker.com/engine/reference/commandline/history/) 서브 명령을 쓰면 레이어를 볼 수 있지만, 결과 레이어가 아닌 빌드 시 사용된 임시 레이어도 포함한다(대부분 크기가 0이다):

```sh
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
d1a2d77ea138   5 days ago    SHELL [/bin/sh -c]                              0B        buildkit.dockerfile.v0
<missing>      5 days ago    HEALTHCHECK &{["CMD-SHELL" "which date"] "0s…   0B        buildkit.dockerfile.v0
<missing>      5 days ago    STOPSIGNAL SIGTERM                              0B        buildkit.dockerfile.v0
<missing>      5 days ago    WORKDIR /                                       0B        buildkit.dockerfile.v0
<missing>      5 days ago    ARG workdir                                     0B        buildkit.dockerfile.v0
<missing>      5 days ago    USER root                                       0B        buildkit.dockerfile.v0
<missing>      5 days ago    VOLUME [/log]                                   0B        buildkit.dockerfile.v0
<missing>      5 days ago    CMD ["--help"]                                  0B        buildkit.dockerfile.v0
<missing>      5 days ago    RUN /bin/sh -c rm /bin/arch # buildkit          0B        buildkit.dockerfile.v0
<missing>      5 days ago    ENTRYPOINT ["date"]                             0B        buildkit.dockerfile.v0
<missing>      5 days ago    COPY copy /copy # buildkit                      1.05MB    buildkit.dockerfile.v0
<missing>      5 days ago    ADD add.tar.gz / # buildkit                     2.1MB     buildkit.dockerfile.v0
<missing>      5 days ago    ENV APP=layer                                   0B        buildkit.dockerfile.v0
<missing>      5 days ago    EXPOSE map[3456/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      5 days ago    LABEL name=layer                                0B        buildkit.dockerfile.v0
```

여기서 [dive](https://github.com/wagoodman/dive)라는 프로그램을 알게 되었다. 도커 이미지의 레이어를 TUI 환경에서 시각적으로 표현해준다. 왼쪽 탭엔 레이어 크기와 해당하는 Dockerfile의 instruction을 확인할 수 있다. 오른쪽엔 왼쪽의 해당 레이어에서 컨테이너 내 루트 파일시스템을 트리로 접었다 펼칠 수 있고 이전 레이어와의 delta가 diff처럼 초록/빨강으로 표시된다. [녹화한 영상](https://asciinema.org/a/w8pDE6cqNxhZoTO1vsIMhxlE3)을 참고.

왼쪽 탭에서 쓰인 instruction이 `RUN`, `ADD`, `COPY` 외에도 `FROM`, `WORKDIR`이 있음을 확인할 수 있다. `FROM`이 `scratch`가 아닌 이상 크기가 있는 이미지 레이어가 있는 것은 이상하지 않다. 하지만 `WORKDIR`은 왜 결과 이미지에 레이어가 있는지 모르겠다. 크기도 0이다. 바로 위의 `RUN`은 파일을 삭제하기 때문에 해당 레이어의 크기는 0이다. 하지만 일반적인 동작으로 `RUN`을 레이어에 추가하는 것은 이해가 간다.

`WORKDIR` 이미지 레이어에 대한 또 다른 의문점은 생기기도 하고 없어지기도 한다는 점이다. 이미지를 몇 번 재생성 했는데 `WORKDIR`에 대한 레이어가 없는 경우도 있었다.

이와 관련한 레퍼런스를 찾진 못했다. 아마 도커 코드를 봐야 알 수 있을 것 같다. 아니면 storage driver 쪽을 더 보면 레이어 다이제스트로 더 많은 정보를 볼 수 있을지도 모르겠다.

### 캐시
이미지 빌드 시 캐시된 레이어를 사용한다. 레이어 캐시가 무효화 되면 자식 레이어부터 전부 새로 빌드한다. 캐시 무효화 여부는 **같은 명령**을 쓰는지로 판단한다. 즉 캐시 키로 instruction의 인자를 본다는 것이다.

다만 `ADD`와 `COPY` 그렇지 않다. 이 둘은 뒤에 오는 `src, [...,] dest` 인자 경로가 아니라, 해당 경로 파일들의 **체크섬**을 캐시 키로 쓴다. 이 때 last modified/accessed time은 제외 된다. 파일 체크섬을 캐시 키로 쓰는 것은 `ADD`, `COPY`에만 해당하기 때문에 주의해야한다. 예를 들어 `RUN apt-get -y update` 명령은 `apt-get -y update`라는 글자가 바뀌지 않으면 캐시를 계속 유지한다. 명령 인자가 컨테이너 내의 파일일 경우도 내용이 바뀌더라도 명령에 쓰이는 파일 경로가 같다면 캐시는 깨지지 않는다.

### 빌드 컨텍스트
도커 이미지 빌드 시, `ADD`, `COPY`를 컨테이너에 포함되는 파일 뿐만 아니라, 빌드 인자 하위의 모든(recursively) 파일과 디렉토리는 도커 대몬에 전송된다. 이 인자를 빌드 컨텍스트라고 한다. 따라서 빌드 컨텍스트에 불필요한 파일을 포함하지 않아야 한다.

이를 실천하기 위해 빌드 시 도커 파일과 빌드 컨텍스트를 둘 다 명시적으로 쓰는게 좋다. 보통은 빌드를 실행하는 곳에 Dockerfile을 두고 `docker build -t <image> .` 하여 현재 디렉토리를 빌드 컨텍스트로, Dockerfile은 default로 넘겨준다. 하지만 `docker build -t <image> -f <path/to/Dockerfile> <path/to/buildcontext>`와 같이 빌드 컨텍스트를 이해하고 명시적으로 쓰는게 낫다. 만약 사용하려는 빌드 컨텍스트에 제외하고 싶은 파일이 있다면 [.dockerignore ](https://docs.docker.com/engine/reference/builder/#dockerignore-file)을 활용하자.

이것도 실험해보았는데(https://github.com/flavono123/dockerstudy/tree/main/buildcontext) 현재(>=18.09) 이미지 빌더인 [Buidkit](https://docs.docker.com/develop/develop-images/build_enhancements/)이 아닌 이전의 레거시 빌더에서 발생하는 일이다.

같은 Dockerfile을 쓰는데 다른 빌드 컨텍스트를 쓰는 두개의 이미지를 만든다. `small` 이미지는 `COPY` 하는 파일이 1KB로 이미지의 크기는 작지만, 활용하지 않는 1GB짜리 더미 파일이 빌드 컨텍스트에 있다. `large` 이미지는 `COPY` 하는 파일이 10MB로 상대적으로 크지만 그 외에 빌드 컨텍스트에 파일은 없다.

먼저 레거시 빌더로 빌드하면 빌드 컨텍스트를 도커 대몬에 전송하느라 `small`의 빌드 시간이 훨씬 오래 걸린다:
```sh
❯ time sh -c "DOCKER_BUILDKIT=0 docker build -t small -f Dockerfile contexts/small"
Sending build context to Docker daemon  1.074GB
Step 1/2 : FROM alpine:3.15.4
 ---> 0ac33e5f5afa
Step 2/2 : COPY file /file
 ---> Using cache
 ---> 5b93097102e3
Successfully built 5b93097102e3
Successfully tagged small:latest

real    0m29.112s
user    0m3.000s
sys     0m5.254s

❯ time sh -c "DOCKER_BUILDKIT=0 docker build -t large -f Dockerfile contexts/large"
Sending build context to Docker daemon  10.49MB
Step 1/2 : FROM alpine:3.15.4
 ---> 0ac33e5f5afa
Step 2/2 : COPY file /file
 ---> 682f4b3630c3
Successfully built 682f4b3630c3
Successfully tagged large:latest

real    0m1.175s
user    0m0.227s
sys     0m0.221s
```

더미 파일을 삭제하거나 .dockerignore에 명시해주면 빌드는 빨라진다. .dockerignore는 빌드 컨텍스트에 써주어야 한다:
```sh
❯ echo dummy > contexts/small/.dockerignore

❯ time sh -c "DOCKER_BUILDKIT=0 docker build -t small -f Dockerfile contexts/small"
Sending build context to Docker daemon  3.136kB
Step 1/2 : FROM alpine:3.15.4
 ---> 0ac33e5f5afa
Step 2/2 : COPY file /file
 ---> Using cache
 ---> 5b93097102e3
Successfully built 5b93097102e3
Successfully tagged small:latest

real    0m0.817s
user    0m0.196s
sys     0m0.255s
```

현재 빌더인 Buildkit으로 빌드하면 더미 파일은 전송하지 않는 것으로 보인다:
```sh
❯ docker rmi small large
Untagged: small:latest
Deleted: sha256:5b93097102e3273884038716f215dadb52e4cbce1c65080138122ad28541aec7
Untagged: large:latest
Deleted: sha256:682f4b3630c3bce99a8286b3b5762e8a5fa05bacb128939043329eb3997d1b1c
Deleted: sha256:9741917140be1d96205fc6ec1fc0df7b8b8d4a56bc5892d00e30a744b3bc4cc1

❯ rm contexts/small/.dockerignore

❯ time docker build -t small -f Dockerfile contexts/small/
[+] Building 3.1s (11/11) FINISHED
 => [internal] load build definition from Dockerfile                                                                                    0.0s
 => => transferring dockerfile: 36B                                                                                                     0.0s
 => [internal] load .dockerignore                                                                                                       0.0s
 => => transferring context: 2B                                                                                                         0.0s
 => resolve image config for docker.io/docker/dockerfile:1                                                                              2.6s
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:91f386bc3ae6cd5585fbd02f811e295b4a7020c23c7691d686830bf6233e91ad         0.0s
 => [internal] load build definition from Dockerfile                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                       0.0s
 => [internal] load metadata for docker.io/library/alpine:3.15.4                                                                        0.0s
 => [internal] load build context                                                                                                       0.0s
 => => transferring context: 26B                                                                                                        0.0s
 => [1/2] FROM docker.io/library/alpine:3.15.4                                                                                          0.0s
 => CACHED [2/2] COPY file /file                                                                                                        0.0s
 => exporting to image                                                                                                                  0.0s
 => => exporting layers                                                                                                                 0.0s
 => => writing image sha256:76d6904f144613a2b663b7b0894fb9e61401ac131c94616c04ffbfc9efa1298f                                            0.0s
 => => naming to docker.io/library/small                                                                                                0.0s

real    0m3.639s
user    0m0.302s
sys     0m0.172s

❯ time docker build -t large -f Dockerfile contexts/large/
[+] Building 1.9s (11/11) FINISHED
 => [internal] load build definition from Dockerfile                                                                                    0.0s
 => => transferring dockerfile: 106B                                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                       0.0s
 => => transferring context: 2B                                                                                                         0.0s
 => resolve image config for docker.io/docker/dockerfile:1                                                                              1.0s
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:91f386bc3ae6cd5585fbd02f811e295b4a7020c23c7691d686830bf6233e91ad         0.0s
 => [internal] load .dockerignore                                                                                                       0.0s
 => [internal] load build definition from Dockerfile                                                                                    0.0s
 => [internal] load metadata for docker.io/library/alpine:3.15.4                                                                        0.0s
 => CACHED [1/2] FROM docker.io/library/alpine:3.15.4                                                                                   0.0s
 => [internal] load build context                                                                                                       0.3s
 => => transferring context: 10.49MB                                                                                                    0.3s
 => [2/2] COPY file /file                                                                                                               0.1s
 => exporting to image                                                                                                                  0.1s
 => => exporting layers                                                                                                                 0.1s
 => => writing image sha256:56c6423a9bbecaaf7d003866f84c72a80548db7c5b8e5183b0f4b8463027bc3f                                            0.0s
 => => naming to docker.io/library/large                                                                                                0.0s

real    0m2.384s
user    0m0.290s
sys     0m0.213s
```

문서는 레거시 빌더 기준으로, v18.09 이상 버전에선 관계 없는 이야기 같다. 그래도 이런 히스토리와 빌드 컨텍스트를 이해하고 빌드 시 명시적으로 써주는 것이 좋아 보인다.



## Instruction Best Practices
몇 가지 Dockerfile 명령에 대한 모범 사례 가이드를 정리했다.

### RUN
`RUN`의 인자로 오는 명령은 ANDing(`&&`)하여 하나의 `RUN`으로 쓰고 가독성을 위해선 여러 줄로 나누라고 한다. 기본적으로 하나의 `RUN` 으로 합치면 이미지 레이어가 줄어드는 이점이 있다. 또 "cache busting"을 활용하기 위해서다. Debina/Ubuntu 이미지에서 많이 쓰는 다음과 같은 패키지 설치의 예에서, 만약 캐시 업데이트(`apt-get update`)와 패키지 설치(`apt-get install`)이 각각의 `RUN`으로 나뉘어 있다면 원하는 동작을 하지 않을 수 있다. 새로운 패키지를 추가하거나 버전을 업그레이드 하기 위해 install `RUN`의 인자가 바뀌더라도 update `RUN` 레이어는 캐시를 그대로 쓰기 때문이다. 연관이 있는 명령을 묶어 cache busting 할 수 있게 해야한다:
```Dockerfile
RUN apt-update && apt-get -y install \
  apkg \
  bkg \
  ...
```

`RUN` 인자의 명령으로 파이프를 쓴다면 마지막 명령의 성공 여부만 판단하여 빌드를 진행한다. 파이프된 모든 명령의 성공 여부를 체크하려면 쉘 옵션 `pipefail` 을 설정하자:
```Dockerfile
RUN set -o pipefail && ... | ...
```

이 옵션은 모든 쉘에서 지원하진 않기 때문에 쉘을 명시해주는 것이 확실하다고 한다(`/bin/bash -c "set -o pipefail ... | ... "`).

### CMD
`CMD`는 단독으로도 명령과 인자를 인자로 쓸 수도 있고:
```Dockerfile
CMD ["executable", "param1", "param2", ...]
```
exec 형태가 아닌 쉘 형태로 쓸 수도 있다:
```Dockerfile
CMD executable param1 param2 ...
```

하지만 가장 좋은 사용법은 `ENTRYPOINT`와 함께(conjunction) 메인 명령의 인자로 쓰는 것이라고 한다:
```Dockerfile
ENTRYPOINT ["command"]
CMD ["param1", "param2"]
```

나는 Dockerfile 작성 이전에 K8s Pod 정의에서 각각에 대응하는 `command`, `args`를 먼저 보아 그런지 납득이 가는 방법이다.


### ADD or COPY
`ADD`, `COPY` 둘 다 비슷한 일을 하지만, 보통의 상황에선 `COPY`를 사용하라고 말한다. `COPY`가 하는 일, 빌드 컨텍스트의 파일을 컨테이너 파일시스템으로 복사하는 일, 이 더 명료하기 때문이다(transparent).

`ADD`는 이외에 부가적인 일을 두 가지 더 할 수 있다. 하나는 **빌드 컨텍스트의 tar 파일을 컨테이너 내로 옮기며 자동으로 추출(extract)해준다.** 이 용도로만 `ADD`를 쓸 것을 권장한다. 다른 하나는 src 경로가 빌드 컨텍스트가 아닌 원격 URL을 사용하면 다운 받을 수 있다는 건데 추천하지 않는 방법이다. 어차피 받은 파일을 쓰거나 다 쓴 후 삭제하려면 `RUN`을 사용할테니 `curl`이나 `wget` 같은 명령을 써서 받은 후 처리도 한 레이어에서 하는 것을 추천한다.


## 정리
- Dockerfile instruction 중 결과 이미지 레이어를 만들 수 있는 것은 `FROM`, `COPY`, `ADD`, `WORKDIR`이다.
- 레이어는 instruction 인자를 캐시 키로 쓴다.
  - `ADD`와 `COPY`는 src 파일의 체크섬을 캐시 키로 쓴다(last modified/accessed time 제외)
- `docker build` 명령의 인자인 빌드 컨텍스트와 빌드 시간에 끼칠 수 있는 영향을 이해한다.
  - 과거 Buildkit 이전의 빌더는 빌드 컨텍스트 아래 모든 파일, 디렉토리를 도커 대몬에 전송했다.
  - Dockerfile과 빌드 컨텍스트를 명시적으로 써주자.
    - `docker build -t <image> .` 대신
    - `docker build -t <image> -f <path/to/Dockerfile> <path/to/context>` 처럼 쓰자.
  - .dockerfile을 활용하자. 빌드 컨텍스트에 위치해야 한다.
- `RUN`은 최대한 한줄에 쓰되
  - 가독성을 위해 `\`로 개행하고
  - cache busting 할 수 있도록 관련 있는 명령을 꼭 묶어야 한다.
- `CMD`는 `ENTRYPOINT`와 함께 써서 가변 인자의 기본 값을 써주자.
- `ADD` 보단 `COPY` 를 쓰자.
  - src 파일이 tar 인 경우에만 `ADD`를 써서 컨테이너에 추출한다.
- 도커 문서는 최신화가 덜 되어 있어 다른 레퍼런스를 많이 참고하자.


---

## 참고
- https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- https://docs.docker.com/engine/reference/builder/
- https://github.com/flavono123/dockerstudy
- https://stackoverflow.com/questions/71853912/docker-workdir-create-a-layer
- https://stackoverflow.com/questions/71838211/docker-build-context-when-it-transferred-and-does-it-cached-by-the-runtime/71885289?noredirect=1#comment127084326_71885289
