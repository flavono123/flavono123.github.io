---
title: "SSL/TLS 인증서 동작 이해하기 1"
date: 2022-01-12T23:21:42+09:00
tags:
- websecure
- cryptography
---

작년에 회사에서 SSL/TLS 인증서 갱신 작업을 했다. 회사가 합병과 이사를 해서 정보가 바뀌어 CSR 생성부터 모든 과정을 했다. 당시엔 연마다 반복해야하는 작업이니 절차를 매뉴얼화 했다. 실제로 권한이 있고 명령어 복붙하면 비개발자 또는 웹 통신과 보안에 이해가 전혀 없는 사람도 할 수 있을 정도로 만들었다.

올해 다시 인증서 갱신하려 보니 내가 그런 사람인거 같았다. 실제로 인증서의 암호화나 인증 과정을 정확히 몰랐다. 따라서 SSL/TLS 인증과 관련한 키워드 검색해서 공부했다. 하지만 내가 단번에 이해할 수 있는 글은 없었다(물론 크게 도움이 된 글도 있다). 너무 추상화된 설명 또는 실제 인증서나 키를 예제로 사용해서 생략되는 부분도 많았다.

이 부분에 끄덕였다면 인증서에 대해 잘 모르고 있다는 반증이다. 인증서는 노출되어도 아무 문제가 없다. 인증서는 누구나 볼 수 있다(노출하면 안되는 것은 키 중에서도 '비밀키'이다).

여러 글들을 읽어가며 나는 실제 TLS 인증서를 흉내내어 로컬에 하고 테스트 해볼 수 있었다(**마참내!**). 물론 그 중에선 더 이상 자세히 들여다 볼 수 없는 부분도 있었다. 하지만 SSL/TLS 인증서가 어떻게 동작하는지 전보다 명확히 이해 되어 글로 정리한다.

이 글을 이런 사람이 읽으면 좋다(aka 나 같은 사람):
- SSL/TLS 인증서를 발급, 갱신 해봤지만 정확히 '인증'이 어떻게 동작하는지 모른다.
- RSA, PKI, SHA256 등 암호 관련 약어, 키워드는 많이 들어봤지만 정확히 어떤 뜻인지 모른다.
- 공개키, 개인키, 대칭/비대칭키 등의 암호학 개념이 추상적인 그림으로만 있고 실제 파일에 쓰인 값과 매칭이 안되는 사람.
- 그리고 나 같은 사람 말고 암호학, 웹 보안 고수가 이 글을 읽고 지적해주시면 고맙겠습니다.

이 글은, 내가 궁금했고 이를 해소한 과정을 재구성하여, 다음 순서로 진행된다:
1. 실제 인증서를 보고 궁금한 점을 찾아 본다.
2. 인증서를 발급하고 인증되는 과정을 최대한 실제와 가깝게 실습한다(Self-signed certificate).
3. 나머지 생략한 개념들을 키워드 별로 정리한다.


## 인증서를 보자
우리 회사 인증서를 예로 들겠다. 크롬 주소창에 cre.ma를 입력해 접속하고 주소창 왼쪽 자물쇠 표시를 눌러 인증서를 확인하자:
![1.gif](/images/understand-tls-certificate-with-hand-on-practice-1/1.gif)

'이 사이트는 보안 연결(HTTPS)이 사용되었습니다.' -> '인증서가 유효함' 순서대로 클릭하면 크롬 중앙 모달에 인증서가 뜬다. 세부사항 신뢰 등을 누르면 인증서가 어떻게 생겼는지 알 수 있다.

여기서 첫번째 궁금증은 "내가 갱신해서 받았던 인증서는 이렇게 안생겼는데"였다. MII...로 시작하는 해시 값 덩어리였다. 세부사항을 눌러서 보이는 제목 이름 이란 섹션은 CSR 생성 시 회사 정보를 입력했던 기억이 있지만 나머지는 뭐지?

두번째 궁금증은 "인증서가 여러개네?"였다. 인증 업체인 DigiCert에 인증서를 요청하고 받아 쓰긴 한다. DigiCert Global Root CA -> DiCert TLS RSA SHA256 2020 CA1 -> 발급 받은 인증서의 3단계로 각각 인증서가 있는걸로 보인다. 인증 업체 이름 인증서들의 역할은 뭘까?

