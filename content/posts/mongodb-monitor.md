---
title: "MongoDB 모니터가 빠진 건에 대하여"
date: 2022-02-23T22:50:45+09:00
tags:
- mongodb
- ansible
---

어느 날, MongoDB 서버에서 CPU 사용율이 높다는 알람이 왔다. CPU 사용율은 금방 정상을 돌아왔지만, pmm-client가 설치되지 않아서 쿼리를 특정할 수 없었다. pmm-client는 MongoDB 각 멤버가 재시작 되면서(한때 장애로 failover를 밥 먹듯이 했다...) server로부터 떨어진거 같았다. 최근엔 재시작 되는 일 없이 안정적으로 서비스 실행 중이라 다시 붙이기로 했다. 그리고 그 사이에 클러스터에 변화도 생겼는데 이것도 반영했다.

## 클러스터 구성

현재 클러스터 구성은 다음과 같다:

|host1|host2|host3|host4|host5|host6|
|:----|:----|:----|:----|:----|:----|
|replicaset1|replicaset1|replicaset1|replicaset2|replicaset2|replicaset2|
|config|config|config||||
|**router**|**router**|router|**router**|**router**|router|

총 6대의 호스트에 설치되어 있다. replicaset1, 2는 각각 shard에 해당한다. 위에서 말한 클러스터의 변화가, 내가 pmm-client 설치한 이후로, shard2가 확장됐다. 그리고 shard1의 멤버와 같은 호스트에 config replicaset도 같이 설치했다. router는 shard1,2 멤버 모두에 같이 설치되어 있는데 실제 서비스에서 요청을 받는 호스트는 각 shard 당 두개씩(host1,2,4,5)이고 나머진 standby이다.

