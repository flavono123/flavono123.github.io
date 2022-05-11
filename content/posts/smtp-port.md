---
title: "SMTP 포트: 465 vs 587 vs 2525??"
date: 2022-05-11T15:32:18+09:00
tags:
- network
---

SMTP 포트를 직접 사용할 일이 얼마나 있을까 싶지만... [예제로 배우는 Go 프로그래밍/STMP 이메일 보내기](http://golang.site/go/article/113-SMTP-%EC%9D%B4%EB%A9%94%EC%9D%BC-%EB%B3%B4%EB%82%B4%EA%B8%B0)라는 예제를 따라하다가 쓰게 되었고, 587은 어떤 포트인지 궁금하여 검색하다 알게 된 내용을 공유한다.

이 포스트는 [What’s the Difference Between Ports 465 and 587?](https://sendgrid.com/blog/whats-the-difference-between-ports-465-and-587/)라는 글에 대한 나의 해석이다. 직역은 아니다. 다만 이 포스트에 나오는 용어, 약어 등에 대한 자세한 설명은 생략한다. 원글의 설명이나 참조 링크정도면 충분할거라 생각하고 또 설명이 없더라도 어디선가 들어본 용어인데 깊게 알지 않아도 이해에 큰 무리가 없기도 해서이다(나의 경우는 그랬다). 또 다른 나의 포스트와 달리 Hands-on lab이나 코드도 없다. 유니크한 컨텐츠인거 같아 포스팅으로 정리한다 ㅎㅎ

## 역사와 배경지식
일단 나는 587을 포함해서 465, 2525 포트에 대해서 잘 몰랐다(현실 감각이 떨어지는 포스트 제목이다).  이러한, well-known(모순적인 표현이다) 포트에 대한 명세를 지정하는 곳은 [IANA(Internet Assigned Numbers Authority)](https://ko.wikipedia.org/wiki/IANA)와 [IETF(Internet Engineering Task Force)](https://ko.wikipedia.org/wiki/%EA%B5%AD%EC%A0%9C_%EC%9D%B8%ED%84%B0%EB%84%B7_%ED%91%9C%EC%A4%80%ED%99%94_%EA%B8%B0%EA%B5%AC) 두 군데가 있다. IETF는 RFC 기반으로 인터넷 표준을 명세한다고 한다.

465는 IANA가 TLS를 적용한 STMP 보안 포트(smtps)로 출판하게 된다. 조금 나중에 IETF에서 startTLS를 적용한 STMP 포트를 587를 사용하기로 정해버렸다.

여기서 잠깐. TLS는 HTTPS에 사용되어 많이 들어봤지만, startTLS는 처음 들어봤다. StartTLS는 평문으로 시작해서 패킷 내용이 TLS 암호화 되어 있다면 TLS 통신으로 업그레이드 하는 암호 프로토콜이라고 한다. 평문과 TLS 암호문을 같은 포트를 사용할 수 있다. 한마디로 HTTP/HTTPS가 80/443 포트로 나뉘지 않고 한 포트를 사용하는 것이라고 생각하면 될 듯 하다. 하지만 TLS에 비해 실제로 사용은 많이 안되고 있는 것 같다.

다시 SMTP 역사 강의로 돌아가자. 465, 587이라는 두 SMTP 보안 포트가 충돌하던 시기에 587을 더 밀어주어 465를 폐기했다고 한다. 하지만 이미 표준으로 발표했던 465을 사용하는 곳이 있어서 혼란을 막기 위해 다시 465도 살렸다고 한다. 그래서 보안 SMTP 포트는 465, 587을 둘 다 사용할 수 있다. 앞서 말한듯 두 포트의 전송 계층 보안 구현은 다르다.

그리고 2525 포트의 존재에 대해서도 알았다. 일반 SMTP 25 포트(이건 대학교 네트워크 강의에서 들어본것도 같은 maybe-well-known 포트이다)는 인터넷 서비스 제공자의 메일 서버에서 사용하니, 그런 ISP를 사용하는 이메일 서버 제공자가 우회하기 위해 사용하는 포트라고 한다.

## 덤
앞서 소개한 Golang 튜토리얼을 진행할 때, 내 지메일 계정을 사용했다. 나는 구글 서비스 2FA를 쓰고 있어서 비밀번호 입력이 어떻게 해야할지 몰랐는데 [이 코멘트](https://gist.github.com/jpillora/cb46d183eca0710d909a?permalink_comment_id=3239532#gistcomment-3239532)에서 소개하는 방법으로 해결했다. 2FA를 사용한다면 구글 계정 로그인 후 [App password](https://security.google.com/settings/security/apppasswords)에서 디바이스용 임시 비밀번호를 만들어 가능하다.

혹시나 해서 말하면 이건 앞서 설명한 TLS와는 전혀 무관하다(말 그대로 덤이다!).

## 정리
SMTP 포트로
- 가능하다면 startTLS를 사용하는 **587** 을 쓰고
- 불가하다면 TLS를 사용하는 465 쓰자
  - 지메일은 587를 사용한다
```sh
❯ telnet smtp.gmail.com 587
Trying 64.233.189.108...
Connected to smtp.gmail.com.
Escape character is '^]'.
220 smtp.gmail.com ESMTP la19-20020a17090b161300b001cd4989fee4sm3145484pjb.48 - gsmtp
```
- 25포트는 HTTP(80)처럼 위험한 전송이다
  - 25대신 2525 포트를 사용해야 할 수도 있다

---

## 참고
- https://sendgrid.com/blog/whats-the-difference-between-ports-465-and-587/
- https://gist.github.com/jpillora/cb46d183eca0710d909a?permalink_comment_id=3239532#gistcomment-3239532