### 더 자세히 보자
터미널에서 [openssl](https://www.openssl.org/)을 사용하여 인증서를 확인해보자. openssl은 해시 함수 암/복호화나 인증서 생성 등 SSL 관한 모든 동작을 할 수 있다. 로컬 맥북에서 했고 homebrew를 통해 최신인 3버전을 설치했다([`brew install openssl@3`](https://formulae.brew.sh/formula/openssl@3)). PATH 앞에 설치한 openssl@3의 bin 경로를 추가하자:

```sh
❯ openssl version
OpenSSL 3.0.1 14 Dec 2021 (Library: OpenSSL 3.0.1 14 Dec 2021)
❯ openssl s_client -connect cre.ma:443 < /dev/null
CONNECTED(00000005)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
verify return:1
depth=1 C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
verify return:1
depth=0 C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
verify return:1
---
Certificate chain
 0 s:C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
   i:C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 21 00:00:00 2021 GMT; NotAfter: Feb 20 23:59:59 2022 GMT
 1 s:C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Sep 24 00:00:00 2020 GMT; NotAfter: Sep 23 23:59:59 2030 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIGKjCCBRKgAwIBAgIQA8E00cotfeJu6tyUUdcRFDANBgkqhkiG9w0BAQsFADBP
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMSkwJwYDVQQDEyBE
aWdpQ2VydCBUTFMgUlNBIFNIQTI1NiAyMDIwIENBMTAeFw0yMTAxMjEwMDAwMDBa
Fw0yMjAyMjAyMzU5NTlaMFwxCzAJBgNVBAYTAktSMQ4wDAYDVQQIEwVTZW91bDEV
MBMGA1UEBxMMU2Vvbmdkb25nLWd1MRMwEQYDVQQKEwpDUkVNQSBJbmMuMREwDwYD
VQQDDAgqLmNyZS5tYTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMoZ
maCa7F+E0lrzC+qV3VxQQRIGqfdVTcCcFxdyJjHuYxeKgiy3lGv/vCWLlQrJM+bJ
3ZI9t3P+ZOO7aZ1PzbRx3FWTP3CFW+22uXpPY5qv1EM3H9fV/FBELp/ScBdf5jcz
/5skFZx7vEGNm9XVNTdtxJe3csVJTf8dOHgcyPb/ikihtgN99CIMZ5Dik0bnFNhH
S+O7S0V8kLNURvRLPb/QKX4kp89BzTaUELBMZQHwlbwGlcCfhx4cXXNmHIWt8nCh
sMT11gKttM1nLhQId3p2doURTn6okutmheEFFKUhO7ep9OG4a/Dppd4e/AWw8Dhg
U1o5jnN5FFlU7DfcIpcCAwEAAaOCAvMwggLvMB8GA1UdIwQYMBaAFLdrouqoqoSM
eeq02g+YssWVdrn0MB0GA1UdDgQWBBRy9XXSRR3z1uzoF4Xn4tnKTycQ4DAbBgNV
HREEFDASgggqLmNyZS5tYYIGY3JlLm1hMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUE
FjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwgYsGA1UdHwSBgzCBgDA+oDygOoY4aHR0
cDovL2NybDMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExTUlNBU0hBMjU2MjAyMENB
MS5jcmwwPqA8oDqGOGh0dHA6Ly9jcmw0LmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydFRM
U1JTQVNIQTI1NjIwMjBDQTEuY3JsMD4GA1UdIAQ3MDUwMwYGZ4EMAQICMCkwJwYI
KwYBBQUHAgEWG2h0dHA6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzB9BggrBgEFBQcB
AQRxMG8wJAYIKwYBBQUHMAGGGGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBHBggr
BgEFBQcwAoY7aHR0cDovL2NhY2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExT
UlNBU0hBMjU2MjAyMENBMS5jcnQwDAYDVR0TAQH/BAIwADCCAQQGCisGAQQB1nkC
BAIEgfUEgfIA8AB2ACl5vvCeOTkh8FZzn2Old+W+V32cYAr4+U1dJlwlXceEAAAB
dyJlOdcAAAQDAEcwRQIhAKWWY4duHzlJlXL3O9C4hU9EjhRcmDHBnAMdlyWudMeu
AiAA0ueN1JP/tDVhzJJ96gjhU8JhoVLTc5qPxPycQwBrzgB2ACJFRQdZVSRWlj+h
L/H3bYbgIyZjrcBLf13Gg1xu4g8CAAABdyJlOiYAAAQDAEcwRQIhAJ5E73JvM8Y7
HU1veJ7PlWbbxrdtw2z9w9xU+7/NAlNuAiAZ2+PGAAdrLjDMnL1CaBlII8kbRYdn
2U4W8bEp5xeWkDANBgkqhkiG9w0BAQsFAAOCAQEArXuG/kKbof6qvgd/CmMf8o3i
61dHzjdO09Vmw73K9wFiggLb3Dt6cxPLfplI9I+2+3K9xkuyiNSghClmiE8+AZsR
pucRi1hZYrckj52AcSZ9WXzaBssMMDtXipmg912sz3cjBqcEhZwbxka5I8ih0PAP
VEXzsZJTbokVD1Nb4SfJuDqgiBARmLiwHi7/b+RGtfHkGsiIYID6VCXPDvWgja4/
qBdZtYG+Qw0K8Iqu0x0pGewq4t6RuyMaTOBj5DvhIuQTqNFWxj5iuTlZpkPUrg9B
KJyRw0fr/aKxqYHbXPc7VAV22k2sok2ihd9uX5JphzCM47b/XhVo6SX7PQ7DRg==
-----END CERTIFICATE-----
subject=C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
issuer=C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 3508 bytes and written 405 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: 3BB10824D86FEADBB2170BB2B728BE31A37A4710E45911E291B200EF2B9F0175
    Session-ID-ctx: 
    Master-Key: 1F4C2B8BDCFAC20C2BF10E65A91F5930EA5DEC664F044E9BEA892B97AE44457FFD856E418C6DFFC3BD96B709ED112707
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 600 (seconds)
    TLS session ticket:
    0000 - b4 2c 8f 0d b9 ca 25 51-49 24 48 ba 4c 97 56 63   .,....%QI$H.L.Vc
    0010 - 35 9c e6 49 0d c3 9d 9b-cc 8f a9 dd fe 21 d9 2d   5..I.........!.-
    0020 - a1 c7 d8 5f 36 01 f7 dd-ba 6b e9 41 a8 6a 52 6d   ..._6....k.A.jRm
    0030 - 4e 2b 14 f3 51 c3 e9 f9-04 5c 6d f2 23 52 33 2d   N+..Q....\m.#R3-
    0040 - 7e 4a f7 36 7e b6 65 4d-b5 23 59 b6 ac 77 00 5a   ~J.6~.eM.#Y..w.Z
    0050 - ab d4 92 b7 05 a7 93 62-ec c1 b8 c6 be 02 0f 3c   .......b.......<
    0060 - c1 72 f9 5c e8 f3 67 1e-74 88 7e 05 ea 5d 53 d9   .r.\..g.t.~..]S.
    0070 - 20 4a 4f f9 1b ab 87 0a-9c f4 aa 65 87 01 b1 70    JO........e...p
    0080 - e5 1f 6e 77 03 ac 8f 6e-c7 ee d6 39 4d 7b 97 59   ..nw...n...9M{.Y
    0090 - 5a f6 49 22 dc 60 c4 0f-6f e7 47 96 a5 7e e0 54   Z.I".`..o.G..~.T
    00a0 - 44 2b e1 fc 0e 10 22 d1-01 6e d6 fe d6 96 f4 78   D+...."..n.....x
    00b0 - 19 2a f9 ce be 93 08 45-d4 b0 4b 6e 1b 1b 1a 65   .*.....E..Kn...e

    Start Time: 1642045212
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: yes
---
DONE
```
 - `s_client`: `-connect` 옵션으로 목적지 포트에 TLS 연결을 한다.
 - `< /dev/null`: SSL 연결 후 메세지 입력을 기다리는데 우린 인증서만 필요하기 때문에 연결만 했다.


이 명령은 브라우저에서 화면 렌더하는 부분 직전까지와 같다. 즉 SSL 연결까지 한다. 명령 결과에서 얻을 수 있는 정보가 더 많아 보인다. 각 부분을 살펴 보면:
```sh
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
verify return:1
depth=1 C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
verify return:1
depth=0 C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
verify return:1
```
1. 먼저 depth가 2,1,0 내림차순으로 있다. CommonName(CN)을 보면 브라우저에서 본 인증서와 같이 DigiCert 인증서들이 있고 내가 설치한 회사 인증서(편의상 앞으로 subject라고 함)가 있다.

```sh
Certificate chain
 0 s:C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
   i:C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 21 00:00:00 2021 GMT; NotAfter: Feb 20 23:59:59 2022 GMT
 1 s:C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Sep 24 00:00:00 2020 GMT; NotAfter: Sep 23 23:59:59 2030 GMT
```
2. Certificate Chain이란 부분이다. 위의 세 인증서 사이를 이어주는 고리이다:
- s: subject(DN)
- i: issuer(발급자 DN)
- a: algorithm?
- v: Verification Dates
이다. subject는 "DigiCert TLS RSA SHA256 2020 CA1"가, 또 그 인증서는 "DigiCert Global Root CA"가 인증해준 것이다.

```
Server certificate
-----BEGIN CERTIFICATE-----
MIIGKjCCBRKgAwIBAgIQA8E00cotfeJu6tyUUdcRFDANBgkqhkiG9w0BAQsFADBP
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMSkwJwYDVQQDEyBE
aWdpQ2VydCBUTFMgUlNBIFNIQTI1NiAyMDIwIENBMTAeFw0yMTAxMjEwMDAwMDBa
Fw0yMjAyMjAyMzU5NTlaMFwxCzAJBgNVBAYTAktSMQ4wDAYDVQQIEwVTZW91bDEV
MBMGA1UEBxMMU2Vvbmdkb25nLWd1MRMwEQYDVQQKEwpDUkVNQSBJbmMuMREwDwYD
VQQDDAgqLmNyZS5tYTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMoZ
maCa7F+E0lrzC+qV3VxQQRIGqfdVTcCcFxdyJjHuYxeKgiy3lGv/vCWLlQrJM+bJ
3ZI9t3P+ZOO7aZ1PzbRx3FWTP3CFW+22uXpPY5qv1EM3H9fV/FBELp/ScBdf5jcz
/5skFZx7vEGNm9XVNTdtxJe3csVJTf8dOHgcyPb/ikihtgN99CIMZ5Dik0bnFNhH
S+O7S0V8kLNURvRLPb/QKX4kp89BzTaUELBMZQHwlbwGlcCfhx4cXXNmHIWt8nCh
sMT11gKttM1nLhQId3p2doURTn6okutmheEFFKUhO7ep9OG4a/Dppd4e/AWw8Dhg
U1o5jnN5FFlU7DfcIpcCAwEAAaOCAvMwggLvMB8GA1UdIwQYMBaAFLdrouqoqoSM
eeq02g+YssWVdrn0MB0GA1UdDgQWBBRy9XXSRR3z1uzoF4Xn4tnKTycQ4DAbBgNV
HREEFDASgggqLmNyZS5tYYIGY3JlLm1hMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUE
FjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwgYsGA1UdHwSBgzCBgDA+oDygOoY4aHR0
cDovL2NybDMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExTUlNBU0hBMjU2MjAyMENB
MS5jcmwwPqA8oDqGOGh0dHA6Ly9jcmw0LmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydFRM
U1JTQVNIQTI1NjIwMjBDQTEuY3JsMD4GA1UdIAQ3MDUwMwYGZ4EMAQICMCkwJwYI
KwYBBQUHAgEWG2h0dHA6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzB9BggrBgEFBQcB
AQRxMG8wJAYIKwYBBQUHMAGGGGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBHBggr
BgEFBQcwAoY7aHR0cDovL2NhY2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExT
UlNBU0hBMjU2MjAyMENBMS5jcnQwDAYDVR0TAQH/BAIwADCCAQQGCisGAQQB1nkC
BAIEgfUEgfIA8AB2ACl5vvCeOTkh8FZzn2Old+W+V32cYAr4+U1dJlwlXceEAAAB
dyJlOdcAAAQDAEcwRQIhAKWWY4duHzlJlXL3O9C4hU9EjhRcmDHBnAMdlyWudMeu
AiAA0ueN1JP/tDVhzJJ96gjhU8JhoVLTc5qPxPycQwBrzgB2ACJFRQdZVSRWlj+h
L/H3bYbgIyZjrcBLf13Gg1xu4g8CAAABdyJlOiYAAAQDAEcwRQIhAJ5E73JvM8Y7
HU1veJ7PlWbbxrdtw2z9w9xU+7/NAlNuAiAZ2+PGAAdrLjDMnL1CaBlII8kbRYdn
2U4W8bEp5xeWkDANBgkqhkiG9w0BAQsFAAOCAQEArXuG/kKbof6qvgd/CmMf8o3i
61dHzjdO09Vmw73K9wFiggLb3Dt6cxPLfplI9I+2+3K9xkuyiNSghClmiE8+AZsR
pucRi1hZYrckj52AcSZ9WXzaBssMMDtXipmg912sz3cjBqcEhZwbxka5I8ih0PAP
VEXzsZJTbokVD1Nb4SfJuDqgiBARmLiwHi7/b+RGtfHkGsiIYID6VCXPDvWgja4/
qBdZtYG+Qw0K8Iqu0x0pGewq4t6RuyMaTOBj5DvhIuQTqNFWxj5iuTlZpkPUrg9B
KJyRw0fr/aKxqYHbXPc7VAV22k2sok2ihd9uX5JphzCM47b/XhVo6SX7PQ7DRg==
-----END CERTIFICATE-----
subject=C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
issuer=C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
```

3. 앞서 말한 해시 값인 인증서 파일이다. subject, issuer의 DN이 한번 더 쓰여 있다.

```sh
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
```

4. TLS handshake 과정에서 [클라이언트가 추가적인 CA names 목록을 주는 것](https://www.openssl.org/docs/man1.1.1/man3/SSL_get0_peer_CA_list.html)에 대한 내용이다. 명령에서 이 옵션을 사용하지 않았고 글에서 handshake는 자세히 설명하지 않는다.

```sh
SSL handshake has read 3508 bytes and written 405 bytes
Verification: OK
```

5. SSL handshake가 맺어졌다.

```sh
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    Session-ID: 3BB10824D86FEADBB2170BB2B728BE31A37A4710E45911E291B200EF2B9F0175
    Session-ID-ctx: 
    Master-Key: 1F4C2B8BDCFAC20C2BF10E65A91F5930EA5DEC664F044E9BEA892B97AE44457FFD856E418C6DFFC3BD96B709ED112707
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 600 (seconds)
    TLS session ticket:
    0000 - b4 2c 8f 0d b9 ca 25 51-49 24 48 ba 4c 97 56 63   .,....%QI$H.L.Vc
    0010 - 35 9c e6 49 0d c3 9d 9b-cc 8f a9 dd fe 21 d9 2d   5..I.........!.-
    0020 - a1 c7 d8 5f 36 01 f7 dd-ba 6b e9 41 a8 6a 52 6d   ..._6....k.A.jRm
    0030 - 4e 2b 14 f3 51 c3 e9 f9-04 5c 6d f2 23 52 33 2d   N+..Q....\m.#R3-
    0040 - 7e 4a f7 36 7e b6 65 4d-b5 23 59 b6 ac 77 00 5a   ~J.6~.eM.#Y..w.Z
    0050 - ab d4 92 b7 05 a7 93 62-ec c1 b8 c6 be 02 0f 3c   .......b.......<
    0060 - c1 72 f9 5c e8 f3 67 1e-74 88 7e 05 ea 5d 53 d9   .r.\..g.t.~..]S.
    0070 - 20 4a 4f f9 1b ab 87 0a-9c f4 aa 65 87 01 b1 70    JO........e...p
    0080 - e5 1f 6e 77 03 ac 8f 6e-c7 ee d6 39 4d 7b 97 59   ..nw...n...9M{.Y
    0090 - 5a f6 49 22 dc 60 c4 0f-6f e7 47 96 a5 7e e0 54   Z.I".`..o.G..~.T
    00a0 - 44 2b e1 fc 0e 10 22 d1-01 6e d6 fe d6 96 f4 78   D+...."..n.....x
    00b0 - 19 2a f9 ce be 93 08 45-d4 b0 4b 6e 1b 1b 1a 65   .*.....E..Kn...e

    Start Time: 1642045212
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: yes
---
DONE
```

6. Handshake 후 맺어진 SSL session에 대한 내용이다. 연결마다 ID 같은 값들은 달라지게 된다. Verify return code를 보고 연결이, 즉 인증이 잘 된것만 확인하자.

Handshake 과정은 생략하고 Certificate chain이 무엇인지, 인증서는 어떤 정보를 담고 있는지 자세히 보자.

### X509

- Certificate chain은 브라우저 인증서에서도 볼 수 있었다.
- openssl s_client 요청에선 인증서가 파일 그대로 왔고, 브라우저 인증서에선 세부 사항이라는 구조로 보였다.

먼저 후자에 대해서 알아보자. 명령 출력에서 보이는 포맷을 PEM이라고 한다. 인증서 파일 확장자도 .pem이다. PEM이 어떤 포맷인지는 나중에 설명한다. 하지만 눈으로 봤을 때 임의의 해시 값처럼 보여 이건 PEM이라고 알아볼 수 있다.

이 PEM으로 쓰인 인증서를 파일로 저장하고:
```sh
❯ openssl s_client -connect cre.ma:443 2>/dev/null </dev/null | gsed -n '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cre_ma.pem

❯ cat cre_ma.pem 
-----BEGIN CERTIFICATE-----
MIIGKjCCBRKgAwIBAgIQA8E00cotfeJu6tyUUdcRFDANBgkqhkiG9w0BAQsFADBP
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMSkwJwYDVQQDEyBE
aWdpQ2VydCBUTFMgUlNBIFNIQTI1NiAyMDIwIENBMTAeFw0yMTAxMjEwMDAwMDBa
Fw0yMjAyMjAyMzU5NTlaMFwxCzAJBgNVBAYTAktSMQ4wDAYDVQQIEwVTZW91bDEV
MBMGA1UEBxMMU2Vvbmdkb25nLWd1MRMwEQYDVQQKEwpDUkVNQSBJbmMuMREwDwYD
VQQDDAgqLmNyZS5tYTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMoZ
maCa7F+E0lrzC+qV3VxQQRIGqfdVTcCcFxdyJjHuYxeKgiy3lGv/vCWLlQrJM+bJ
3ZI9t3P+ZOO7aZ1PzbRx3FWTP3CFW+22uXpPY5qv1EM3H9fV/FBELp/ScBdf5jcz
/5skFZx7vEGNm9XVNTdtxJe3csVJTf8dOHgcyPb/ikihtgN99CIMZ5Dik0bnFNhH
S+O7S0V8kLNURvRLPb/QKX4kp89BzTaUELBMZQHwlbwGlcCfhx4cXXNmHIWt8nCh
sMT11gKttM1nLhQId3p2doURTn6okutmheEFFKUhO7ep9OG4a/Dppd4e/AWw8Dhg
U1o5jnN5FFlU7DfcIpcCAwEAAaOCAvMwggLvMB8GA1UdIwQYMBaAFLdrouqoqoSM
eeq02g+YssWVdrn0MB0GA1UdDgQWBBRy9XXSRR3z1uzoF4Xn4tnKTycQ4DAbBgNV
HREEFDASgggqLmNyZS5tYYIGY3JlLm1hMA4GA1UdDwEB/wQEAwIFoDAdBgNVHSUE
FjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwgYsGA1UdHwSBgzCBgDA+oDygOoY4aHR0
cDovL2NybDMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExTUlNBU0hBMjU2MjAyMENB
MS5jcmwwPqA8oDqGOGh0dHA6Ly9jcmw0LmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydFRM
U1JTQVNIQTI1NjIwMjBDQTEuY3JsMD4GA1UdIAQ3MDUwMwYGZ4EMAQICMCkwJwYI
KwYBBQUHAgEWG2h0dHA6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzB9BggrBgEFBQcB
AQRxMG8wJAYIKwYBBQUHMAGGGGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBHBggr
BgEFBQcwAoY7aHR0cDovL2NhY2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExT
UlNBU0hBMjU2MjAyMENBMS5jcnQwDAYDVR0TAQH/BAIwADCCAQQGCisGAQQB1nkC
BAIEgfUEgfIA8AB2ACl5vvCeOTkh8FZzn2Old+W+V32cYAr4+U1dJlwlXceEAAAB
dyJlOdcAAAQDAEcwRQIhAKWWY4duHzlJlXL3O9C4hU9EjhRcmDHBnAMdlyWudMeu
AiAA0ueN1JP/tDVhzJJ96gjhU8JhoVLTc5qPxPycQwBrzgB2ACJFRQdZVSRWlj+h
L/H3bYbgIyZjrcBLf13Gg1xu4g8CAAABdyJlOiYAAAQDAEcwRQIhAJ5E73JvM8Y7
HU1veJ7PlWbbxrdtw2z9w9xU+7/NAlNuAiAZ2+PGAAdrLjDMnL1CaBlII8kbRYdn
2U4W8bEp5xeWkDANBgkqhkiG9w0BAQsFAAOCAQEArXuG/kKbof6qvgd/CmMf8o3i
61dHzjdO09Vmw73K9wFiggLb3Dt6cxPLfplI9I+2+3K9xkuyiNSghClmiE8+AZsR
pucRi1hZYrckj52AcSZ9WXzaBssMMDtXipmg912sz3cjBqcEhZwbxka5I8ih0PAP
VEXzsZJTbokVD1Nb4SfJuDqgiBARmLiwHi7/b+RGtfHkGsiIYID6VCXPDvWgja4/
qBdZtYG+Qw0K8Iqu0x0pGewq4t6RuyMaTOBj5DvhIuQTqNFWxj5iuTlZpkPUrg9B
KJyRw0fr/aKxqYHbXPc7VAV22k2sok2ihd9uX5JphzCM47b/XhVo6SX7PQ7DRg==
-----END CERTIFICATE-----
```

다음 명령으로 파싱한다:
```sh
❯ openssl x509 -text -noout -in cre_ma.pem 
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            03:c1:34:d1:ca:2d:7d:e2:6e:ea:dc:94:51:d7:11:14
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
        Validity
            Not Before: Jan 21 00:00:00 2021 GMT
            Not After : Feb 20 23:59:59 2022 GMT
        Subject: C = KR, ST = Seoul, L = Seongdong-gu, O = CREMA Inc., CN = *.cre.ma
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:ca:19:99:a0:9a:ec:5f:84:d2:5a:f3:0b:ea:95:
                    dd:5c:50:41:12:06:a9:f7:55:4d:c0:9c:17:17:72:
                    26:31:ee:63:17:8a:82:2c:b7:94:6b:ff:bc:25:8b:
                    95:0a:c9:33:e6:c9:dd:92:3d:b7:73:fe:64:e3:bb:
                    69:9d:4f:cd:b4:71:dc:55:93:3f:70:85:5b:ed:b6:
                    b9:7a:4f:63:9a:af:d4:43:37:1f:d7:d5:fc:50:44:
                    2e:9f:d2:70:17:5f:e6:37:33:ff:9b:24:15:9c:7b:
                    bc:41:8d:9b:d5:d5:35:37:6d:c4:97:b7:72:c5:49:
                    4d:ff:1d:38:78:1c:c8:f6:ff:8a:48:a1:b6:03:7d:
                    f4:22:0c:67:90:e2:93:46:e7:14:d8:47:4b:e3:bb:
                    4b:45:7c:90:b3:54:46:f4:4b:3d:bf:d0:29:7e:24:
                    a7:cf:41:cd:36:94:10:b0:4c:65:01:f0:95:bc:06:
                    95:c0:9f:87:1e:1c:5d:73:66:1c:85:ad:f2:70:a1:
                    b0:c4:f5:d6:02:ad:b4:cd:67:2e:14:08:77:7a:76:
                    76:85:11:4e:7e:a8:92:eb:66:85:e1:05:14:a5:21:
                    3b:b7:a9:f4:e1:b8:6b:f0:e9:a5:de:1e:fc:05:b0:
                    f0:38:60:53:5a:39:8e:73:79:14:59:54:ec:37:dc:
                    22:97
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier: 
                B7:6B:A2:EA:A8:AA:84:8C:79:EA:B4:DA:0F:98:B2:C5:95:76:B9:F4
            X509v3 Subject Key Identifier: 
                72:F5:75:D2:45:1D:F3:D6:EC:E8:17:85:E7:E2:D9:CA:4F:27:10:E0
            X509v3 Subject Alternative Name: 
                DNS:*.cre.ma, DNS:cre.ma
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 CRL Distribution Points: 
                Full Name:
                  URI:http://crl3.digicert.com/DigiCertTLSRSASHA2562020CA1.crl
                Full Name:
                  URI:http://crl4.digicert.com/DigiCertTLSRSASHA2562020CA1.crl
            X509v3 Certificate Policies: 
                Policy: 2.23.140.1.2.2
                  CPS: http://www.digicert.com/CPS
            Authority Information Access: 
                OCSP - URI:http://ocsp.digicert.com
                CA Issuers - URI:http://cacerts.digicert.com/DigiCertTLSRSASHA2562020CA1.crt
            X509v3 Basic Constraints: critical
                CA:FALSE
            CT Precertificate SCTs: 
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 29:79:BE:F0:9E:39:39:21:F0:56:73:9F:63:A5:77:E5:
                                BE:57:7D:9C:60:0A:F8:F9:4D:5D:26:5C:25:5D:C7:84
                    Timestamp : Jan 21 00:43:15.287 2021 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:A5:96:63:87:6E:1F:39:49:95:72:F7:
                                3B:D0:B8:85:4F:44:8E:14:5C:98:31:C1:9C:03:1D:97:
                                25:AE:74:C7:AE:02:20:00:D2:E7:8D:D4:93:FF:B4:35:
                                61:CC:92:7D:EA:08:E1:53:C2:61:A1:52:D3:73:9A:8F:
                                C4:FC:9C:43:00:6B:CE
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 22:45:45:07:59:55:24:56:96:3F:A1:2F:F1:F7:6D:86:
                                E0:23:26:63:AD:C0:4B:7F:5D:C6:83:5C:6E:E2:0F:02
                    Timestamp : Jan 21 00:43:15.366 2021 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:9E:44:EF:72:6F:33:C6:3B:1D:4D:6F:
                                78:9E:CF:95:66:DB:C6:B7:6D:C3:6C:FD:C3:DC:54:FB:
                                BF:CD:02:53:6E:02:20:19:DB:E3:C6:00:07:6B:2E:30:
                                CC:9C:BD:42:68:19:48:23:C9:1B:45:87:67:D9:4E:16:
                                F1:B1:29:E7:17:96:90
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        ad:7b:86:fe:42:9b:a1:fe:aa:be:07:7f:0a:63:1f:f2:8d:e2:
        eb:57:47:ce:37:4e:d3:d5:66:c3:bd:ca:f7:01:62:82:02:db:
        dc:3b:7a:73:13:cb:7e:99:48:f4:8f:b6:fb:72:bd:c6:4b:b2:
        88:d4:a0:84:29:66:88:4f:3e:01:9b:11:a6:e7:11:8b:58:59:
        62:b7:24:8f:9d:80:71:26:7d:59:7c:da:06:cb:0c:30:3b:57:
        8a:99:a0:f7:5d:ac:cf:77:23:06:a7:04:85:9c:1b:c6:46:b9:
        23:c8:a1:d0:f0:0f:54:45:f3:b1:92:53:6e:89:15:0f:53:5b:
        e1:27:c9:b8:3a:a0:88:10:11:98:b8:b0:1e:2e:ff:6f:e4:46:
        b5:f1:e4:1a:c8:88:60:80:fa:54:25:cf:0e:f5:a0:8d:ae:3f:
        a8:17:59:b5:81:be:43:0d:0a:f0:8a:ae:d3:1d:29:19:ec:2a:
        e2:de:91:bb:23:1a:4c:e0:63:e4:3b:e1:22:e4:13:a8:d1:56:
        c6:3e:62:b9:39:59:a6:43:d4:ae:0f:41:28:9c:91:c3:47:eb:
        fd:a2:b1:a9:81:db:5c:f7:3b:54:05:76:da:4d:ac:a2:4d:a2:
        85:df:6e:5f:92:69:87:30:8c:e3:b6:ff:5e:15:68:e9:25:fb:
        3d:0e:c3:46
```

이번엔 x509이란 서브 명령을 썼는데 이것 무엇일까?

[X.509 - 위키백과, 우리 모두의 백과사전](https://ko.wikipedia.org/wiki/X.509)

TLS 인증에 사용되는 공개 키 기반(PKI)의 표준이다. 오늘날 사용하는 인증서는 다 이 표준을 따른다고 보면 된다. 위의 출력 구조를 보면 크게 인증서, 서명 알고리즘, 서명 값 등이 보인다.

이 값들은 앞서 브라우저에서 본 인증서의 세부 사항 내용과 일치한다. 비교적 눈에 잘 보이는 subject DN, 발급자 DN부터 그 다음인 공개키 정보(Subject Public Key Info)를 매칭해볼 수 있다.

긴 X509 전체 구조가 아닌, 원하는 부분만 출력할 수 있다. 예를 들어 공개 키의 modulus 부분을 보고 싶으면,  전체를 출력하는 `-text` 대신, `-modulus` 옵션으로 명령하면 된다:
```
❯ openssl x509 -modulus -noout -in cre_ma.pem
Modulus=CA1999A09AEC5F84D25AF30BEA95DD5C50411206A9F7554DC09C1717722631EE63178A822CB7946BFFBC258B950AC933E6C9DD923DB773FE64E3BB699D4FCDB471DC55933F70855BEDB6B97A4F639AAFD443371FD7D5FC50442E9FD270175FE63733FF9B24159C7BBC418D9BD5D535376DC497B772C5494DFF1D38781CC8F6FF8A48A1B6037DF4220C6790E29346E714D8474BE3BB4B457C90B35446F44B3DBFD0297E24A7CF41CD369410B04C6501F095BC0695C09F871E1C5D73661C85ADF270A1B0C4F5D602ADB4CD672E1408777A767685114E7EA892EB6685E10514A5213BB7A9F4E1B86BF0E9A5DE1EFC05B0F03860535A398E7379145954EC37DC2297
```

이는 브라우저에서 보이는 공개키(256바이트)와 더 유사하다(위 `-text` 출력에서 나온 결과는 270바이트다). 이것은 RSA로 암호화된 공개키이다. RSA는 나중에 자세히 설명 한다.

더 많은 출력 방법은 [openssl-x509 매뉴얼](https://www.openssl.org/docs/man1.1.1/man1/x509.html)의 옵션을 참고(1.1.1 버전 매뉴얼이지만 3버전에서 하위 호환되고 이 페이지의 옵션 설명이 더 자세하다).

아직 모르는 부분이 많지만 X509 구조로 파싱해 본 결과 회사 인증서(subject)는 RSA 2048(256바이트)로 암호화 되었고 이를 확인했다. 그럼 서명(Signature)은 무엇일까? 이쯤에서 이 글을 읽고 오자.

https://gruuuuu.github.io/security/what-is-x509/

내가 직접 RSA 같은 해시 함수부터 인증서 체인의 개념까지 찾아보던 중 이 글이 전체 큰 그림을 잡는데 많은 도움이 됐다. 

## 인증서 체인

정리하면, subject는 X509라는 PKI로 인증한다. 공개키는 뿌리고 비밀키를 subject가 가지고 있어 subject가 암호화한 데이터를 공개키로 복호화하여 본다. 이때 subject가 뿌린 공개키가 유효하다고 인증해 주는 것이 인증 기관(CA)이다. CA는 디지털 서명을 통해 인증서를 암호화해서 발급한다. CA 역시 인증서 형태로 공개키가 제공되는 구조이다. 이걸 Chain of Trust라 하고, 결국 만나는 최상위 루트 CA는 OS 또는 브라우저나 openssl 같은 프로그램에 내장되어 있다. 루트 CA 인증서는 모두가 신뢰한다고 가정한다.

여기서 인증 자체에 사용하는 암호화 알고리즘은 RSA이다:
```sh
❯ openssl x509 -text -noout -in cre_ma.pem | grep "Public Key Algorithm"
            Public Key Algorithm: rsaEncryption
```

그리고 CA에서 서명 알고리즘은 SHA256이다:
```
❯ openssl x509 -text -noout -in cre_ma.pem | grep "Signature Algorithm"
        Signature Algorithm: sha256WithRSAEncryption
    Signature Algorithm: sha256WithRSAEncryption
```

우리 예제에서 CA와 subject 인증서는 다음과 같다:
- 루트: DigiCert Global Root CA
- 중간자: DigiCert TLS RSA SHA256 2020 CA1
- subject: *.cre.ma

상위 CA 서명이라는 chain 때문에 인증서를 더 자세하게 들여다 보는건 어렵다(CA의 서명 비밀키가 있어야 인증서 복호화가 가능할 것이다). 대신 openssl 명령을 통해 chain 과정이 어떻게 되는지 또 각 CA의 모든 인증서가 어디에 위치하는지 찾아보자.

### SSL handshake
HTTPS로 통신하면 TCP 연결을 맺는 3 way handshake 후에 SSL handshake를 맺어 인증서를 교환 후 통신한다. 사실 이 과정은 `openssl s_client -connect` 할 때에도 맺어졌는데, 여기에 `-msg` 옵션을 주면 디버그 해볼 수 있다. `>>>`는 클라이언트에서 서버 반대로 `<<<`는 서버에서 클라이언트로 보내는 메세지 종류를 보여준다. 패킷은 덤프하지 않을거라 생략했다(나중에 tcpdump 사용법을 익히면 해볼것!). 이번엔 인증 체인 depth를 보여주는 표준 에러도 표시하지 않았다:
```sh
❯ openssl s_client -connect cre.ma:443 -msg 2>/dev/null < /dev/null | grep Handshake
>>> TLS 1.3, Handshake [length 0133], ClientHello
<<< TLS 1.3, Handshake [length 0045], ServerHello
<<< TLS 1.2, Handshake [length 0b29], Certificate
<<< TLS 1.2, Handshake [length 012c], ServerKeyExchange
<<< TLS 1.2, Handshake [length 0004], ServerHelloDone
>>> TLS 1.2, Handshake [length 0025], ClientKeyExchange
>>> TLS 1.2, Handshake [length 0010], Finished
<<< TLS 1.2, Handshake [length 00ca], NewSessionTicket
<<< TLS 1.2, Handshake [length 0010], Finished
```

SSL handshake 과정 설명은 [이 글](https://aws-hyoh.tistory.com/entry/HTTPS-%ED%86%B5%EC%8B%A0%EA%B3%BC%EC%A0%95-%EC%89%BD%EA%B2%8C-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0-3SSL-Handshake)로 대체한다

### 중간자 인증서

`s_client` 명령에  `-showcerts` 옵션을 추가하면 subject 뿐만 아니라 중간자 인증서도 확인할 수 있다(결과에서 중복 부분은 제거하고 중간자 인증서만 보이게 함):
```sh
❯ openssl s_client -connect cre.ma:443 -showcerts < /dev/null
...
Certificate chain
...
1 s:C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Sep 24 00:00:00 2020 GMT; NotAfter: Sep 23 23:59:59 2030 GMT
-----BEGIN CERTIFICATE-----
MIIE6jCCA9KgAwIBAgIQCjUI1VwpKwF9+K1lwA/35DANBgkqhkiG9w0BAQsFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0yMDA5MjQwMDAwMDBaFw0zMDA5MjMyMzU5NTlaME8xCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxKTAnBgNVBAMTIERpZ2lDZXJ0IFRMUyBS
U0EgU0hBMjU2IDIwMjAgQ0ExMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAwUuzZUdwvN1PWNvsnO3DZuUfMRNUrUpmRh8sCuxkB+Uu3Ny5CiDt3+PE0J6a
qXodgojlEVbbHp9YwlHnLDQNLtKS4VbL8Xlfs7uHyiUDe5pSQWYQYE9XE0nw6Ddn
g9/n00tnTCJRpt8OmRDtV1F0JuJ9x8piLhMbfyOIJVNvwTRYAIuE//i+p1hJInuW
raKImxW8oHzf6VGo1bDtN+I2tIJLYrVJmuzHZ9bjPvXj1hJeRPG/cUJ9WIQDgLGB
Afr5yjK7tI4nhyfFK3TUqNaX3sNk+crOU6JWvHgXjkkDKa77SU+kFbnO8lwZV21r
eacroicgE7XQPUDTITAHk+qZ9QIDAQABo4IBrjCCAaowHQYDVR0OBBYEFLdrouqo
qoSMeeq02g+YssWVdrn0MB8GA1UdIwQYMBaAFAPeUDVW0Uy7ZvCj4hsbw5eyPdFV
MA4GA1UdDwEB/wQEAwIBhjAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIw
EgYDVR0TAQH/BAgwBgEB/wIBADB2BggrBgEFBQcBAQRqMGgwJAYIKwYBBQUHMAGG
GGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBABggrBgEFBQcwAoY0aHR0cDovL2Nh
Y2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0R2xvYmFsUm9vdENBLmNydDB7BgNV
HR8EdDByMDegNaAzhjFodHRwOi8vY3JsMy5kaWdpY2VydC5jb20vRGlnaUNlcnRH
bG9iYWxSb290Q0EuY3JsMDegNaAzhjFodHRwOi8vY3JsNC5kaWdpY2VydC5jb20v
RGlnaUNlcnRHbG9iYWxSb290Q0EuY3JsMDAGA1UdIAQpMCcwBwYFZ4EMAQEwCAYG
Z4EMAQIBMAgGBmeBDAECAjAIBgZngQwBAgMwDQYJKoZIhvcNAQELBQADggEBAHer
t3onPa679n/gWlbJhKrKW3EX3SJH/E6f7tDBpATho+vFScH90cnfjK+URSxGKqNj
OSD5nkoklEHIqdninFQFBstcHL4AGw+oWv8Zu2XHFq8hVt1hBcnpj5h232sb0HIM
ULkwKXq/YFkQZhM6LawVEWwtIwwCPgU7/uWhnOKK24fXSuhe50gG66sSmvKvhMNb
g0qZgYOrAKHKCjxMoiWJKiKnpPMzTFuMLhoClw+dj20tlQj7T9rxkTgl4ZxuYRiH
as6xuwAwapu3r9rxxZf+ingkquqTgLozZXq8oXfpf2kUCwA/d5KxTVtzhwoT0JzI
8ks5T1KESaZMkE4f97Q=
-----END CERTIFICATE-----
...
```

명시적으로 인증서 체인을 증명 해보기 위해 복사하여 파일로 저장해두자:
```sh
❯ cat DigiCert-TLS-RSA-SHA256-2020-CA1.pem 
-----BEGIN CERTIFICATE-----
MIIE6jCCA9KgAwIBAgIQCjUI1VwpKwF9+K1lwA/35DANBgkqhkiG9w0BAQsFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0yMDA5MjQwMDAwMDBaFw0zMDA5MjMyMzU5NTlaME8xCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxKTAnBgNVBAMTIERpZ2lDZXJ0IFRMUyBS
U0EgU0hBMjU2IDIwMjAgQ0ExMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAwUuzZUdwvN1PWNvsnO3DZuUfMRNUrUpmRh8sCuxkB+Uu3Ny5CiDt3+PE0J6a
qXodgojlEVbbHp9YwlHnLDQNLtKS4VbL8Xlfs7uHyiUDe5pSQWYQYE9XE0nw6Ddn
g9/n00tnTCJRpt8OmRDtV1F0JuJ9x8piLhMbfyOIJVNvwTRYAIuE//i+p1hJInuW
raKImxW8oHzf6VGo1bDtN+I2tIJLYrVJmuzHZ9bjPvXj1hJeRPG/cUJ9WIQDgLGB
Afr5yjK7tI4nhyfFK3TUqNaX3sNk+crOU6JWvHgXjkkDKa77SU+kFbnO8lwZV21r
eacroicgE7XQPUDTITAHk+qZ9QIDAQABo4IBrjCCAaowHQYDVR0OBBYEFLdrouqo
qoSMeeq02g+YssWVdrn0MB8GA1UdIwQYMBaAFAPeUDVW0Uy7ZvCj4hsbw5eyPdFV
MA4GA1UdDwEB/wQEAwIBhjAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIw
EgYDVR0TAQH/BAgwBgEB/wIBADB2BggrBgEFBQcBAQRqMGgwJAYIKwYBBQUHMAGG
GGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBABggrBgEFBQcwAoY0aHR0cDovL2Nh
Y2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0R2xvYmFsUm9vdENBLmNydDB7BgNV
HR8EdDByMDegNaAzhjFodHRwOi8vY3JsMy5kaWdpY2VydC5jb20vRGlnaUNlcnRH
bG9iYWxSb290Q0EuY3JsMDegNaAzhjFodHRwOi8vY3JsNC5kaWdpY2VydC5jb20v
RGlnaUNlcnRHbG9iYWxSb290Q0EuY3JsMDAGA1UdIAQpMCcwBwYFZ4EMAQEwCAYG
Z4EMAQIBMAgGBmeBDAECAjAIBgZngQwBAgMwDQYJKoZIhvcNAQELBQADggEBAHer
t3onPa679n/gWlbJhKrKW3EX3SJH/E6f7tDBpATho+vFScH90cnfjK+URSxGKqNj
OSD5nkoklEHIqdninFQFBstcHL4AGw+oWv8Zu2XHFq8hVt1hBcnpj5h232sb0HIM
ULkwKXq/YFkQZhM6LawVEWwtIwwCPgU7/uWhnOKK24fXSuhe50gG66sSmvKvhMNb
g0qZgYOrAKHKCjxMoiWJKiKnpPMzTFuMLhoClw+dj20tlQj7T9rxkTgl4ZxuYRiH
as6xuwAwapu3r9rxxZf+ingkquqTgLozZXq8oXfpf2kUCwA/d5KxTVtzhwoT0JzI
8ks5T1KESaZMkE4f97Q=
-----END CERTIFICATE-----
```

x509 명령으로 공개키를 파싱할 수 있다:
```sh
 ❯ openssl x509 -pubkey -noout -in DigiCert-TLS-RSA-SHA256-2020-CA1.pem  > DigiCert-TLS-RSA-SHA256-2020-CA1.pub.pem

❯ cat DigiCert-TLS-RSA-SHA256-2020-CA1.pub.pem 
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwUuzZUdwvN1PWNvsnO3D
ZuUfMRNUrUpmRh8sCuxkB+Uu3Ny5CiDt3+PE0J6aqXodgojlEVbbHp9YwlHnLDQN
LtKS4VbL8Xlfs7uHyiUDe5pSQWYQYE9XE0nw6Ddng9/n00tnTCJRpt8OmRDtV1F0
JuJ9x8piLhMbfyOIJVNvwTRYAIuE//i+p1hJInuWraKImxW8oHzf6VGo1bDtN+I2
tIJLYrVJmuzHZ9bjPvXj1hJeRPG/cUJ9WIQDgLGBAfr5yjK7tI4nhyfFK3TUqNaX
3sNk+crOU6JWvHgXjkkDKa77SU+kFbnO8lwZV21reacroicgE7XQPUDTITAHk+qZ
9QIDAQAB
-----END PUBLIC KEY-----
```

### 루트 인증서
루트 인증서는 OS 또는 프로그램에 내장되어 있다고 했다. openssl의 경우 /usr/local/etc/openssl@3/certs/cert.pem를 바라보고 있다. 우리 예제의 루트인 DigiCert Global Root CA도 찾을 수 있다.

```sh
❯ grep Digi /usr/local/etc/openssl@3/certs/cert.pem
DigiCert Assured ID Root CA
DigiCert Global Root CA
DigiCert High Assurance EV Root CA
DigiCert Assured ID Root G2
DigiCert Assured ID Root G3
DigiCert Global Root G2
DigiCert Global Root G3
DigiCert Trusted Root G4

❯ grep -A 20 "DigiCert Global Root CA" /usr/local/etc/openssl@3/certs/cert.pem
DigiCert Global Root CA
=======================
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBhMQswCQYDVQQG
EwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cuZGlnaWNlcnQuY29tMSAw
HgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBDQTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAw
MDAwMDBaMGExCzAJBgNVBAYTAlVTMRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3
dy5kaWdpY2VydC5jb20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkq
hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsBCSDMAZOn
TjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97nh6Vfe63SKMI2tavegw5
BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt43C/dxC//AH2hdmoRBBYMql1GNXRor5H
4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7PT19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y
7vrTC0LUq7dBMtoM1O/4gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQAB
o2MwYTAOBgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbRTLtm
8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUwDQYJKoZIhvcNAQEF
BQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/EsrhMAtudXH/vTBH1jLuG2cenTnmCmr
EbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIt
tep3Sp+dWOIrWcBAI+0tKIJFPnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886
UAb3LujEV0lsYSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk
CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=
-----END CERTIFICATE-----
```

이번에도 개별 인증서와 공개키를 파일로 저장하자:
```sh
❯ cat DigiCert-Global-Root-CA.pem 
-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBhMQswCQYDVQQG
EwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cuZGlnaWNlcnQuY29tMSAw
HgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBDQTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAw
MDAwMDBaMGExCzAJBgNVBAYTAlVTMRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3
dy5kaWdpY2VydC5jb20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkq
hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsBCSDMAZOn
TjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97nh6Vfe63SKMI2tavegw5
BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt43C/dxC//AH2hdmoRBBYMql1GNXRor5H
4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7PT19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y
7vrTC0LUq7dBMtoM1O/4gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQAB
o2MwYTAOBgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbRTLtm
8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUwDQYJKoZIhvcNAQEF
BQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/EsrhMAtudXH/vTBH1jLuG2cenTnmCmr
EbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIt
tep3Sp+dWOIrWcBAI+0tKIJFPnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886
UAb3LujEV0lsYSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk
CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=
-----END CERTIFICATE-----

❯ openssl x509 -pubkey -noout -in DigiCert-Global-Root-CA.pem > DigiCert-Global-Root-CA.pub.pem

❯ cat DigiCert-Global-Root-CA.pub.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKP
C3eQyaKl7hLOllsBCSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtx
RuLWZscFs3YnFo97nh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmF
aG5cIzJLv07A6Fpt43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvU
X7Q6hL+hqkpMfT7PT19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrT
C0LUq7dBMtoM1O/4gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOv
JwIDAQAB
-----END PUBLIC KEY-----
```

### 인증서 증명
증명에 필요한 모든 상위 CA 인증서와 subject 인증서가 준비됐다. 명시적으로 모든 상위 인증서를 사용하여 증명해보면:
```sh
❯ openssl verify -CAfile DigiCert-Global-Root-CA.pem -untrusted DigiCert-TLS-RSA-SHA256-2020-CA1.pem cre_ma.pem
cre_ma.pem: OK
```

`-CAfile` 옵션엔 루트 인증서를 주고 `-untrusted` 옵션엔 중간자 인증서를 줬다. untrusted라는 이름이 좀 난해하지만 매뉴얼을 보면 정확히 중간자 인증서 역할이다 😑:

```
-untrusted filename|uri
           A file or URI of untrusted certificates to use for chain building.  This option
           can be specified more than once to load certificates from multiple sources.
```

openssl 루트 CA에 DigiCert Global Root CA가 이미 있기 때문에 `-CAfile` 옵션은 빼도 무방하다. 다른 루트 인증서를 옵션으로 주면 증명이 실패하게 된다.


## 정리
- openssl s_client 명령으로 SSL handshake 및 세션과 인증서를 확인할 수 있다.
- openssl x509 명령으로 인증서를 파싱할 수 있다.
- 전체적인 과정에 두가지 암호화가 있다
  - SSL/TLS 인증 암호화 PKI로 한다. PKI의 공개키가 인증서이다. PKI 표준 이름은 X509이고, 암호 알고리즘은 RSA.
  - Chain of Trust를 위해 CA에서 인증서(공개키)에 디지털 서명(암호화)하여 발급한다. 여기서 암호 알고리즘은 SHA256

한 포스팅에 쓰려고 했는데 예상보다 길어져서 글을 나눈다. 원래 쓰려던 분량은 3편 정도면 충분하지만, 이 글을 쓰면서 인증 매커니즘인 PKI와 디지털 서명 부분을 헷갈린다는 걸 알았다. 그래서 다 쓰면 나중에 4~5편도 나올 수 있지만 일단 3부작까지 끝내 보려고 함... 시간이 굉장히 오래 걸렸지만 글로 쓰는 순간에도 어떤걸 알고 어떤걸 모르는지 정리가 되어 쓰기 잘했단 생각이 든다(덤으로 openssl 명령이 손에 익어버렸다).

다음 글에선 A self-signed certificate이란 이름으로 많이 알려진 방법을 써서 CA에서 어떻게 인증서를 발급하는지 직접 해볼 것이다.