MongoDB는 클라이언트로부터 요청을 받고 분배하는 라우터도 별도의 프로세스(mongos)이고, 설정 서버도 그렇다. 설정 서버는 shard와 같이 replicaset을 구성해야 한다([3.4 버전 이상부터 그렇다](https://docs.mongodb.com/v3.4/tutorial/upgrade-config-servers-to-replica-set/). 회사에선 4.4.6 버전을 사용 중이다).

## pmm-client 등록
MongoDB를 포함한 DB 서버엔 pmm-client가 설치되어 있다(다른 DB는 MariaDB 사용 중). 맨 처음엔 [PMM for MongoDB: Quick Start Guide](https://www.percona.com/blog/2019/07/23/pmm-for-mongodb-start-guide/)이라는 글을 참고해 pmm-server에 mongodb를 등록했다. 딱히 클러스터 구성을 신경쓰지 않은, standalone용 설치법이다. 이번엔 [Configuring PMM Monitoring for MongoDB Cluster](https://www.percona.com/blog/2018/07/05/configuring-pmm-monitoring-mongodb-cluster/) 글을 이해하고 작업했다.

먼저 shard 멤버 `mongodb:metrics` 등록 시 `--cluster` 옵션을 추가했다. pmm이 제공하는 grafana 대시보드에서 이게 없으면 몇가지 안 나오는 데이터 포인트가 있었다:
```sh
$ sudo pmm-admin add mongodb --uri mongodb://<pmm_user>:<pmm_password>@<mongodb_host>:<mongodb_shard_port>
```

shard1부터 등록했는데, shard2는 uri에 접속이 거부되며 실패했다.

## shard2만 없는 mongodb_exporter
위 명령에서 `<pmm_user>`로 표시한, MongoDB 사용자 이름은 `mongodb_exporter`이다. 관련 있는 `explainRole`이라는 권한도 [이 문서](https://www.percona.com/doc/percona-monitoring-and-management/1.x/conf-mongodb.html)에 설명되어 있다. 이 부분만 노출한 ansible 태스크 파일을 보면 다음과 같다:
```yaml
---

- name: Add crema mongodb user
  mongodb_user:
    # opaque
    # ...

- name: Add pmm role of shard server
  command: >-
    /usr/bin/mongo {{ ansible_hostname }}:{{ mongodb_port }} -u {{ mongodb_admin_user }} -p {{ mongodb_admin_password }} --authenticationDatabase admin --eval
    'db.
      getSiblingDB("admin").
      createRole({
        role: "explainRole",
        privileges: [{
          resource: {
            db: "",
            collection: ""
          },
          actions: [
            "listIndexes",
            "listCollections",
            "dbStats",
            "dbHash",
            "collStats",
            "find"
          ]
        }],
      roles:[]
    })'
  ignore_errors: true
  when:
    - ansible_hostname == mongodb_replica_members[0]

- name: Configure pmm user of shard server
  mongodb_user:
    database: admin
    user: "{{ pmm_user }}"
    password: "{{ pmm_password }}"
    login_user: "{{ mongodb_admin_user }}"
    login_password: "{{ mongodb_admin_password }}"
    login_port: "{{ mongodb_port }}"
    state: present
    update_password: on_create
    roles:
      - db: admin
        role: explainRole
      - db: admin
        role: clusterMonitor
      - db: local
        role: read
  when:
  - ansible_hostname == mongodb_replica_members[0]
```
`mongodb_replica_members`란 변수는 shard 확장 전이라 `[host1, host2, host3]`이었다. 그런데 팀장이 shard 확장 작업을 하며 태스크 파일을 수정하셨는데, `mongodb_init_user`라는 변수로 `[host4, host5, host6]`에 대해선 `false`로 해놓았다:

```yaml
- name: Add crema mongodb user
  mongodb_user:
    # opaque
    # ...
- name: Configure pmm user of shard server
  ...
  when:
    - ansible_hostname == mongodb_replica_members[0]
    - mongodb_init_user == True
- name: Add pmm role of shard server
  ...
  when:
    - ansible_hostname == mongodb_replica_members[0]
    - mongodb_init_user == True
```

그래서 shard2에만 `mongodb_exporter`가 없었다. `mongodb_init_user` 처음 한번 실행할 때만? 필요하다고 붙인 조건이라 들었다. 그래서 이 부분을 별도 파일로 분리하고 `mongodb_init_user` 조건을 제거하여 실행했다. shard2에도 `mongodb_exporter` 계정을 생성했다.

그러고나니 config 서버도 모니터링 하면 좋겠다는 생각이 들었다. 분리한 파일에 포트를 변수(`mongodb_port_for_pmm`)로 주어 같은 호스트에 shard 서버 뿐만 아니라 config 서버도 `mongodb_exporter`를 생성했다(호스트가 다르다면 hostname이나, shard 서버와 config 서버를 구분하는 것을 변수로 만들고 인자로 주면 될 것이다):

```yaml
---
- name: Check pmm role exist
  command: >-
    /usr/bin/mongo {{ ansible_hostname }}:{{ mongodb_port_for_pmm }} -u {{ mongodb_admin_user }} -p {{ mongodb_admin_password }} --authenticationDatabase admin -quiet --eval
    'db.getSiblingDB("admin").getRole("explainRole")'
  register: result_pmm_role

- name: Add pmm role of replicaset members
  command: >-
    /usr/bin/mongo {{ ansible_hostname }}:{{ mongodb_port_for_pmm }} -u {{ mongodb_admin_user }} -p {{ mongodb_admin_password }} --authenticationDatabase admin --eval
    'db.
      getSiblingDB("admin").
      createRole({
        role: "explainRole",
        privileges: [{
          resource: {
            db: "",
            collection: ""
          },
          actions: [
            "listIndexes",
            "listCollections",
            "dbStats",
            "dbHash",
            "collStats",
            "find"
          ]
        }],
      roles:[]
    })'
  when:
    - result_pmm_role is not skipped
    - result_pmm_role.stdout == "null"

- name: Configure pmm user of replicaset members
  mongodb_user:
    database: admin
    user: "{{ pmm_user }}"
    password: "{{ pmm_password }}"
    login_user: "{{ mongodb_admin_user }}"
    login_password: "{{ mongodb_admin_password }}"
    login_port: "{{ mongodb_port_for_pmm }}"
    state: present
    update_password: on_create
    roles:
      - db: admin
        role: explainRole
      - db: admin
        role: clusterMonitor
      - db: local
        role: read

---

- name: Setting up users for shard replicaset members
  import_tasks: configure-pmm-user-role.yml
  vars:
  - mongodb_port_for_pmm: '{{ mongodb_port }}'
  when:
    - ansible_hostname == mongodb_replica_members[0]

- name: Setting up users for config replicaset members
  import_tasks: configure-pmm-user-role.yml
  vars:
  - mongodb_port_for_pmm: '{{ mongodb_config_port }}'
  when:
    - ansible_hostname == mongodb_config_replica_members[0]
```

`when` 조건을 보면 유추할 수 있지만, replicaset에서 primary에만 사용자 생성을 하기 위해 변수의 첫번째의 priority를 높여 놓고 실행한다.

router 서버의 계정은, replicaset과 다르게, 어디든 한번만 생성하면 모든 router 서버에서 공유한다. 그럼 이것도 `mongodb_init_user` 조건 필요 없이 `run_once` 옵션으로 가능하다. `mongodb_init_user`는 왜 필요할까?

## Stateful에게 idempotent란
ansible 코드는 idempotent 하게 만들어야 한다. 하지만 프로비저닝이 여러번 실행할 필요는 없는데, 그에 비해 호스트 상태를 확인하고 그에 따라 여러 조건을 확인해야한다. 그러한 이유로 ansible 코드를 idempotent하게 만드는 일을 '적당히' 타협하게 된다.

위 예제에서도 ansible의 호스트 변수인 `mongodb_replica_members[0]`가 replicaset의 primary라는 보장은 없다. replicaset에서 상태를 가져와 비교하고(primary가 바뀌지 않도록 lock을 걸고?), 권한과 사용자를 생성해야 할 것이다.

그런 의미에서 `mongodb_init_user`는 필요 없다. ansible을 쓴 프로비저닝을 '처음에 한번만 실행하는 작업이야'라는 관점으로 작업하면 imperative가 아닌가! 그럼에도 코드의 복잡도와 적당히 타협하는건 여전히 의미 있다. 특히 ansible과 같은, 설정은 평평한 것이 좋다. 그래도 ansible에 `when`과 `ignore_*`들이 있는 한 cron `* * * * *`으로 돌려도 문제 없을 코드를 짜보고 싶다(??).

오늘은 무슨 말을 하고 싶었던건지 잘 모르겠다. 그냥 내가 잘 하고 있는건가 싶었나보다..

---


아무튼, MongoDB 클러스터는 복잡하지만 failover 하는 모습 보면 아주 맘에 든다.

- 음, MariaDB에서 고생해서 그럴지도 모른다.
- 음음, 요즘 DB는 다 이런 분산 합의 코디네이터 포함하고 잘 동작하나 모르겠다(카프카의 주키퍼였던 것이나 etcd의 Raft도 잘 동작하겠지? 장애 나 본적이 없어서, vm에서 테스트해 봐야겠다).
- 음음음, MariaDB 엔터프라이즈는 멋있게 failover 할 수 있겠지? ...
