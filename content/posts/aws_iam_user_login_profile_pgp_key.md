---
title: "Terraform으로 AWS 로그인 자격 증명 생성 시 PGP 키 사용"
date: 2022-03-01T16:51:33+09:00
tags:
- aws
- terraform
---

회사 AWS IAM를 테라포밍하고 있다. 계기가 되는 사건이 하나 있는데, 내가 devops니까, 다른 팀에서 AWS에 특정 권한이 있는 계정을 생성해달라는 요구를 받았다. 난 우리가 AWS를 거의 안 쓰고 내가 로그인하는 루트 계정도 공용 계정이라 따로 IAM 관리를 아예 안하는 줄 알았다. 하지만 그렇지 않았다... 적어도 코드로 리뷰하고 공유하면 이런 지식이 단절 되는 일은 없을거 같아 (방치해둔) terraform을 쓰자고 했다.

그렇게 최소한의 권한만 허가하며 terraform으로 만들어 가던 중, 요청한 팀에서 콘솔 로그인도 필요하다고 했다. 그 과정에서 알게 된 `aws_iam_user_login_profile` 리소스 최초 생성 시 필요한 `pgp_key` 인자 설정하는 방법을 정리한다.

## PGP, GPG
`aws_iam_user_login_profile`의 `pgp_key`에 대해 설명하기 전에 몇 가지 배경 지식부터 정리한다. PGP(Pretty Good Privacy)는 공개키 암호화 방식이다. 자세한 구현(역사적이 이슈도 있는거 같다)을 살펴보진 않았고, 공개키를 공유해서 암호화하고 복호화는 개인키로만 할 수 있다는, 큰 맥락에 대해서만 이해했다([링크](https://academy.binance.com/ko/articles/what-is-pgp)로 설명을 대체한다).

GPG(GNU Privacy Guard)에서 PGP를 사용할 수 있다. 맥에서 `gpg` CLI 프로그램을 brew로 받을 수 있다(`brew install gpg`). gpg를 써서 키 쌍을 만들면, 공개키를 export하지 않았을 때, 개인키는 로컬에만 있게 된다($HOME/.gnupg/pubring.kbx). terraform은 더 간편한 방법, 로그인을 통해서 개인키에 안전하게 접근할 수 있도록, `keybase`라는 프로그램을 지원한다(`gpg` 동작을 실습해보고 싶으면 [링크](https://johngrib.github.io/wiki/gpg/)를 따라해보는걸 추천, 이 글에선 `keybase`를 사용해서 키를 만들고 결과를 `gpg`로 확인하는 정도로만 사용한다).

## Keybase
`keybase`는 pgp키를 생성하고, 비밀키를 암호화해서 서버에 올려 두어 접근할 수 있게 한다. https://keybase.io/ 에서 가입 하고 로컬에 CLI 프로그램을 받은 후 로그인한다:
```sh
❯ keybase login
Your keybase username: flavono123


************************************************************
* Name your new device!                                    *
************************************************************



Enter a public name for this device: hansukhongmacbookpro



✔ Success! You provisioned your device hansukhongmacbookpro.

You are logged in as flavono123
  - type `keybase help` for more info.

```
로그인 하는 노트북(device)의 정보도 같이 등록한다. 다음 명령으로 pgp 키 페어를 만든다:
```sh
❯ keybase pgp gen
Enter your real name, which will be publicly visible in your new key: flavono123
Enter a public email address for your key: flavono123@gmail.com
Enter another email address (or <enter> when done): 
Push an encrypted copy of your new secret key to the Keybase.io server? [Y/n] 
When exporting to the GnuPG keychain, encrypt private keys with a passphrase? [Y/n] 
▶ INFO PGP User ID: flavono123 <flavono123@gmail.com> [primary]
▶ INFO Generating primary key (4096 bits)
▶ INFO Generating encryption subkey (4096 bits)
▶ INFO Generated new PGP key:
▶ INFO   user: flavono123 <flavono123@gmail.com>
▶ INFO   4096-bit RSA key, ID F5E1C12BAD8A84B7, created 2022-03-01
▶ INFO Exported new key to the local GPG keychain
```

두번째로 물어보는 이메일(*Enter another email address (or \<enter\> when done)*) 질문은 엔터만 쳐서 건너 뛰었다. primary 키([링크](https://academy.binance.com/ko/articles/what-is-pgp)에서 설명한 세션 키, 데이터 서명에 사용)와 암호화 키 모두 키쌍으로 만든다. 이 명령은 `gpg --gen-key`와 거의 유사하고 `gpg --full-generate-key` 명령을 사용하면, 키들의 길이, 알고리즘, 유효기간 등을 세세하게 설정할 수 있다.

keybase에서 추가로 물어보는 내용은:
1. 비밀키를 서버에 암호화하여 저장할 것인지
2. 비밀키의 passphrase도 암호화하여 저장할 것인지

이며 기본으로 yes이다. 기본 값으로 진행하면 된다. 그게 keybase를 쓰는 이유, 편하게 pgp 키를 사용, 이니까.

생성한 pgp 키 목록을 확인할 수 있다. 이것은 gpg를 통해서도 가능하다:
```sh
❯ keybase pgp list
Keybase Key ID:  010153589a269395b8ad158602759357ceee58b2a50fdacce395ee4cd932737361e00a
PGP Fingerprint: 92a9310cd875567ab7ba0e12f5e1c12bad8a84b7
PGP Identities:
   flavono123 <flavono123@gmail.com>

❯ gpg --list-keys # 또는 gpg -k
gpg: checking the trustdb
gpg: no ultimately trusted keys found
/Users/hansuk/.gnupg/pubring.kbx
--------------------------------
pub   rsa4096 2022-03-01 [SC] [expires: 2038-02-25]
      92A9310CD875567AB7BA0E12F5E1C12BAD8A84B7
uid           [ unknown] flavono123 <flavono123@gmail.com>
sub   rsa4096 2022-03-01 [E] [expires: 2038-02-25]

❯ gpg --list-secret-keys # 또는 gpg -K
/Users/hansuk/.gnupg/pubring.kbx
--------------------------------
sec   rsa4096 2022-03-01 [SC] [expires: 2038-02-25]
      92A9310CD875567AB7BA0E12F5E1C12BAD8A84B7
uid           [ unknown] flavono123 <flavono123@gmail.com>
ssb   rsa4096 2022-03-01 [E] [expires: 2038-02-25]
```

keybase는 핑거프린트와 keybase의 아이디정도로 실제 pgp 키에 관한 정보는 볼 수 없지만, gpg 명령으로 공개키와 개인키 각(primary, sub) 쌍이 있는걸 눈으로 확인할 수 있다. 또 keybase 생성 시 설정하지 않았던 만료 기간도 보인다.

## aws_iam_user_login_profile
![1](/images/aws_iam_user_login_profile_pgp_key/1.png)
[`aws_iam_user_login_profile` 리소스](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user_login_profile)는 위 그림의 AWS IAM 콘솔에서 '사용자 > 보안 자격 증명 > 로그인 자격 증명'에 해당하는 리소스이다. 기본적으론 비활성 되어 있고, 리소스를 생성해주어야 활성화 된다.

활성화하면 AWS에서 자동으로 비밀번호를 만들어 준다. 이 때 이 **비밀번호를 안전하게 받아오기 위해 PGP 키를 사용한다.** 따라서 `pgp_key` 인자는 optional이지만 최초 apply 하려면 꼭 필요하다.

PGP 키가 있는 keybase 계정에 로그인이 되어 있는 터미널에서 `terraform apply`를 실행할 경우, `pgp_key` 인자에 `"keybase:<username>"`을 써주면 된다. 앞의 keybase는 literally "keybase"이다. 뒤 텍스트와 차별이 없어서 엄청 해맸다 -_-;:

```ruby
resource "aws_iam_user_login_profile" "user" {
  user = aws_iam_user.user.name
  pgp_key = "keybase:keybase_username"
}
```

만약 PGP 공개키 자체를 넘겨주고 싶다면 base64로 디코딩한 파일을 인자로 써줄 수도 있다(`filebase64("path/to/pubkey")`). 이 방법을 쓰면 공개키를 추출(export)하고 terraform 레포에도 공개키를 저장하고 있어야 해서, keybase 사용보다 더 복잡한 방법 같다:
```sh
❯ keybase pgp export -o path/to/pubkey
# 또는
❯ gpg --armor --export <keyring> > path/to/pubkey
```

암호화된 출력 비밀번호를 복호화 할 때도 차이가 있다. keybase는 passphrase 입력 인터랙티브가 앱으로 뜨지만, gpg를 쓰면 `export GPG_TTY=$(tty)` 명령하여 터미널에서 입력 받을 수 있도록 열어 주어야 한다. 또 terraform 명령을 공용으로 쓴다면 이 keybase를 사용해 PGP키 공유하는게 더 편할거 같다(우린 각자 로컬에서 해서 잘 모르겠다..):

```ruby
output "password" {
  value = aws_iam_user_login_profile.user.encrypted_password
}
```

```sh
❯ terraform output -raw password | base64 --decode | keybase pgp decrypt
# 또는
❯ export GPG_TTY=$(tty)
❯ terraform output -raw password | base64 --decode | gpg -d
```

결국은 회사에서 keybase도 사용 안하고 aws_iam_user_login_profile 리소스도 terraform으로 관리 안하기로 했다. 다른 서드파티 앱을 쓰는 비용 대비 IAM 작업할 빈도가 적어 1password로 비밀번호 생성해 수작업 하기로 했다. 그냥 나의 PGP 만드는 계기가 되었다 ㅎㅎ.

## 정리
- `aws_iam_user_login_profile` 생성 시 `pgp_key` 옵션을 써주어야 한다.
- `keybase`를 사용하면 로그인으로 편하게 할 수 있다.
  - export 한 공개키를 직접 쓸 수도 있다.

---

## 참고
- https://academy.binance.com/ko/articles/what-is-pgp
- https://johngrib.github.io/wiki/gpg/
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user_login_profile
- https://stackoverflow.com/questions/53513795/pgp-key-in-terraform-for-aws-iam-user-login-profile
