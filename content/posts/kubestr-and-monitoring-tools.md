---
title: "Kubestr"
date: 2022-05-30T23:24:25+09:00
tags:
- kubernetes
- linux
---

[kubestr](https://kubestr.io/)라는 쿠버네티스 저장소 IO 벤치마크 도구로 저장소 성능을 측정해본다. [지난 글](/posts/statefulset-vs-deployment/)에서 만든 local-path-provisioner와 [nfs-subdir-external-provisioner](https://artifacthub.io/packages/helm/nfs-subdir-external-provisioner/nfs-subdir-external-provisioner)를 새로 설치해 둘을 비교한다. 로컬 VM에서 하는 것이기 때문에 실제 성능 측정보단 벤치마크 자체를 연습해본다. 또 벤치마킹하는 동안 컴퓨팅 자원 지표를 측정할 수 있는 모니터 도구 sar를 사용해본다.

## NFS
nfs-subdir-external-provisioner를 설치하기 위해 노드에 NFS를 마운트한다. 별도 NFS 노드를 준비하지 않고 controlplane에 마운트한다:

```yaml
---
# variables
nfs_mount_path: /nfs-storage
nfs_network_cidr: 192.168.1.0/24
---
# tasks
- name: Turn up NFS in controlplane
  become: yes
  block:
  - name: Install nfs-kernel-server
    apt:
      state: present
      update_cache: yes
      name:
      - nfs-kernel-server

  - name: Create NFS mount path
    file:
      state: directory
      owner: nobody
      group: nogroup
      mode: "0777"
      path: "{{ nfs_mount_path }}"

  - name: Add exports table to /etc/exports
    blockinfile:
      path: /etc/exports
      block: |
        {{ nfs_mount_path }}  {{ nfs_network_cidr }}(rw,sync,no_subtree_check)
  - name: Export /etc/exports
    command: exportfs -a

  - name: Restart nfs-kernel-server service
    command: systemctl restart nfs-kernel-server

  when:
  - "'controlplane' in group_names"

- name: Install nfs-client
  become: yes
  apt:
    state: present
    update_cache: yes
    name:
    - nfs-common
```

controlplane엔:
- nfs-kernel-server를 설치한다.
- 마운트 경로를 만든다.
  - 누구나(nobody:nogroup) 접근할 수 있는 권한(777)이어야 한다
- 마운트 경로와 접근 가능한 네트워크를 노출(exports)한다.
  - sync: (읽기, 쓰기) [요청에 대해 커밋한 저장소에 대해 실행](https://linux.die.net/man/5/exports)
  - no_subtree_check: 예시처럼 디스크 전체가 아닌 서브디렉토리를 마운트하는 경우 [체크를 해제(=no_subtree_check)해야 문제가 거의 없다고 한다](http://nfs.sourceforge.net/#section_c)

모든 노드에 NFS 클라이언트 사용을 위해 nfs-common을 설치한다(controlplane은 nfs-kernel-server 설치 시 의존성으로 이미 설치됐다).

NFS 자체를 테스트하는 대신 nfs 저장소 프로비저너를 통해 검증한다.


## nfs-subdir-external-provisioner

nfs-subdir-external-provisioner는 helm에 공개되어 있다:
```sh
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```

설치 시 아래 values.yaml 파일을 넘겨 프로비저너(디플로이먼트의 파드)의 명세를 바꾼다:
```yaml
nfs:
  path: /nfs-storage
  server: 192.168.1.2
nodeSelector:
  kubernetes.io/hostname: cluster1-master1
tolerations:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  operator: Exists
```

- NFS 루트 경로와 마운트한 노드(controlplane)의 IP 주소 전달
- 파드가 controlplane에서 실행될 수 있도록 NodeSelect와 Toleration 추가

values 적용이 됐는지 잘 확인하고 설치한다. 릴리즈와 같은 이름의 네임스페이스를 만들어 그곳에 설치했다:
```sh
$ k create ns nfs-provisioner
$ helm install nfs-provisioner -n nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --values values.yaml --dry-run | yq ea 'split_doc | [.][] | select(.kind=="Deployment")'
# Source: nfs-subdir-external-provisioner/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-provisioner-nfs-subdir-external-provisioner
  labels:
    chart: nfs-subdir-external-provisioner-4.0.16
    heritage: Helm
    app: nfs-subdir-external-provisioner
    release: nfs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-subdir-external-provisioner
      release: nfs-provisioner
  template:
    metadata:
      annotations:
      labels:
        app: nfs-subdir-external-provisioner
        release: nfs-provisioner
    spec:
      serviceAccountName: nfs-provisioner-nfs-subdir-external-provisioner
      securityContext: {}
      nodeSelector:
        kubernetes.io/hostname: cluster1-master1
      containers:
        - name: nfs-subdir-external-provisioner
          image: "k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2"
          imagePullPolicy: IfNotPresent
          securityContext: {}
          volumeMounts:
            - name: nfs-subdir-external-provisioner-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: cluster.local/nfs-provisioner-nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.1.2
            - name: NFS_PATH
              value: /nfs-storage
      volumes:
        - name: nfs-subdir-external-provisioner-root
          nfs:
            server: 192.168.1.2
            path: /nfs-storage
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
          operator: Exists

$ helm install nfs-provisioner -n nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --values values.yaml
NAME: nfs-provisioner
LAST DEPLOYED: Tue May 31 06:51:23 2022
NAMESPACE: nfs-provisioner
STATUS: deployed
REVISION: 1
TEST SUITE: None

```

저장소 클래스 nfs-client 생성과 프로비저너 실행 중을 확인한다:
```sh
$ k get sc nfs-client
NAME         PROVISIONER                                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-provisioner-nfs-subdir-external-provisioner   Delete          Immediate           true                   106s
$ k get all -n nfs-provisioner
NAME                                                                  READY   STATUS    RESTARTS   AGE
pod/nfs-provisioner-nfs-subdir-external-provisioner-6b56f65d85qdvp2   1/1     Running   0          107s

NAME                                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-provisioner-nfs-subdir-external-provisioner   1/1     1            1           108s

NAME                                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-provisioner-nfs-subdir-external-provisioner-6b56f65d85   1         1         1       107s
```

간단한 프로비저너 동작 검증을 위해 다음 PVC와 파드를 생성한다:
```yaml
# NFS PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
# Pod
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: test
  name: test
spec:
  containers:
  - args:
    - sh
    - -c
    - echo Test2 >> /logs/log
    image: busybox
    name: test
    resources: {}
    volumeMounts:
    - name: nfs-pvc
      mountPath: /logs
  volumes:
  - name: nfs-pvc
    persistentVolumeClaim:
      claimName: nfs-pvc
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

파드 실행 후 NFS 마운트 지점에 로그가 쓰인 것을 확인한다. 파드는 워커 노드에서 실행됐지만 controlplane에 로그가 쓰였다:
```sh
$ k get po -owide
NAME   READY   STATUS      RESTARTS   AGE   IP             NODE               NOMINATED NODE   READINESS GATES
test   0/1     Completed   0          15s   172.16.40.76   cluster1-worker1   <none>           <none>

$ cat /nfs-storage/default-nfs-pvc-pvc-b7c2e64c-4296-4f6f-816f-84a83b599490/log
Test
```


## Kubestr

kubestr의 기능 중 [fio](https://github.com/axboe/fio)라는 I/O 벤치마크 도구를 이용해 저장소 클래스의 I/O 성능을 측정한다.

먼저 kubestr를 설치한다:
```sh
$ wget -qO- https://github.com/kastenhq/kubestr/releases/download/v0.4.31/kubestr_0.4.31_Linux_amd64.tar.gz | tar xvfz -
$ mv kubestr /usr/local/bin/
$ chmod +x /usr/local/bin/kubestr
```

서브 명령 fio에 대한 도움말을 보면:
```sh
$ kubestr fio -h
Run an fio test

Usage:
  kubestr fio [flags]

Flags:
  -f, --fiofile string        The path to a an fio config file.
  -h, --help                  help for fio
  -i, --image string          The container image used to create a pod.
  -n, --namespace string      The namespace used to run FIO. (default "default")
  -z, --size string           The size of the volume used to run FIO. Note that the FIO job definition is not scaled accordingly. (default "100Gi")
  -s, --storageclass string   The name of a Storageclass. (Required)
  -t, --testname string       The Name of a predefined kubestr fio test. Options(default-fio)

Global Flags:
  -e, --outfile string   The file where test results will be written
  -o, --output string    Options(json)
```

-s 옵션으로 저장소 클래스를 줄 수 있다. 기본 테스트하는 PV 크기가 100Gi로 지금 연습하는 상황에선 적합하지 않으니 줄여준다. 어떤 테스트를 한 것인지를 결과를 보고 이해하자:
```sh
$ kubestr fio -s local-path --size 4G
PVC created kubestr-fio-pvc-wjwt2
Pod created kubestr-fio-pod-nvvmj
Running FIO test (default-fio) on StorageClass (local-path) with a PVC of Size (4G)
Elapsed time- 30.590197824s
FIO test results:

FIO version - fio-3.20
Global options - ioengine=libaio verify=0 direct=1 gtod_reduce=1

JobName: read_iops
  blocksize=4K filesize=2G iodepth=64 rw=randread
read:
  IOPS=1068.180298 BW(KiB/s)=4289
  iops: min=758 max=1422 avg=1074.133301
  bw(KiB/s): min=3033 max=5688 avg=4297.033203

JobName: write_iops
  blocksize=4K filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=647.105408 BW(KiB/s)=2605
  iops: min=304 max=808 avg=653.133362
  bw(KiB/s): min=1216 max=3232 avg=2612.933350

JobName: read_bw
  blocksize=128K filesize=2G iodepth=64 rw=randread
read:
  IOPS=1117.911987 BW(KiB/s)=143626
  iops: min=764 max=1500 avg=1124.699951
  bw(KiB/s): min=97792 max=192000 avg=143969.593750

JobName: write_bw
  blocksize=128k filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=445.627716 BW(KiB/s)=57573
  iops: min=336 max=544 avg=448.766663
  bw(KiB/s): min=43008 max=69748 avg=57449.035156

Disk stats (read/write):
  sda: ios=38257/18409 merge=336/382 ticks=2136400/2041390 in_queue=4064392, util=99.882767%                                     -  OK
```

총 네개의 테스트 job을 볼 수 있다. 각각 읽기 IOPS, 쓰기 IOPS, 읽기 Bandwidth, 쓰기 Bandwidth를 측정한 테스트 job이다.
IOPS와 Bandwidth 테스트는 블록 사이즈가 다르다(4K/128k). VM 위에서 하는 모의테스트이기도 하지만, 성능이 좋다 나쁘다를 판단하긴 어렵다..

local-path 보단 성능이 떨어질 것으로 예상되는 nfs-client에 대해서도 실행해본다:
```sh
kubestr fio -s nfs-client --size 4G
PVC created kubestr-fio-pvc-jsvtv
Pod created kubestr-fio-pod-vt2bw
Running FIO test (default-fio) on StorageClass (nfs-client) with a PVC of Size (4G)
Elapsed time- 1m42.829777425s
FIO test results:

FIO version - fio-3.20
Global options - ioengine=libaio verify=0 direct=1 gtod_reduce=1

JobName: read_iops
  blocksize=4K filesize=2G iodepth=64 rw=randread
read:
  IOPS=155.228043 BW(KiB/s)=636
  iops: min=109 max=190 avg=157.903229
  bw(KiB/s): min=439 max=760 avg=632.419373

JobName: write_iops
  blocksize=4K filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=153.714279 BW(KiB/s)=630
  iops: min=45 max=196 avg=155.774200
  bw(KiB/s): min=183 max=784 avg=623.903198

JobName: read_bw
  blocksize=128K filesize=2G iodepth=64 rw=randread
read:
  IOPS=153.777771 BW(KiB/s)=20195
  iops: min=49 max=192 avg=155.806458
  bw(KiB/s): min=6387 max=24576 avg=19979.226562

JobName: write_bw
  blocksize=128k filesize=2G iodepth=64 rw=randwrite
write:
  IOPS=154.718903 BW(KiB/s)=20315
  iops: min=86 max=193 avg=157.000000
  bw(KiB/s): min=11008 max=24782 avg=20132.097656

Disk stats (read/write):
  -  OK
```

nfs-client는 IOPS나 BW 읽기 쓰기 모두 local-path 대비 현저하게 느리다.

## sar

병목 부분 확인을 위해 [sar](https://linux.die.net/man/1/sar)로 장치 메트릭을 수집한다. 우분투에서 sar는 sysstat 패키지를 받으면 설치된다:
```sh
$ sar --dev=dev8-0 -d 5
```

위 명령을 대상이 되는 노드에서 실행한다. 즉 local-path 테스트라면 파드가 실행 중인 워커 노드(kubestr fio 테스트 시 파드 위치를 모니터 하자; `k get po -owide -w`), nfs-client라면 controlplane 노드에서 실행한다. dev는 장치 이름인데 lsblk하면 볼 수 있는 장치의 major, minor 숫자로 sar에서 장치 이름을 이처럼 매칭해 놓은 것 같다. -d는 메트릭 출력 간격을 나타낸다:
```sh
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0    7:0    0 67.8M  1 loop /snap/lxd/22753
loop1    7:1    0 61.9M  1 loop /snap/core20/1434
loop2    7:2    0 44.7M  1 loop /snap/snapd/15534
loop3    7:3    0 55.5M  1 loop /snap/core18/2409
loop4    7:4    0  4.7M  1 loop /snap/yq/1702
loop5    7:5    0 44.7M  1 loop /snap/snapd/15904
loop6    7:6    0 61.9M  1 loop /snap/core20/1494
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0   10M  0 disk
$ sar -d 1
Linux 5.4.0-113-generic (cluster1-master1)      05/31/22        _x86_64_        (2 CPU)

08:31:19          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
08:31:20       dev7-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev7-1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev7-2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev7-3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev7-4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev7-5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev7-6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:31:20       dev8-0     18.00      0.00     92.00      0.00      5.11      0.00      0.17      2.40
08:31:20      dev8-16      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

local-path와 nfs-client의 피크 시 메트릭을 비교하면 await가 local-path가 더 높다. 이는 NFS 특성 상 네트워크 요청 병목 때문에 IOPS나 BW 등 성능이 더 낮게 나온 것으로 생각된다:
```sh
# cluster1-worker1; local-path
$ sar --dev=dev8-0 -d 5
...
07:57:53          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
07:57:58       dev8-0   2210.00  97584.80  38856.80      0.00     61.74    240.78    110.94     99.28

07:57:58          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
07:58:03       dev8-0   1772.60  76728.00  32655.20      0.00     61.71    247.30    141.49    100.00

07:58:03          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
07:58:08       dev8-0   2312.20 104272.80  45652.80      0.00     64.84    234.24    103.30     98.24
...

# cluster1-master1; nfs-client
$ sar --dev=dev8-0 -d 5
...
08:20:46          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
08:20:51       dev8-0    510.00   5934.40  57926.40      0.00    125.22     11.15     23.63     85.20

08:20:51          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
08:20:56       dev8-0    662.20   5815.20  61637.60      0.00    101.86      7.38     12.74     90.80

08:20:56          DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
08:21:01       dev8-0    610.78   5457.09  46407.19      0.00     84.92      7.08     13.16     95.17
...
```

###### 1 node(s) had taint node.kubernetes.io/disk-pressure:NoSchedule

kubestr fio로, 특히 nfs-client에 대해 테스트를 여러번 하다 보면, 디스크가 꽉 차서 파드 스케쥴이 안되는 현상이 발생한다.

이 때 해당 노드(여기선 control-plane)은 node.kubernetes.io/disk-pressure:NoSchedule라는 테인트가 생긴 상태이므로, 디스크를 비워주고 테인트를 해제하자:
```sh
# cluster1-master1
$ rm -rf /nfs-storage/*
$ df -h /nfs-storage/
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        39G   13G   26G  34% /
$ k taint node cluster1-master1 node.kubernetes.io/disk-pressure:NoSchedule-

```

## 정리
이번에도 소스는 DOIK 스터디였고, 지나가듯 나오는 kubestr fio 테스트를 해보려 이것저것 연습 수준에서 경험을 하게 됐다:
- 우분투에 NFS 서버를 구성할 수 있다.
- kubestr로 저장소 클래스를 fio로 벤치마크 해 볼 수 있다.
- Disk Pressure로 파드 스케줄이 안되는 현상을 트러블슈팅 해 보았다.

---

## 참고
- https://ko.linux-console.net/?p=631
- https://linux.die.net/man/5/exports
- http://nfs.sourceforge.net/
- https://fio.readthedocs.io/en/latest/fio_doc.html
- https://man7.org/linux/man-pages/man1/sar.1.html
- https://javawebigdata.tistory.com/entry/Kubernetes-%EA%B4%80%EB%A6%AC-taint-toleration-%EB%AC%B8%EC%A0%9C-disk-pressure
- https://gasidaseo.notion.site/e49b329c833143d4a3b9715d75b5078d
