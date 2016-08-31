---
layout: post
title: "New Rails Test Runner"
date: 2016-08-31 20:12:03 +0900
categories:
---

## New Rails Test Runner

Rails 5에서는 새로운 테스트 러너가 도입되었습니다. 알고 계시나요?
이번에는 새로 도입된 테스트 러너의 기능에 대해서 알아볼까 합니다. 짧은 요약본은 아래의 릴리스 노트를 참고하세요.

### `bin/rails test`

기본적인 테스트 실행은 이전과 동일합니다. 단 `rake` 명령들이 전부 `rails`로 통합되었으므로
이제 `bin/rake test`가 아닌 `bin/rails test`를 통해서 실행할 수 있습니다.

### 친절한 테스트 러너씨

이제 테스트 러너에서 특정 테스트/파일/폴더를 실행할 수 있게 되었습니다. 구체적인 사용방법은 RSpec의 사용법과 동일합니다.

- 테스트의 줄번호를 지정하여 한 테스트만을 실행
```bash
# 6번째 줄에 있는 테스트를 실행
$ bin/rails test test/models/article_test.rb:6
```
- 테스트 파일을 지정하여 파일 내의 테스트를 실행
```bash
# test/models/article_test.rb의 모든 테스트를 실행
$ bin/rails test test/models/article_test.rb
```
- 폴더를 지정하여 폴더 내의 모든 테스트를 실행
```bash
# test/models의 모든 테스트를 실행
$ bin/rails test test/models
```
- 매칭되는 테스트를 실행
```bash
# test_the_truth라는 문자열과 테스트 이름이 매칭되는 모든 테스트를 실행
$ bin/rails test -n test_the_truth
```
- 매칭되지 않는 테스트만을 실행
```bash
# test_the_truth라는 문자열과 테스트 이름이 매칭되지 않는 모든 테스트를 실행
$ bin/rails test --exclude test_the_truth
```
- 실패한 경우에 보여주는 메시지가 개선되어, 실패한 테스트를 곧장 재실행할 수 있게 되었습니다. 이예이!
```bash
$ bin/rails test test/models/article_test.rb
Run options: --seed 44656

# Running:

F

Failure:
ArticleTest#test_should_not_save_article_without_title [/path/to/blog/test/models/article_test.rb:6]:
Expected true to be nil or false

bin/rails test test/models/article_test.rb:6

Finished in 0.023918s, 41.8090 runs/s, 41.8090 assertions/s.

1 runs, 1 assertions, 1 failures, 0 errors, 0 skips
```

### 새로운 실행 옵션

- `--environment`, `-e` 옵션을 사용해 테스트가 실행될 환경을 지정할 수 있습니다.
- `--backtrace`, `-b` 옵션을 사용하면 예외에 대한 전체 백트레이스를 얻을 수 있습니다.
- `--defer-output`, `-d` 옵션을 켜면 테스트가 완료될때까지 메시지 출력을 미룹니다.
- `--fail-fast`, `-f` 옵션을 켜면 실패했을 때에 곧바로 테스트를 정지할 수 있습니다.

### 결론

레일스를 개발하며 Minitest를 쓸지, Rspec을 쓸지 고민하는 경우가 꽤 있습니다.
이 둘을 비교할 때에 Minitest를 사용하는 경우의 단점으로 많이 나오는 이야기 중 하나는 단일 테스트,
파일의 테스트, 폴더 내의 테스트를 실행하기가 난해하다는 점인데요. 이 때문에 [m](https://github.com/qrush/m)과
같은 잼도 나오곤 했습니다만, 적어도 레일스 환경 내에서는 더 이상 이런 것은 변명이 될 수 없게 되었습니다.

Minitest, 한 번 써보지 않으실래요? XD

### 참고자료

- [Rails 5 릴리스 노트](http://guides.rorlab.org/5_0_release_notes.html#테스트-러너)
- [Rails Guide - 테스트하기](http://guides.rorlab.org/testing.html#레일스-테스트-러너)

