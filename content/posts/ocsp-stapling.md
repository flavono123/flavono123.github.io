---
title: "Nginx OCSP Stapling 경고 재현하기"
date: 2022-04-26T17:41:35+09:00
tags:
- websecure
- nginx
---

```sh
nginx: [warn] "ssl_stapling" ignored, issuer certificate not found for certificate "/path/to/cert"
```

TLS 인증서를 교체하고 Nginx 설정 테스트(`nginx -t`)를 해보았는데 위 경고가 나왔다. 테스트할 서버부터 적용했는데 인증과 파일 서빙 자체는 잘 되었다. 위 메세지로 검색하다 보니 CRL, OCSP 등의 처음 보는 개념을 알게 됐고, 로컬 Nginx에서 위와 같은 문제를 재현하고 해결할 수 있었다.


### CRL
먼저 앞서 말한 CRL과 OCSP는 인증서 폐기(Certificate Revocation)에 관련한 용어이다. 보통 인증서에 문제가 있으면 새로 발급을 받더라도 이전 것이 폐기 됐는지 안했던터라 조금 생소했다. CA에서 열심히 하겠지..? 아무튼 CRL, OCSP는 이 인증서가 폐기된 인증서인지 확인하는 방법이다.

그 중 과거의 방법인 CRL(Certificate Revocation List)는 CA 폐기된 인증서 목록을 제공하고, 클라이언트는 인증시 이 목록을 보고 인증서가 유효한지 판단하는 것이다.

CRL는 인증서에 확장키 `crlDistributionPoints`에 URI 목록이 제공된다. DigiCert 인증서를 예로 들면 다음과 같다:
```sh
❯ openssl x509 -in /path/to/cert.pem -noout -ext crlDistributionPoints
X509v3 CRL Distribution Points:
    Full Name:
      URI:http://crl3.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl
    Full Name:
      URI:http://crl4.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl

# 키 이름에 확신이 없으면 그렙질로 확인
❯ openssl x509 -in /path/to/cert.pem -noout -text | grep -i crl
            X509v3 CRL Distribution Points:
                  URI:http://crl3.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl
                  URI:http://crl4.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl
```

