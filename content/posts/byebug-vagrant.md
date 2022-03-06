---
title: "Vagrant byebug로 디버깅하기"
date: 2022-03-07T00:38:49+09:00
tags:
- vagrant
- ruby
---

Vagrant를 쓰던 중 byebug로 breakpoint를 잡고 디버깅 해보고 싶어졌다. [이 글](https://nts.strzibny.name/debugging-vagrant-with-pry-byebug/)을 참고했으나, 난 pry, pry-byebug가 아닌 [byebug](https://github.com/deivid-rodriguez/byebug)를 사용했다(과거엔 저렇게 썼다고 들었다). 또 글과 달리, (이젠) 맥에서 brew로 설치한 vagrant는 소스 코드를 받아 실행하지 않는걸로 보인다.

먼저 원래 쓰고 있던 brew로 설치한 vagrant를 지운다:
```sh
❯ brew uninstall vagrant
```

[문서](https://www.vagrantup.com/docs/installation/source)대로 vagrant를 코드를 받아 설치한다:
```sh
❯ git clone https://github.com/hashicorp/vagrant.git
❯ cd vagrant
❯ bundle install
❯ bundle --binstubs exec # Deprecation warning
❯ cd /usr/local/bin/
❯ ln -sf /path/to/vagrant/exec/vagrant
```

`bundle --binstubs exec`을 실행하면 deprecation warning이 나올 것이다. 하지만 이대로 실행해야 한다. 최신 bundler엔 [`bundle-binstubs`](https://bundler.io/man/bundle-binstubs.1.html)라는 서브 명령으로 대체됐지만 동작이 아예 다르다([1버전 bundler를 보면 `--binstubs` 옵션 설명을 볼 수 있다](https://bundler.io/v1.0/man/bundle-install.1.html)). 레일즈 개발하면서 한번도 사용해보지 않은 명령이다.

이제 vagrant gemspec에 byebug를 추가해준다:
```ruby
# /path/to/vagrant/vagrant.gemspec
...
# Monkey patch
s.add_dependency "byebug"
...
```

레일즈 개발할 때 Gemfile에 직접 add해서 역시 처음 써보는 방법이다.

이제 Vagranfile에서 byebug를 찍으면 디버깅이 가능하다:
```ruby
Vagrant.configure("2") do |config|
...
  require 'byebug'; byebug
...
end
```

`vagrant up` 같은 명령을 실행하면 배포판이 아닌걸 쓴다고 주의를 준다:
```sh
You appear to be running Vagrant outside of the official installers.                                     
Note that the installers are what ensure that Vagrant has all required                                   
dependencies, and Vagrant assumes that these dependencies exist. By                                      
running outside of the installer environment, Vagrant may not function                                   
properly. To remove this warning, install Vagrant using one of the                                       
official packages from vagrantup.com.
```

## 문제 해결?
원하는대로 Vagrantfile 내에서 breakpoint는 잘 잡힌다. 하지만 이걸로 문제를 해결한건 아니다:

```ruby
Vagrant.configure("2") do |config|
  # VM1
  config.vm.define "audrey" do |audrey|
    ...
  end

  # VM2
  config.vm.define "dismas" do |dismas|
    ...
  end

  # VM3
  config.vm.define "valdwin" do |valdwin|
    ...
  end

  # Ansible
  config.vm.provision "ansible" do |ansible|
    ...
  end
end
```
내가 문제(?)라고 생각한 부분은 위 같은 Vagrantfile에서 provisioner, 즉 ansible 동작 문제였다. 내가 기대한건 vm 세팅이 다 끝나고 ansible playbook이 전체 vm에 대해 한번 도는거였다. 하지만 vm setup과 동시에 provision도 각각 되었다.

먼저 모든 vm에 대해 provision 하는 것은 limit 옵션을 all을 주어 해결할 수 있었다:

```ruby
Vagrant.configure("2") do |config|
  # VM1
  ...

  # VM2
  ...

  # VM3
  ...

  # Ansible
  config.vm.provision "ansible" do |ansible|
    ...
    ansible.limit = "all"
    ...
  end
end
```

하지만 이렇게 하니 모든 vm에 대해 각 vm up할 때마다 실행했다. ansible이 얼마나 멱등성 있는지 테스트하기에 좋겠다만... 만약 최초 up이라면 vm1에서 실행할 땐 vm2,3 모두 unreachable이라 실패할 것이다.

[이러한 트릭](https://www.vagrantup.com/docs/provisioning/ansible#tips-and-tricks)을 쓰는데, 난 위처럼 vm 이름을 인덱스로 나열한게 아니라서 원치 않는 방법이었다(코드의 vm 이름은 예시이다).


그래서 ansible block에 byebug를 걸고 실행해봤지만:
```ruby
Vagrant.configure("2") do |config|
  # Ansible
  config.vm.provision "ansible" do |ansible|
    ...
    require 'byebug'; byebug
    ...
  end
end
```

vagrant는 Vagrantfile의 DSL(?)을 먼저 쭉 평가한 후 실행된다. 아마 순회 가능한 객체로 담은 후에 풀어서 순서대로 하는 것 같다. 이때 provisioner가 vm마다 들어가게 될텐데... vagrant의 코드가 엄청 많았다. [vm.rb](https://github1s.com/hashicorp/vagrant/blob/HEAD/plugins/kernel_v2/config/vm.rb) 하나 보다가 포기했다.

그래도 배운 점은 있다. 어떤 언어 쓸 수 있다고 디버깅과 코딩까진 그리 쉽지 않다는 것. 요즘 nginx나 커널 코드 보다가 막히면 'C를 다시 공부해?' 이런 생각을 많이 했다("실제로" 한 생각이다).. 그리고 이렇게 오만한 생각을 했던 것도 좋은 판단을 하는데 도움을 준거 같다. 

사실 vagrant 코드도 시간과 관심이 있으면 들여다 보는 것은 어렵진 않겠지만, **효율성**의 문제다. 나는 빠르게 쉽게 타협하는 법을 알고 있다:

```ruby
Vagrant.configure("2") do |config|
  # VM1
  config.vm.define "audrey" do |audrey|
    ...
  end

  # VM2
  config.vm.define "dismas" do |dismas|
    ...
  end

  # VM3
  config.vm.define "valdwin" do |valdwin|
    ...
    # Ansible
    # provisioning "after" the valdwin setup
    valdwin.vm.provision "ansible" do |ansible|
      ...
      ansible.limit = "all"
      ...
    end
  end
end
```

내 눈엔 valdwin만 provisioning 하는것 같아서 이 방법을 안쓰고 싶었는데.. 나중에 정말~ 못 참겠다 싶으면, 그리고 맨 처음의 Vagrantfile처럼 쓰는 것이 더 타당하다는 근거와 vagrant 코드에 기여하고 싶은 욕심이 생기면 그 때 vagrant가 어떻게 루비로 vm을 올리고 provision 하는지 살펴봐야겠다.

---

## 참고
- https://nts.strzibny.name/debugging-vagrant-with-pry-byebug/
- https://www.vagrantup.com/docs/installation/source
- https://www.vagrantup.com/docs/provisioning/ansible#tips-and-tricks
