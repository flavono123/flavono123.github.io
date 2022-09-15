---
title: "Ansible에서 unprivileged become_user 실패"
date: 2022-09-15T10:28:13+09:00
tags:
- ansible
- linux
- maxscale
---

Ansible 태스크에서 `become: yes`를 사용해 `sudo`처럼 루트 권한으로 명령을 실행한다. 예를 들면 Ubuntu에서 apt 패키지를 쓸 때 사용할 수 있다. 하지만 루트가 아닌 특정 사용자로 실행해야 하는 명령도 있다. 최근에 [MaxScale](https://mariadb.com/kb/en/maxscale/)을 2.5 버전으로 업그레이드 했는데, 구성 확인 명령이 이전과 달리 `maxscale` 사용자만 실행할 수 있도록 강제하고 있다:

```sh
$ whoami
ubuntu
$ maxscale --config-check
info   : MaxScale will be run in the terminal process.
2022-09-15 10:34:01   notice : Worker message queue size: 1MiB
2022-09-15 10:34:01   warning: Number of threads set to 8, which is greater than the number of processors available: 1
2022-09-15 10:34:01   warning: Number of threads set to 8, which is greater than the number of processors available: 1
2022-09-15 10:34:01   notice : Using up to 296.41MiB of memory for query classifier cache
2022-09-15 10:34:01   notice : syslog logging is enabled.
2022-09-15 10:34:01   notice : maxlog logging is enabled.
alert  : MaxScale doesn't have write permission to '/var/cache/maxscale'.: Permission denied.
$ ll /var/cache/maxscale/
total 12
drwxr-xr-x  2 maxscale maxscale 4096 Sep 14 18:14 ./
drwxr-xr-x 14 root     root     4096 Sep 14 17:45 ../
-rwxr-xr-x  1 maxscale maxscale    4 Sep 14 18:14 maxscale.lock*
$ sudo maxscale --config-check
alert  : MaxScale cannot be run as root.
$ sudo chown -R ubuntu:ubuntu /var/cache/maxscale/
$ ll /var/cache/maxscale/
total 12
drwxr-xr-x  2 ubuntu ubuntu 4096 Sep 14 18:14 ./
drwxr-xr-x 14 root   root   4096 Sep 14 17:45 ../
-rwxr-xr-x  1 ubuntu ubuntu    4 Sep 14 18:14 maxscale.lock*
$ maxscale --config-check
info   : MaxScale will be run in the terminal process.
2022-09-15 10:57:36   notice : Worker message queue size: 1MiB
2022-09-15 10:57:36   warning: Number of threads set to 8, which is greater than the number of processors available: 1
2022-09-15 10:57:36   warning: Number of threads set to 8, which is greater than the number of processors available: 1
2022-09-15 10:57:36   notice : Using up to 296.41MiB of memory for query classifier cache
2022-09-15 10:57:36   notice : syslog logging is enabled.
2022-09-15 10:57:36   notice : maxlog logging is enabled.
alert  : MaxScale doesn't have write permission to '/var/run/maxscale'.: Permission denied.
```

로그인한 사용자인 `ubuntu`로 실행했을 때 /var/cache/maxscale 쓰기 권한 문제로 막힌다. sudo를 써서 루트로 명령하는 것은 아예 막혀 있다. /var/cache/maxscale의 소유자와 그룹을 `ubuntu`로 바꿔도 실패한다.

```sh
$ sudo chown -R maxscale:maxscale /var/cache/maxscale/
ubuntu@ldev1:~$ maxscale --config-check
info   : MaxScale will be run in the terminal process.
2022-09-15 10:57:53   notice : Worker message queue size: 1MiB
2022-09-15 10:57:53   warning: Number of threads set to 8, which is greater than the number of processors available: 1
2022-09-15 10:58:00   warning: Number of threads set to 8, which is greater than the number of processors available: 1
2022-09-15 10:58:00   notice : Using up to 296.41MiB of memory for query classifier cache
2022-09-15 10:58:00   notice : syslog logging is enabled.
2022-09-15 10:58:00   notice : maxlog logging is enabled.
2022-09-15 10:58:00   notice : Running OS: Linux@5.15.0-47-generic, #51-Ubuntu SMP Thu Aug 11 07:51:15 UTC 2022, x86_64 with 1
processor cores.
2022-09-15 10:58:00   notice : Total usable main memory: 1.93GiB.
2022-09-15 10:58:00   notice : MariaDB MaxScale 2.5.21 started (Commit: eb659891d7b507958f3c5f100d1ebe5f0f68afaf)
2022-09-15 10:58:00   notice : MaxScale is running in process 4713

Configuration file : /etc/maxscale.cnf
Log directory      : /var/log/maxscale
Data directory     : /var/lib/maxscale
Module directory   : /usr/lib/x86_64-linux-gnu/maxscale
Service cache      : /var/cache/maxscale

2022-09-15 10:58:00   notice : Configuration file: /etc/maxscale.cnf
2022-09-15 10:58:00   notice : Log directory: /var/log/maxscale
2022-09-15 10:58:00   notice : Data directory: /var/lib/maxscale
2022-09-15 10:58:00   notice : Module directory: /usr/lib/x86_64-linux-gnu/maxscale
2022-09-15 10:58:00   notice : Service cache: /var/cache/maxscale
2022-09-15 10:58:00   notice : Loaded module qc_sqlite: V1.0.0 from /usr/lib/x86_64-linux-gnu/maxscale/libqc_sqlite.so
2022-09-15 10:58:00   notice : Query classification results are cached and reused. Memory used per thread: 37.05MiB
2022-09-15 10:58:00   warning: File format of '/var/lib/maxscale/.secrets' is deprecated. Please generate a new encryption key ('maxkeys') and re-encrypt passwords ('maxpasswd').
2022-09-15 10:58:00   notice : Using encrypted passwords. Encryption key read from '/var/lib/maxscale/.secrets'.
2022-09-15 10:58:00   notice : MaxScale started with 8 worker threads, each with a stack size of 8388608 bytes.
2022-09-15 10:58:00   notice : Loading /etc/maxscale.cnf.
2022-09-15 10:58:00   notice : Loaded module MariaDBClient: V1.1.0 from /usr/lib/x86_64-linux-gnu/maxscale/libmariadbclient.so
2022-09-15 10:58:00   notice : Loaded module readconnroute: V2.0.0 from /usr/lib/x86_64-linux-gnu/maxscale/libreadconnroute.so
2022-09-15 10:58:00   notice : Loaded module mariadbmon: V1.5.0 from /usr/lib/x86_64-linux-gnu/maxscale/libmariadbmon.so
2022-09-15 10:58:00   notice : (Read_Listener) Loaded module MariaDBAuth: V2.1.0 from /usr/lib/x86_64-linux-gnu/maxscale/libmariadbauth.so
2022-09-15 10:58:00   notice : Configuration was successfully verified.
2022-09-15 10:58:00   notice : MaxScale is shutting down.
2022-09-15 10:58:00   notice : Stopped MaxScale REST API
2022-09-15 10:58:00   notice : All workers have shut down.
2022-09-15 10:58:00   notice : MaxScale shutdown completed
```

다시 /var/cache/maxscale/의 소유자와 그룹을 `maxscale`로 바꾸고 sudo를 사용해 `maxscale` 사용자로 실행하니 성공한다.

이를 Ansible에서 하기 위해 태스크에 become과 become_user를 추가했다:
```yaml
- name: Test Maxscale Configuration
  become: yes
  become_user: maxscale
  command: maxscale --config-check
```


하지만 play하면 다음 에러가 발생한다:
```sh
Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user (rc: 1, err: chmod: invalid mode: ‘A+user:maxscale:rx:allow’\nTry 'chmod --help' for more information.\n}). For information on working around this, see https://docs.ansible.com/ansible-core/2.12/user_guide/become.html#risks-of-becoming-an-unprivileged-user
```

에러는 Ansible이 unprivileged user(maxscale)로 실행하기 위해 임시 파일을 만드는데 권한 실패 문제, 즉 chmod를 잘못 써서 발생했다(`A+user:maxscale:rx:allow`는 딱 봐도 chmod 모드 포맷이 아닌데 왜 저리 썼는지 모르겠다). [참고 링크](https://docs.ansible.com/ansible-core/2.12/user_guide/become.html#risks-of-becoming-an-unprivileged-user)를 들어가 읽어 보았다:

지금과 같은 상황, 즉 become 사용 시 become_user가 루트 또는 루트 권한이 없는 unprivileged 사용자일 경우, Ansible은 POSIX.1e ACL을 지원하는 파일시스템에서 권한을 unprivileged 사용자에게 공유하여 이를 해결한다. [setfacl](https://www.gsp.com/cgi-bin/man.cgi?section=1&topic=setfacl) POSIX.1e ACL 도구 중 하나이며 Ubuntu에선 패키지로 설치할 수 있다:
```sh
$ sudo apt install acl
```

설치하여 실행 파일 setfacl이 있다면 위 태스크는 성공한다. POSIX ACL에 대한 더 자세한 내용은 파일 권한과 비교한 [이 글](https://www.bangseongbeom.com/linux-acl-guide.html#%ED%8C%8C%EC%9D%BC-%ED%8D%BC%EB%AF%B8%EC%85%98-vs-acl)을 참고하자


## 정리
- Ansible become은 unprivileged 사용자로 실행 시 실패한다.
- Ubuntu에선 setfacl를 설치하여 Ansible이 unprivileged 사용자에게 파일 권한을 주어 이를 해결한다.


---

## 참고
- https://docs.ansible.com/ansible/latest/user_guide/become.html#risks-of-becoming-an-unprivileged-user
- https://www.gsp.com/cgi-bin/man.cgi?section=3&topic=posix1e
- https://www.bangseongbeom.com/linux-acl-guide.html#%ED%8C%8C%EC%9D%BC-%ED%8D%BC%EB%AF%B8%EC%85%98-vs-acl
