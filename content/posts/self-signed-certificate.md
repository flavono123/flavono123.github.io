---
title: "Self-signed Certificate(NGINX)"
date: 2022-02-05T13:25:51+09:00
tags:
- websecure
- nginx
---

### 0. 준비
- openssl3
- kubernetes(minikube)

### 1. CA RSA 키 페어 생성
실제 CA가 아니라 우리가 직접 CA를 만들어 TLS 인증하는 과정을 모의로 해본다(self-signed certificate)
```sh
$ openssl genrsa -aes256 -out rootCA.key 2048
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:


$ ll rootCA.key 
-rw-------  1 hansuk  staff  1874 Jan 14 14:30 rootCA.key
```
- 비밀키 분실을 대비하여 AES256으로 암호화 한다. 이때 passphrase를 잘 기억하고 발급할 때 사용하자

### 2. CA 인증서 발급
```sh
$ openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.crt
Enter pass phrase for rootCA.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:KR
State or Province Name (full name) [Some-State]:RootState
Locality Name (eg, city) []:RootCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Root Inc.
Organizational Unit Name (eg, section) []:Root CA
Common Name (e.g. server FQDN or YOUR name) []:Self-signed Root CA
Email Address []:

$ ll
total 16
-rw-r--r--  1 hansuk  staff  1033 Jan 14 14:45 rootCA.crt
-rw-------  1 hansuk  staff  1874 Jan 14 14:30 rootCA.key
```

상호작용하는 입력은 대충 해주어도 된다. 키 외에 crt 파일이 생긴걸 확인.

### 3. localhost(subject) CSR 생성
```sh
$ cat << EOF > localhost.csr.cnf
> [req]
> default_bits = 2048
> prompt = no
> default_md = sha256
> distinguished_name = dn
> 
> [dn]
> C=KR
> ST=Seoul
> L=Seocho-gu
> O=Localhost
> CN = localhost
> EOF

$ openssl req -new -sha256 -nodes -out localhost.csr -newkey rsa:2048 -keyout localhost.key -config localhost.csr.cnf
...+......+.....+...+.+.........+............+...+..+......+.......+...+...+..+....+.....+.............+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*............+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+..+.+...+......+.....+....+.....+...............+....+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+............+..+......+.+...+........+...+....+......+..+...+.........+..................+....+...+...+..+...+...+....+......+............+..+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+.+...........+.......+.....+...+....+...+......+..+...+......+......+..................+.+......+.....+....+...+...+.........+....................+.+..+...+...+.......+...+............+..+...............+.+.........+...+........+.+.....+....+............+...+...........+...+......+....+........+...+......+.+...+.........+........+....+...+......+..+.............+......+........+.+..+...+......+...+..........+..+.............+......+.........+......+..+...+.+..............+.........+......+.+.....+...+...+.......+...+.....+..........+...+.........+...+..............+...+................+..+.........+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----

$ ll localhost.*
-rw-r--r--  1 hansuk  staff   985 Jan 14 14:53 localhost.csr
-rw-r--r--  1 hansuk  staff   141 Jan 14 14:52 localhost.csr.cnf
-rw-------  1 hansuk  staff  1704 Jan 14 14:53 localhost.key
```
위에서 CA CSR 발급 시 상호작용으로 입력했던 common name과 같은 입력 값을 파일에 쓰고 인자로 준다.

### 4. localhost 인증서 생성

```sh
$ cat << EOF > v3.ext 
> authorityKeyIdentifier=keyid,issuer
> basicConstraints=CA:FALSE
> keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
> subjectAltName = @alt_names
> 
> [alt_names]
> DNS.1 = localhost
> EOF

$ openssl x509 -req -in localhost.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out localhost.pem -days 365 -sha256 -extfile v3.ext 
Enter pass phrase for rootCA.key:

$ openssl verify -CAfile rootCA.crt localhost.pem 
localhost.pem: OK
```
X509 v3 확장 명세를 정의하고 인증서를 발급한다. `alt_names` (Subject Alternative Name, SAN)필드에 `DNS.n`를 추가하면 인증서로 신뢰할 수 있는 도메인을 추가할 수 있다.

### 5. subject 서버 준비

