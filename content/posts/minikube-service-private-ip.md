---
title: "Minikube에서 NodePort 서비스 로컬 프라이빗 IP 찾기"
date: 2022-01-07T10:44:54+09:00
tags:
- kubernetes
- minikube
---

# `minikube service --url <service-name>`

NodePort 서비스를 만들어 포트는 노출시켰는데 어느 IP로 접속할지 몰라 한참을 헤맸다...

출처: https://minikube.sigs.k8s.io/docs/handbook/accessing/#getting-the-nodeport-using-the-service-command
