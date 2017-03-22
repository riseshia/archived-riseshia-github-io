---
layout: post
title: "capistrano3-puma 버전업에 따른 버그 해결하기"
date: 2017-03-22 12:11:31 +0900
categories:
---

근 2주 사이에 `capistrano3-puma` 버전이 `v1.2.1`에서 `v3.0.2`로 올라가는
대격변이 있었습니다. 이 때문에 있었던 ~~왠지 저만 빠진듯한~~ 버그를 해결하고
겸사겸사 잡지식도 설명합니다.

## 버그 간단 소개

`puma` 관련 태스크를 실행하려고 하면 다음과 같이 `bundle` 명령어를 찾을 수
없다는 메시지가 출력됩니다.

```bash
00:31 puma:restart
      01 bundle exec pumactl -S /destination/app/shared/tmp/pids/puma.state -F /destination/app/shared/puma.rb restart
      01 bash: bundle: command not found
bundle exec pumactl -S /destination/app/shared/tmp/pids/puma.state -F /destination/app/shared/puma.rb restart
```

심플하네요. 디버깅 과정은 생략하고 일단 상황을 재현해보죠.

## 상황 재현하기

- `capistrano3-puma` >= 2.x
- `capistrano-rbenv` # 루비 버전 관리 하는 젬이라면 사실 뭐든 ok. 설명 편의를 위해 예제는 rbenv로 가정하고 진행합니다.

```ruby
# deploy.rb
set :rbenv_map_bins, %w{rake gem bundle ruby rails}
# rvm이라면 `:rvm_map_bins`, chruby라면 `:chruby_map_bins`겠죠.
```

단지 이것만으로 당신의 배포 프로세스가 죽는건 한순간!

## 원인

`capistrano-bundler`와 `capistrano-rbenv`는 조건에 맞는 명령에 한정해서 이를
감싸주게 되는데요. 로그를 잘 살펴보면 rbenv의 래핑이 동작하지 않고 있음을
확인할 수 있습니다.

### `:{rvm, chruby, rbenv}_map_bins`, `:bundle_bins`

여기서 명령을 감쌀지 아닐지를 결정하는 것이 이 변수들입니다. 이 변수 배열에
들어있는 명령어인 경우 `RBENV_ROOT=#{fetch(:rbenv_path)} RBENV_VERSION=#{fetch(:rbenv_ruby)} #{fetch(:rbenv_path)}/bin/rbenv exec`를
명령어의 앞에 추가해줍니다(기본값이며 변경 가능).
돌아와서 이 값에 기본으로 들어있는 명령어들은
`%w{rake gem bundle ruby rails}`입니다.

```patch
# Ref: https://github.com/seuros/capistrano-puma/commit/c6e34a45d917d76dea941707f4ed6e60d8442213#diff-56c283d8f205519259d1321eb2dcf468R26

+ set :bundle_bins, fetch(:bundle_bins).to_a.concat(%w{ puma pumactl })

...

- execute :bundle, 'exec', :pumactl, "-S #{fetch(:puma_state)} #{command}"
+ execute :pumactl, "-S #{fetch(:puma_state)} #{command}"
```

위는 문제를 발생시키는 커밋의 일부입니다. 잘 살펴보면
`bundle_bins`에 `puma`, `pumactl`을 추가하고 덮어쓰고 있는 것을 확인할 수
있습니다. 그런데 왜 안될까요?

### 변수 초기화 순서

답은 실행 순서에 있습니다.

- `Capfile`의 `require 'capistrano-rbenv'`를 읽어서 해당 변수를 설정합니다.
- `Capfile`의 `require 'capistrano3-puma'`를 읽어서 해당 변수에 `puma`, `pumactl`을 추가합니다.
- `deploy.rb`를 불러옵니다. 이때 `set`으로 **덮어씁니다**.

여기서 알 수 있는 사실은 두 가지가 있겠네요.

- (당연하지만) `Capfile` 내부의 `require`는 순서 의존성이 존재한다.
- (당연하지만) `set`이 두번 호출되는 경우에 똑똑하게 뭔가 처리해주지 않는다.

### 지금까진 왜 동작했습니까?

위의 변경사항을 살펴보면 이전에는 `bundle` 명령을 경유해서 `puma`를 실행하고
있었음을 알 수 있습니다. 다시 말해 execute가 인식하는 명령은
`bundle exec hogehoge`이므로 `deploy.rb`가 덮어쓴 목록으로도 대응할 수 있었던
것입니다! ~~유레카!~~

## 커스텀하고 싶은 경우에는 어떻게 해야할까요?

해당 변수에 개발자가 원하는 명령을 추가하고 싶다면 `append`를 쓰면 됩니다.

### `append`

[공식 문서](http://capistranorb.com/documentation/getting-started/configuration/)를
참고하도록 합시다.

> New in Capistrano 3.5: for a variable that holds an Array, easily add values to it using append. This comes in especially handy for :linked_dirs and :linked_files (see Variables reference below).

~~여지껏 모르고 있었지만~~ 좋은게 있었습니다.

우리는 동작하는 코드를 사랑하니까 `capistrano3-puma`의 코드를 보도록 하죠. XD

```ruby
# Ref: https://github.com/seuros/capistrano-puma/commit/c1d165948602a150f438b0a7f11835ac90852a54#diff-56c283d8f205519259d1321eb2dcf468R22

append :rbenv_map_bins, 'puma', 'pumactl'
# 위와 아래는 동일한 결과를 가져옵니다.
set :rbenv_map_bins, fetch(:chruby_map_bins).to_a.concat(%w{ puma pumactl })
```

## 결론

- `deploy.rb`에서는 함부로 덮어쓰지 말자.
- 변경이 필요한 경우에는 `append`, `remove`를 쓰자.
- 이하 버전은 업데이트를 하자.
