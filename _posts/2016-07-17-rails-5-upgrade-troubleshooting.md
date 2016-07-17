---
layout: post
title: "Rails 5 Upgrade Troubleshooting"
date: 2016-07-17 14:29:10 +0900
categories:
---

# Rails 5 Upgrade Troubleshooting

[Rails 5가 릴리스](http://weblog.rubyonrails.org/2016/6/30/Rails-5-0-final/)된 지 대략 2주가 흘렀습니다. 회사에서도 도입하고, 개인 플젝도 하나씩 버전을 올리고 있는 와중에 있었던 상황들에 대한 몇몇 팁들을 정리해보았습니다.

## Requirement

우선 [Rails Guides](http://guides.rubyonrails.org/upgrading_ruby_on_rails.html)를 참고하세요. 한국어판은 아직 갱신되지 않았습니다만, 생각보다 영문판도 그럭저럭 읽을 만 합니다. 이 글에서는 Rails 4.2 -> Rails 5에 대해서 다룹니다.

기본적인 요구조건은 준비가 되었다고 가정합니다.

### 개략적인 흐름

* 각 Gem들이 Rails 5에 대응하는지 확인(대응하는 버전이 몇인지 확인해 둘 것)
* Rails 5 && `bundle update`
* `rails app:update` 실행하기
* `ApplicationRecord` && `ApplicationJob` 추가하기
* `throw(:abort)` 추가하기
* Controller Test에서 명시적으로 인자들을 넘기도록 변경
* Hash처럼 다루던 Parameter를 수정하기

~~더 설명하면 끝이 없으니~~ 자세한 절차는 가이드를 참고하세요.

## Troubleshooting

이하는 마주칠 법한 ~~-정확히는 제가 만났던-~~ 문제들에 대하여 다룹니다.

### `bundle update`가 안됩니다.

Rails 5를 지원하지 않는 Gem이 포함되어 있을 가능성이 높습니다. 하나하나 지워나가면서 범인을 찾아서 버전을 올려주세요. (i.e: jquery-rails, simple_form, draper, sinatra, ...)

평소에 각 Gem들의 최신판을 잘 적용하고 있었다면 업데이트에서는 큰 문제가 없을 가능성이 높습니다. 다만 적용 여부는 신중하게 결정해주세요. 최신판도 Rails 5에 대응하지 않는 경우 해당 Gem의 Github을 찾아가시면 `master` 브랜치에 아직 릴리즈가 되지 않았지만 관련 커밋이 이미 포함되어 있을 가능성도 있습니다.

### `mysql` 관련 에러가 나요!

Rails 5에서는 `mysql` Gem에 대한 지원이 끝났고 `mysql2`만을 지원하게 되었습니다. (직접 해당 어댑터를 사용하는 경우는 드물기 때문에) 문제가 없을거라 생각하지만, 만약 문제가 있다면 어디가 원인인지 확인해보세요.

구 버전에서 `mysql`로 캐싱되어 있다가 새 버전이 올라가자 캐시에서 객체를 복원하지 못하는 경우도 있으니 만약 캐시를 사용하고 있다면 캐시를 지워보는 것도 방법입니다.


### Helper Test가 실패해요!


테스트에서 Route helper를 사용할 수 없다는 에러가 뜬다면, 잘 찾아오셨습니다. 아마도 `assign`, `assert_template`을 사용하기 위해서 `rails-controller-testing` Gem을 추가하셨을 겁니다. 해당 Gem과 관련한 [버그](https://github.com/rails/rails-controller-testing/issues/24)가 있으며, Rails 본체가 아닌 관련된 Gem들이 해당하는 이슈를 해결해야하는 상황이라 해결에는 좀 더 시간이 걸릴 것으로 보입니다.

이 문제를 어떻게 다룰지는 각자의 몫입니다만, [ActionController::TestCase가 5.1에서 본체로부터 분리될 예정](https://github.com/rails/rails/blob/master/actionpack/lib/action_controller/test_case.rb#L26)이니 의사결정에 참고하세요.

### `/lib` 폴더에 있는 클래스들이 로딩되지 않습니다.

[이 설명](http://guides.rubyonrails.org/upgrading_ruby_on_rails.html#autoloading-is-disabled-after-booting-in-the-production-environment)을 참고하세요.

### 배포환경에서 `puma`를 사용하는 경우

`bundle exec puma`로는 서버를 기동할 수 없습니다. 그러므로 이러한 경우에는 `Gemfile`에 명시적으로 `gem "puma"`를 추가해주도록 합시다.

## Conclusion

올리자마자 아예 동작하지 않게끔 만드는 변경점은 Parameters 객체를 다루는 방법이 바뀐 것과 Autoloading과 관련된 패치 정도입니다. 대부분은 `"~~~" will be deprecated in Rails 5.1` 메시지를 주기 때문에 잘 작성된 테스트가 있다면 이것들을 하나하나 지워나가는 것으로, 생각보다 어렵지 않게 대응할 수 있습니다. ~~그러니까 테스트를 작성합시다.~~

그럼 Rails 최신 버전과 함께 즐거운 개발하세요. :)

ps: 잘못된 설명이나, 추가 설명/의견이 있으시면 연락주세요!
