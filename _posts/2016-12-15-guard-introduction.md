---
layout: post
title: "Re; Guard부터 시작하는 테스트 생활"
date: 2016-12-15 16:32:33 +0900
categories:
---

이 글은 [Advent Calender 2016 for Ruby Korea](https://ruby-korea.github.io/advent-calendar/)를 위해서 작성되었습니다.

- 15일: 미정
- 17일: 미정

## 들어가기 전에

Minitest를 손쉽게 Guard로 동작시키는 방법에 대해서 알아볼까 합니다.

사용된 환경 구성은 다음과 같습니다.

- Ruby 2.3.x
- Rails 5.0
- Minitest 5.10

## Guard?

[Guard](https://github.com/guard/guard)는 파일이나 디렉토리의 변경사항을 추적하여 특정 작업을 수행하기 위한 도구입니다.
그 중에서 가장 흔한 경우는 소스/테스트 파일이 변경되는 경우 테스트를 자동으로 수행한다, 가 있습니다.
여기에서도 그런 용도로 사용하며 기존에 파일/폴더 단위 테스트가 어려웠던 Minitest를 대상으로 합니다.

## Configuration

필요한 젬은 다음 두 가지 입니다.

```ruby
group :development do
  gem "guard"
  gem "guard-minitest"
end
```

젬을 설치하고 필요한 초기 파일을 생성해보죠.

```ruby
bundle
bundle exec guard init minitest
```

`Guardfile`이 생성됩니다. 내용물을 조금 수정하죠.

```ruby
guard :minitest, spring: "bin/rails test" do
  watch(%r{^app/(.+)\.rb$})                               { |m| "test/#{m[1]}_test.rb" }
  watch(%r{^app/controllers/application_controller\.rb$}) { "test/controllers" }
  watch(%r{^app/controllers/(.+)_controller\.rb$})        { |m| "test/controllers/#{m[1]}_test.rb" }
  watch(%r{^lib/(.+)\.rb$})                               { |m| "test/lib/#{m[1]}_test.rb" }
  watch(%r{^test/.+_test\.rb$})
  watch(%r{^test/test_helper\.rb$}) { "test" }
end
```

주석으로 이미 생성되어 있는 Rails > 4 이상의 설정을 그대로 가져왔습니다. 간단히 해설하자면 `app/(.+)\.rb`에 해당하는 파일이 변경되면, `test/#{m[1]}_test.rb`를 실행하겠다, 라는 의미입니다. 이러한 조건들을 여러 개 늘어놓은거죠.

Scaffold된 코드에서 달라진 점은 하나인데요.
바로 `spring: "bin/rails test"`라는 설정입니다. 복잡한 설정을 피하기 위해서 `spring` 옵션을 사용하면 문제없이 가드를 사용할 수 있게 됩니다.

```
16:07:22 - INFO - Running: test/models/app_test.rb
[Coveralls] Set up the SimpleCov formatter.
[Coveralls] Using SimpleCov's default settings.
Run options: --seed 33956

# Running:

.......

Fabulous run in 0.083183s, 84.1518 runs/s, 84.1518 assertions/s.

7 runs, 7 assertions, 0 failures, 0 errors, 0 skips
```

위 로그는 `app/models/app.rb`를 변경한 결과입니다. 무사히 원하는 테스트만을 실행하고 있네요.

설정은 여기까지입니다. 그럼 이제 실행하는 방법만 배우면 되겠네요.

```ruby
bundle exec guard
```

그러면 REPL이 하나 뜨면서 테스트가 실행됩니다. 축하합니다! 이제 파일이 변경되면 변경된 파일에 관련된 테스트가 자동으로 실행되게 됩니다.

## Conclusion

Guard를 사용하는 방법을 알아보았습니다. 이를 통해서 테스트에 대한 허들이 하나 더 낮아졌으면 좋겠습니다. XD

ps: `Rails 5.0`에서 Minitest라도 단일 파일, 단일 테스트를 돌릴 수 있는 [테스트 스텁이 추가되었다는 사실](http://guides.rorlab.org/upgrading_ruby_on_rails.html#태스크와-테스트를-실행하기-위해서-bin-rails를-사용)을 알고 계시나요? 이를 사용하면 손으로도 편하게 필요한 파일 전체/단일 테스트 등을 손쉽게 돌릴 수 있게 되었습니다. 한번 사용해보시면 어떨까요?