로컬 nginx에 인증서를 설치하여 인증을 테스트해본다. 먼저 인증서를 설치를 위한 시크릿과 설정맵도 만든다. 쿠버네티스는 [TLS 타입에 대해 빌트인 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/#tls-%EC%8B%9C%ED%81%AC%EB%A6%BF)이 정의되어 있다:
```sh
$ kubectl create secret tls nginxsecret --key localhost.key --cert localhost.pem
secret/nginxsecret created

$ cat << EOF > default.conf
> server {
>         listen 80 default_server;
>         listen [::]:80 default_server ipv6only=on;
> 
>         listen 443 ssl;
> 
>         root /usr/share/nginx/html;
>         index index.html;
> 
>         server_name localhost;
>         ssl_certificate /etc/nginx/ssl/tls.crt;
>         ssl_certificate_key /etc/nginx/ssl/tls.key;
> 
>         location / {
>                 try_files $uri $uri/ =404;
>         }
> }
> EOF

$ kubectl create configmap nginxconfmap --from-file=default.conf
configmap/nginxconfmap created
```

만든 인증서를 설치할 nginx 컨테이너를 up 한다:
```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-cert
  labels:
    app: nginx-cert
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
    nodePort: 30080
  - port: 443
    protocol: TCP
    name: https
    nodePort: 30443
  selector:
    app: nginx-cert
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx-cert
  name: nginx-cert
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cert
  template:
    metadata:
      labels:
        app: nginx-cert
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: nginxsecret
      - name: configmap-volume
        configMap:
          name: nginxconfmap
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        - containerPort: 443
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
        - mountPath: /etc/nginx/conf.d
          name: configmap-volume
```

### 6. SSL 요청
minikube 서비스 IP를 알아내어 openssl로 먼저 요청해본다:
```sh
$ minikube service --url nginx-cert -n cert
http://192.168.64.2:30080
http://192.168.64.2:30443


$ openssl s_client -CAfile rootCA.crt -connect 192.168.64.2:30443 2>/dev/null < /dev/null
CONNECTED(00000003)
---
Certificate chain
 0 s:C = KR, ST = Seoul, L = Seocho-gu, O = Localhost, CN = localhost
   i:C = KR, ST = RootState, L = RootCity, O = Root Inc., OU = Root CA, CN = Self-signed Root CA
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 17 03:37:15 2022 GMT; NotAfter: Jan 17 03:37:15 2023 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIDzzCCAregAwIBAgIUYMe6nRgsZwq9UPMKFgj9dt9z9FIwDQYJKoZIhvcNAQEL
BQAweDELMAkGA1UEBhMCS1IxEjAQBgNVBAgMCVJvb3RTdGF0ZTERMA8GA1UEBwwI
Um9vdENpdHkxEjAQBgNVBAoMCVJvb3QgSW5jLjEQMA4GA1UECwwHUm9vdCBDQTEc
MBoGA1UEAwwTU2VsZi1zaWduZWQgUm9vdCBDQTAeFw0yMjAxMTcwMzM3MTVaFw0y
MzAxMTcwMzM3MTVaMFkxCzAJBgNVBAYTAktSMQ4wDAYDVQQIDAVTZW91bDESMBAG
A1UEBwwJU2VvY2hvLWd1MRIwEAYDVQQKDAlMb2NhbGhvc3QxEjAQBgNVBAMMCWxv
Y2FsaG9zdDCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALc9retjBorw
RKbuyC1SNx1U9L5LJPPbBkBh4kg98saQxtRX0Wqs5mgswWMZYL3E6yRl0gfwBkdq
t8GVQ49dgg0QO5MbG9ylfCLS9xR3WWjAgxaDJ0W96PyvTzmg295aqqHFKPSaG/nM
JyZgFJDuGoRRgwoWNqZ1pRCDLMIENDx4qgjOnQch529pM9ZRwFQSswKpn4BVkY00
/u8jIvax67kFOg70QGY16paGEg7YfSNle7BFZY0VJ8rIiBoqwRmPH6hbF/djxe5b
yzkI9eqts9bqw8eDLC28S36x62FxkdqkK8pI/rzWAKSV43TWML1zq4vM2bI+vp0k
a06GhSsS1bUCAwEAAaNwMG4wHwYDVR0jBBgwFoAUURHNpOE9zTgXgVYAGvLt94Ym
P+8wCQYDVR0TBAIwADALBgNVHQ8EBAMCBPAwFAYDVR0RBA0wC4IJbG9jYWxob3N0
MB0GA1UdDgQWBBSS1ZHHT6OHTomYIRsmhz6hMJLGnDANBgkqhkiG9w0BAQsFAAOC
AQEAWA23pCdAXtAbdSRy/p8XURCjUDdhkp3MYA+1gIDeGAQBKNipU/KEo5wO+aVk
AG6FryPZLOiwiP8nYAebUxOAqKG3fNbgT9t95BEGCip7Cxjp96KNYt73Kl/OTPjJ
KZUkHQ7MXN4vc5gmca8q+OqwCCQ/daMkzLabPQWNk3R/Hzo/mT42v8ht9/nVh1Ml
u3Dow5QPp8LESrJABLIRyRs0+Tfp+WodgekgDX5hnkkSk77+oXB49r2tZUeG/CVv
Fg8PuUNi+DWpdxX8fE/gIbSzSsamOf29+0sCIoJEPvk7lEVLt9ca0SoJ7rKn/ai4
HxwTiYo9pNcoLwhH3xdXjvbuGA==
-----END CERTIFICATE-----
subject=C = KR, ST = Seoul, L = Seocho-gu, O = Localhost, CN = localhost
issuer=C = KR, ST = RootState, L = RootCity, O = Root Inc., OU = Root CA, CN = Self-signed Root CA
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 1620 bytes and written 390 bytes
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
    Session-ID: 00C994177DCF3DA424433CD0EDB33FF54C1198D0DE2C08D61BBD155D85890F20
    Session-ID-ctx: 
    Master-Key: B88A94A3556F9A595B3B395E0F4701290D3837BC49CA125CA25E0351261E7C59F0DF0928B4A12E33509B38B2F2684711
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - 5d 96 c8 cf 8f 9c 9f c4-15 25 b9 57 56 13 ec 9d   ]........%.WV...
    0010 - 2a 26 f8 e5 09 e4 01 11-fa 59 25 a1 6c e7 ba 94   *&.......Y%.l...
    0020 - 85 b2 6c 49 b1 1f 78 6c-4c b1 ed b1 af 1b ce 46   ..lI..xlL......F
    0030 - c6 f1 bd f8 36 da b0 c4-31 ce ba 62 72 4b 76 a1   ....6...1..brKv.
    0040 - 2d d4 b6 8d cb c3 39 ee-8f 05 77 48 9e 61 3e 4a   -.....9...wH.a>J
    0050 - 2c 6b f4 19 59 f4 93 cf-bd d7 16 76 ce af 48 b5   ,k..Y......v..H.
    0060 - b7 cc c2 aa 9b 36 44 b5-93 4e 1e 8a 96 6c e7 96   .....6D..N...l..
    0070 - b5 0e 04 96 77 d7 df 2a-ef 70 ad c1 7b 82 a9 3a   ....w..*.p..{..:
    0080 - 5c 9b 4a 94 fb ea dc de-15 11 fa 3b e1 af dd cf   \.J........;....
    0090 - 57 48 98 ed c8 d7 6b 5d-00 c3 84 5f 86 0c a8 b5   WH....k]..._....
    00a0 - 18 66 52 14 c4 9c b0 ff-0b 39 12 b6 0a 78 49 4d   .fR......9...xIM

    Start Time: 1644478485
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: yes
---
```

브라우저에서도 확인해본다. openssl 요청 시 CAfile 옵션으로 CA 인증서를 넘겨 주었지만, 브라우저 요청에 적용하기 위해선 맥 시스템 키체인에 추가해야한다. 신뢰 탭을 펼쳐 이 인증서 사용 시 **항상 신뢰**로 변경하자:
![1](/images/self-signed-certificate/1.gif)

![2](/images/self-signed-certificate/2.png)


- [쿠버네티스 공식 문서에도 self-signed certificate를 설명하는 글](https://kubernetes.io/ko/docs/tasks/administer-cluster/certificates/)이 있다. CKA를 준비하며 알게 된것인데, 쿠버네티스는 클러스터 사용자 인증에 TLS 인증을 사용한다. 인증서 요청 승인, 발급 등의 과정은 kubectl로 더 간단하게 할 수 있지만 CSR 생성은 위와 같은 방법으로도 하고, openssl이 아닌 다른 도구를 쓰기도 한다.
  - 아무튼 나는 openssl로 self-signed certificate 해본 것이 CKA 시험 보는데에도 큰 도움이 됐다.
  - openssl 뿐만 아니라 인증서 발급할 수 있는 다른 여러 도구가 있다. 하지만 openssl로 연습해보길 잘했다. 가장 기초가 되는 도구 같다.
- nginx 설정하다가 오타가 생겨 헤맸는데 스택오버플로에서 도움을 받아 해결했다: https://stackoverflow.com/questions/70830744/nginx-pod-responds-with-its-listening-port-in-self-signed-certificate-examples

### 참고
- [https의 원리, 그리고 Self-signed SSL 까지](https://baek9.github.io/security/2019/04/10/https%EC%9D%98_%EC%9B%90%EB%A6%AC,_%EA%B7%B8%EB%A6%AC%EA%B3%A0_Self-signed-SSL_%EA%B9%8C%EC%A7%80.html)
- [OpenSSL 로 ROOT CA 생성 및 SSL 인증서 발급](https://www.lesstif.com/system-admin/openssl-root-ca-ssl-6979614.html)
- [서비스와 애플리케이션 연결하기](https://kubernetes.io/ko/docs/concepts/services-networking/connect-applications-service/)