URI는 CRL를 암호화하여 응답한다. [`openssl-crl`](https://www.openssl.org/docs/manmaster/man1/openssl-crl.html) 로 디코딩할 수 있다:
```sh
❯ wget -qO - http://crl3.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl | openssl crl -noout -text | head -25
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
        Last Update: Apr 25 06:51:39 2022 GMT
        Next Update: May  2 06:51:39 2022 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                B7:6B:A2:EA:A8:AA:84:8C:79:EA:B4:DA:0F:98:B2:C5:95:76:B9:F4
            X509v3 CRL Number:
                211
            X509v3 Issuing Distribution Point: critical
                Full Name:
                  URI:http://crl3.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl
Revoked Certificates:
    Serial Number: 048A6B01889FA7BEF34AC92DAAC36079
        Revocation Date: Jun 22 00:31:11 2021 GMT
        CRL entry extensions:
            X509v3 CRL Reason Code:
                Key Compromise
    Serial Number: 098AB8A98137F3432A18DE8C1B7F6D2F
        Revocation Date: Jul  3 05:58:20 2021 GMT
        CRL entry extensions:
            X509v3 CRL Reason Code:
                Key Compromise

❯ wget -qO - http://crl3.digicert.com/DigiCertTLSRSASHA2562020CA1-4.crl | openssl crl  -noout -text | grep "Serial Number" | wc -l
51470
```

하나의 CRL에만 폐기 인증서가 5만여개가 된다. 실제로 응답시간도 꽤 있고 head로 자르지 않으면 출력이 오래 걸릴만큼 폐기 인증서가 많다는 것을 느낄 수 있다. 이런 단점 때문에 현재 CRL는 잘 쓰이지 않는다. 실제로 현재는 비어 있는 CRL도 많다:
```sh
❯ wget -qO - http://crl4.digicert.com/TERENAeScienceSSLCA3.crl | openssl crl -noout -text
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = NL, ST = Noord-Holland, L = Amsterdam, O = TERENA, CN = TERENA eScience SSL CA 3
        Last Update: Apr 26 07:04:02 2022 GMT
        Next Update: May  3 07:04:02 2022 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                29:AA:1B:6E:30:F9:30:67:63:A5:87:26:0C:AC:F1:81:9C:69:74:49
            X509v3 CRL Number:
                31163
No Revoked Certificates.
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        c4:c8:3a:54:9a:f0:a6:57:33:0e:57:e2:03:64:3e:e1:2b:1d:
        36:1f:5f:2c:f2:9b:48:92:b4:f3:42:93:f1:e3:72:58:5b:67:
        5b:d2:0d:b3:d8:bf:d4:76:e9:3a:d7:e8:dd:42:a1:18:d5:b0:
        fa:8a:36:af:c0:07:ac:32:b2:0d:e1:8d:08:2e:80:25:4b:1c:
        6c:56:25:d8:cd:d2:a1:af:42:57:0b:de:20:7a:7a:64:98:8e:
        ce:3b:9d:07:5e:fc:ed:73:8f:c7:8d:2c:ee:53:af:2c:04:9b:
        c5:a1:21:7f:b6:5a:49:37:b7:10:91:43:5c:29:2e:68:90:bf:
        6a:d8:17:ed:8a:81:e7:51:f0:ad:0d:19:ab:3d:30:f3:1f:10:
        16:3c:c1:59:0a:9d:8c:08:e5:0c:dc:e9:2a:bb:9f:6c:2d:e9:
        2f:ac:8a:45:f1:3c:b8:14:73:aa:2b:ab:eb:25:09:c7:46:28:
        ca:84:c7:2f:96:ee:4e:7c:cf:d6:9a:65:f1:30:24:1c:75:3d:
        5a:f4:99:43:0c:f5:f1:f8:fa:70:37:17:ac:3d:96:93:89:b5:
        b8:8e:0a:8f:6d:22:dc:2c:29:58:0a:40:f2:28:ef:5f:5e:a6:
        e2:5a:eb:43:05:9b:98:e6:d4:fa:cb:c4:68:7e:f7:d8:0a:fd:
        2c:ee:10:93
```


### OCSP
이런 CRL의 단점을 보완한 방법이 OCSP(Online Certificate Status Protocol)이다. 무거운 폐기 인증서 목록을 가져와 파싱하여 폐기 여부를 확인하는 대신, 클라이언트가 확인한 인증서의 상태를 확인하는 방법이다. 상태는 "good", "revoked", "unknown" 세가지이다. OCSP를 사용하는 인증서 TLS 요청 시 응답에 포함되어 있다. [`openssl-s_client`의 `-status` 옵션](https://www.openssl.org/docs/man1.1.1/man1/openssl-s_client.html)으로 확인할 수 있다:
```sh
❯ openssl s_client -connect cre.ma:443 -status 2>&1 < /dev/null | grep -i ocsp -A10
OCSP response:
======================================
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: B76BA2EAA8AA848C79EAB4DA0F98B2C59576B9F4
    Produced At: Apr 26 06:48:49 2022 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: E4E395A229D3D4C1C31FF0980C0B4EC0098AABD8
      Issuer Key Hash: B76BA2EAA8AA848C79EAB4DA0F98B2C59576B9F4
      Serial Number: 018A11B52A0F60854F9A0EDA2A63B7AC
    Cert Status: good
```

상태는 실제 CA에 요청을 보내어 받은 응답으로, 인증서 authorityInformationAccess키에 요청 주소와 프로토콜(OCSP)가 정의되어 있다:
```sh
❯ openssl x509 -in /path/to/cert.pem -noout -text | grep -i ocsp -C1
            Authority Information Access:
                OCSP - URI:http://ocsp.digicert.com
                CA Issuers - URI:http://cacerts.digicert.com/DigiCertTLSRSASHA2562020CA1-1.crt
```

### OCSP Stapling
하나의 CA 서버로만 모든 OCSP 요청을 보내면 부하가 클 것이다. CA에서 매번 OCSP 요청에 응답하지 않고, 웹 서버에 이 응답을 캐시("staples") 하는 것을 OCSP Stapling이라 한다.

맨 처음 봤던 경고의 원인은 결국 OCSP도 [Chain of Trust](https://letsencrypt.org/certificates/)로 구현되어 있기 때문이었다.  로컬에서 OCSP를 구성하여 재현해보자.


## 실습
OCSP를 문제 상황과 똑같이 구현하기 위해 3단계의 Chain of Trust를 만든다. [Chain이 없는 2단계](/posts/self-signed-certificate)와 비슷한 부분은 설명이 생략될 것이다. 또 다른 점은 OpenSSL 설정을 기본 값이 아닌 파일을 직접 만들어 인증서 관련한 키와 옵션을 정확히 알고 사용하게 될 것이다.

실습할 위치에 root, root/cert, intermediate, intermediate/cert 디렉토리를 준비한다. CA와 데이터베이스, OpenSSL 설정 파일 그리고 여러 인증서를 구분하기 위함이다:
```sh
❯ mkdir -p root root/cert intermediate intermediate/cert
❯ tree .
.
├── intermediate
│   ├── certs
└── root
    └── certs

```



### openssl.cnf
OpenSSL 설정 파일은 Root와 Intermediate CA용 두 개이다. 먼저 파일을 올려 놓고 각 섹션별로 필요한곳에서 다시 가져가 설명한다:

- root/openssl.cnf
```
[ ca ]
default_ca = RootCA

[ RootCA ]
dir = root
private_key = $dir/RootCA.key
certificate = $dir/RootCA.crt
new_certs_dir = $dir/certs
database = $dir/index.txt
policy = policy_very_loose
serial = $dir/serial

[ policy_very_loose ]
commonName = supplied

[ req ]
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

- intermediate/openssl.cnf
```
[ ca ]
default_ca = IntermediateCA

[ IntermediateCA ]
dir = intermediate
private_key = $dir/IntermediateCA.key
certificate = $dir/IntermediateCA.crt
new_certs_dir = $dir/certs
database = $dir/index.txt
policy = policy_very_loose
serial = $dir/serial

[ policy_very_loose ]
commonName = supplied

[ req ]
distinguished_name  = req_distinguished_name

[ req_distinguished_name ]
commonName                      = Common Name

[ v3_cert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName = DNS:localhost
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
authorityInfoAccess = OCSP;URI:http://ocsp:2560

[ ocsp ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

### Root CA
Root CA용 키 쌍을 생성한다:
```sh
❯ openssl genrsa -aes256 -out root/RootCA.key 2048
❯ chmod 400 root/RootCA.key
```

Root CA는 chain 할 상위 인증서가 없으므로 CSR이 아니라 곧바로 인증서를 만든다. 그와 관련한 OpenSSL 설정은 다음이다:
```
[ req ]
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca

[ req_distinguished_name ]
commonName                      = Common Name

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

- `req_distinguished_name`: DN(Distinguished Name)을 정의할 때 쓰는 키를 정의한다. 실습에선 간단히 CN(Common Name)만 쓴다.
- `v3_ca`: `openssl-req`에서 Root CA처럼 CSR이 아닌 인증서를 만들 때(옵션 `-x509`) 인증서 필드를 정의한다. 다음 설명하는 필드들은 x509 v3 확장의 명세로써 CA가 CSR로부터 인증서를 발급할 때(`openssl-ca`)도 쓰이는 옵션들이다. [openssl x509v3_config 참고](https://www.openssl.org/docs/manmaster/man5/x509v3_config.html)

위 설정으로 Root CA 인증서를 만든다. 유효기간은 20년으로 한다:
```sh
❯ chmod 400 root/RootCA.key
❯ openssl req -config root/openssl.cnf -key root/RootCA.key -new -x509 -sha256 -extensions v3_ca -out root/RootCA.crt -days 7304
Enter pass phrase for RootCA.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name []:Root CA
```

발급자와 주체를 확인하여 스스로 만든 인증서임을 확인한다:
```sh
❯ openssl x509 -noout -issuer -subject -in root/RootCA.crt
issuer=CN = Root CA
subject=CN = Root CA
```

또 OpenSSL 설정이 잘 적용됐는지 확인해본다(e.g. keyUsage 확장 필드):
```sh
❯ openssl x509 -noout -ext keyUsage -in root/RootCA.crt
X509v3 Key Usage: critical
    Digital Signature, Certificate Sign, CRL Sign
```

### Intermediate CA
Intermediate CA는 Root CA에게 CSR 요청 후 인증서를 발급받는다. Root CA가 인증서 발급을 위해 필요한 옵션은 다음과 같다:
```
[ ca ]
default_ca = RootCA

[ RootCA ]
dir = root
private_key = $dir/RootCA.key
certificate = $dir/RootCA.crt
new_certs_dir = $dir/certs
database = $dir/index.txt
policy = policy_very_loose
serial = $dir/serial

[ policy_very_loose ]
commonName = supplied
```
- `ca`: `openssl-ca`시 사용할 기본 설정으로 아래 `RootCA` 섹션에서 세부 내용을 모두 정의하고 있다.
- `RootCA`: Root CA의 인증서, 키 경로 등을 설정.
  - `new_certs_dir`: 발급하는 인증서 파일을 만든다(`-out` 옵션으로 별도로 인증서 경로를 지정 가능)
  - `database/serial`: 인증서를 관리하는 DB. `database` 파일엔 목록이 쓰이고, `serial`에 쓰인 번호가 순증하며 인증서 발급. 위 `new_certs_dir`엔 `<serial>.pem`(기본 PEM 인코딩)으로 인증서 파일이 쓰인다.
  - `policy`: DN의 필수, 선택 필드를 정의
- `policy_very_loose`: CN만 필수(supplied)로 지정

먼저 DB에 해당하는 파일들을 만들어 준다:
```sh
❯ touch root/index.txt
❯ echo 1000 > root/serial
```

Intermediate CA의 키 쌍과 CSR 생성 후 Root CA에서 인증서를 발급 받는다(유효기간 10년):
```sh
❯ openssl genrsa -aes256 -out intermediate/IntermediateCA.key 2048
❯ chmod 400 intermediate/IntermediateCA.key
❯ openssl req -config root/openssl.cnf -new -sha256 -key intermediate/IntermediateCA.key -out intermediate/IntermediateCA.csr

❯ openssl ca -config root/openssl.cnf -extensions v3_intermediate_ca -days 3652 -notext -md sha256 -in intermediate/IntermediateCA.csr -out intermediate/IntermediateCA.crt
Using configuration from openssl.cnf
Enter pass phrase for RootCA.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'Intermediate CA'
Certificate is to be certified until Apr 23 06:33:28 2032 GMT (3652 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Root CA DB 관련하여 위에서 설명한 내용들을 검증한다:
```
❯ cat root/index.txt
V       320423063328Z           1000    unknown /CN=Intermediate CA

❯ diff root/certs/1000.pem intermediate/IntermediateCA.crt

❯ cat root/serial
1001

```


또 Intermediate CA 인증서도 Root CA것과 비슷하게 설정 값 적용을 확인한다:
```sh
❯ openssl x509 -noout -issuer -subject -in intermediate/IntermediateCA.crt
issuer=CN = Root CA
subject=CN = Intermediate CA

❯ openssl x509 -noout -ext keyUsage -in intermediate/IntermediateCA.crt
X509v3 Key Usage: critical
    Digital Signature, Certificate Sign, CRL Sign
```

마지막으로 Chain of Trust를 확인한다:
```sh
❯ openssl verify -CAfile root/RootCA.crt intermediate/IntermediateCA.crt
IntermediateCA.crt: OK
```


### Server
테스트할 Nginx 서버용 키 쌍, CSR 및 인증서를 만든다. Intermediate CA로부터 발급 받으며 위에서 Root CA의 경우와 OpenSSL 설정 설명이 거의 다 중복되기 때문에 생략한다(유효기간 397일). 서버 인증서용 x509 확장 세션(`v3_ca`)에서 한가지 의미 있는 설정은 `authorityInfoAccess`에 OCSP를 사용한다는 점이다. 이따 OCSP Responder로 할 도메인(ocsp)과 포트(2560)를 써준다:

```
[ v3_cert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
subjectAltName = DNS:localhost
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
authorityInfoAccess = OCSP;URI:http://ocsp:2560
```

CA들과 달리 Nginx 서버용 인증서는 키에 passphrase를 설정하지 않았다:
```sh
# Intermediate CA DB 생성
❯ touch root/index.txt
❯ echo 1000 > root/serial

❯ openssl genrsa -out localhost.key 2048
❯ openssl req -config intermediate/openssl.cnf -new -sha256 -key localhost.key -out localhost.csr
❯ openssl ca -config intermediate/openssl.cnf -extensions v3_cert -days 397 -notext -md sha256 -in localhost.csr -out localhost.crt
```

인증서 포맷이나 체인을 검증하는 과정도 역시 생략한다.

### OCSP
OCSP 역시 Chain of Trust로 동작한다고 설명했다. x509 확장으로 설정한다:
```
[ ocsp ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```
- `extendedKeyUsage`: `OCSPSigning` 용도임을 명시한다.

Intermeidate CA의 인증서를 발급한다. 여기서 **OCSP 인증서의 CN은 요청 URI의 FQDN**을 써주어야 한다. 위 localhost.crt의 authorityInfoAccess 설정과 같게 **`ocsp`**라고 써준다. 인증서 유효기간은 서버 것과 같은 397일로 한다:

```sh
# 키 쌍, CSR, 인증서 생성
❯ openssl genrsa -aes256 -out intermediate/ocsp.key 2048
❯ openssl req -config intermediate/openssl.cnf -new -sha256 -key intermediate/ocsp.key -out intermediate/ocsp.csr
❯ openssl ca -config intermediate/openssl.cnf -extensions ocsp -days 397 -notext -md sha256 -in intermediate/ocsp.csr -out intermediate/ocsp.crt

# Verify
❯ openssl x509 -noout -ext extendedKeyUsage -in ocsp.crt
X509v3 Extended Key Usage: critical
    OCSP Signing

```

[`openssl-ocsp`](https://linux.die.net/man/1/ocsp) 명령으로 테스트용의 간단한 응답 서버(responder)를 만들 수 있다. 포트는 2560로 하였다:
```sh
❯ openssl ocsp -index intermediate/index.txt -CA intermediate/IntermediateCA.crt -rkey intermediate/ocsp.key -rsigner intermediate/ocsp.crt -port 2560
```

같은 명령으로 요청을 보내 테스트 할 수 있다. 이 때 CA는 Chain 하여 사용한다. 응답은 OCSP 부분만 확인한다:
```sh
❯ cat root/RootCA.crt intermediate/IntermediateCA.crt > intermediate/ChainCA.crt

❯ openssl ocsp -CAfile intermediate/ChainCA.crt -url http://127.0.0.1:2560 -resp_text -issuer intermediate/IntermediateCA.crt -cert localhost.crt | head -15
Response verify OK
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: CN = ocsp
    Produced At: Apr 26 11:48:52 2022 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: D19EDD09C8F3985F81A9166EF1F6E96FABF207F1
      Issuer Key Hash: D05FEC1B16A57CFE288306630CF0821B750973FF
      Serial Number: 1002
    Cert Status: good
    This Update: Apr 26 11:48:52 2022 GMT
```

여기선 `-url`에 127.0.0.1을 주었지만, 인증서를 이용할 때는 ocsp라는 도메인 이름을 사용해야 한다. 따라서 로컬 DNS에 추가한다:

```sh
❯ grep ocsp /etc/hosts
127.0.0.1 ocsp

```

그리고 다음 설정으로 로컬에서 Nginx 서버를 실행한다:
```
    server {
        listen       8443 ssl;
        server_name  localhost;

        ssl_certificate      /path/to/localhost.crt;
        ssl_certificate_key  /path/to/localhost.key;

        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /path/to/intermediate/ChainCA.crt;

    }
```

`openssl-s_client`로 요청 시 `-status` 옵션을 주면 응답에 OCSP로 revocation을 체크한 것을 확인할 수 있다:
```sh
❯ openssl s_client -connect localhost:8443 -status 2>&1 < /dev/null | grep -i "ocsp response" -A10
OCSP response:
======================================
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: CN = ocsp
    Produced At: Apr 26 14:53:58 2022 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: D19EDD09C8F3985F81A9166EF1F6E96FABF207F1
      Issuer Key Hash: D05FEC1B16A57CFE288306630CF0821B750973FF
      Serial Number: 1002
    Cert Status: good
```

### 문제 재현
맨 처음의 오류는 OCSP의 Chain of Trust를 끊으면 발생한다. Nginx 설정 중 다음 설정을 코멘트 처리 해보자:
```
        # ssl_trusted_certificate /path/to/intermediate/ChainCA.crt;
```

```sh
❯ sudo nginx -t
nginx: [warn] "ssl_stapling" ignored, issuer certificate not found for certificate "/path/to/localhost.crt"
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
```

앞서 만들어 보았듯, OCSP도 `rsinger/rkey` 옵션에 Intermediate CA가 발급한 인증서를 사용하고 인증서 체인이 되지 않았을 때 이 같은 오류가 발생한다. 이대로 Nginx를 실행하면 OCSP 응답을 받을 순 없으나 서버 자체는 HTTPS로 서빙하는데 문젠 없었다:
```sh
❯ openssl s_client -CAfile root/RootCA.crt -connect localhost:8443 -status 2>&1 < /dev/null | grep -i "ocsp response"
OCSP response: no response sent

```

다만 실제로 겪은 경우는 이 로컬 재현과 조금 달랐다. 실제 서버에선 Nginx 디렉티브 `ssl_trusted_certificate`을 사용하지 않고 있다. 실수는 교체하는 인증서에서 Intermediate CA를 누락했기 때문이다.

DigiCert에서 [Nginx용 인증서는, 다음처럼 서버용 인증서와 Intermediate CA 인증서를 이어 붙여 보내 주는데](https://stackoverflow.com/questions/25750890/nginx-install-intermediate-certificate) 아래 것은 빼고 인증서를 교체하여 발생했다:
```sh
❯ cat localhost.crt intermediate/IntermediateCA.crt > localhost.nginx.crt
```
```
-----BEGIN CERTIFICATE-----
(localhost 인증서)
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
(Intermediate CA 인증서 - 누락한 부분)
-----END CERTIFICATE-----
```

따라서 이 문제는 **OCSP의 인증서 체인을 검증할 수 없었기 때문에 발생한 문제**였다. 다만 경고(warn)로 나오듯, HTTPS의 TLS에는 영향을 주지 않는 것으로 보였다.

실습하며 만든 인증서나 설정 등의 파일은 [이 커밋](https://github.com/flavono123/t/commit/d7e1f8d8fe0374bb82f7c6ad64eb74a00cae1a0a)과 같다.

## 정리
- OCSP는 인증서의 폐기 여부를 확인하는 방법이다.
  - OCSP Responder 역시 인증서를 사용하고 인증서 체인(Chain of Trust)을 따른다.
  - Intermediate CA에서 발급한 인증서를 사용한다.
- Nginx 인증서는 서버(Subject)와 Intermediate CA(Issuer)의 두 개의 인증서를 붙인(concat) 형태로 사용한다.


---

## 참고
- https://www.keyfactor.com/blog/what-is-a-certificate-revocation-list-crl-vs-ocsp/
- https://www.openssl.org/docs/
- https://superuser.com/questions/1717279/no-crl-in-certificates-nowadays/1717305?noredirect=1#comment2649722_1717305
- https://jamielinux.com/docs/openssl-certificate-authority/
- https://stackoverflow.com/questions/25750890/nginx-install-intermediate-certificate
