---
layout: post
title: "ActionController::UrlGenerationError in RSpec"
date: 2017-01-31 16:03:12 +0900
categories:
---

RSpec에서 이러한 에러가 발생하는 경우가 있습니다.
회사 프로젝트에서 이런 문제가 있었는데, 괴이한 점은 CI에서는 멀쩡하게 돌고 로컬에서만 실패한다는 점입니다.
이러나저러나 로컬에서 잘 알 수 없는 이유로 테스트가 실패하는 것은 불편하기도 하고,
기분나쁘기도 해서 이번에 동료와 함께 조사한 결과를 정리해봤습니다.

## Situation

```
Failure/Error: before { get :edit, params: { id: user.id } }

ActionController::UrlGenerationError:
  No route matches {:action=>"edit", :controller=>"users", :id=>1}
# ./spec/controllers/admins/users_controller_spec.rb:8
```

- 테스트 셋 전체를 실행하면(로컬환경) 실패한다.
- 해당 테스트만을 실행하면 성공한다.

일단 테스트끼리 뭔가 꼬였구나, 하는 촉은 왔습니다.
우선 테스트 중에 라우터 자체를 만지는 부분이 없는지 찾아봤지만 그런 부분은 없었습니다.

키워드로 검색하면 될 걸 가지고 ~~욕심만 있어서~~ 스택 트레이스를 뒤지는 사이,
동료가 [이런 정보](http://stackoverflow.com/questions/19027199/rspec-controllers-in-and-out-of-namespace-with-same-name)를 찾아왔습니다.
역시 구글링은 위대합니다.

## Reproduce

재현하는 법은 간단합니다. 이런 테스트 코드를 작성하면 됩니다.

```ruby
# spec/controllers/admins/users_controller_spec.rb
require "rails_helper"
require "users_controller"

module Admins
  RSpec.describe UsersController, type: :controller do
    describe "#index" do
      before { get :index }
      it { is_expected.to redirect_to(new_admins_session_path) }
    end
  end
end
```

동일한 이름의 컨트롤러(이 경우, `UsersController`)가 전역에 존재하고,
이 테스트가 `UsersController`에서 동작하지 않는다면 이 테스트는 언제나 실패합니다.
왜일까요?

## Why??

Rails를 사용하면서 이런 의문을 가져보신 적은 없으신가요?

> require를 거의 안쓰는데 왜지?

Rails에는 상수를 자동으로 불러오는 기능이 있습니다.
Rails 개발자라면 누구나 의존하고 있는 기능이기도 하죠.
이 테스트가 실패하는 요인은 바로 그 습관에서 비롯됩니다.

위 코드를 설명하면 이렇습니다.

```ruby
# spec/controllers/admins/users_controller_spec.rb
require "rails_helper"
require "users_controller" # UsersController라는 상수를 불러온다.

module Admins
  RSpec.describe UsersController, type: :controller do
  # 윗 줄에서 `UsersController`를 요청한다.
  # 실제로 원하는 것은 `Admins::UsersController`이지만,
  # 현재 컨텍스트에는 이미 `UsersController`가 존재하므로 루비는 이 상수를 넘겨준다.
  # 결과 Rails의 상수 자동 로딩이 동작하지 않는다.
    describe "#index" do
      before { get :index }
      # 그리고 `UsersController`에게 리퀘스트를 생성하여 던지므로
      # 당연하다는 듯이 테스트가 실패한다.
      it { is_expected.to redirect_to(new_admins_session_path) }
    end
  end
end
```

언제나 일처리를 싹싹하게 해주던 상수 자동 로딩이 발생하지 않는 슬픈 상황이 되었습니다.

## Solution

다음 3가지 방법이 있습니다.

### `require`!!!

```ruby
# spec/controllers/admins/users_controller_spec.rb
require "rails_helper"
require "users_controller"
require "admins/users_controller" # 강제로 불러온다.

module Admins
  RSpec.describe UsersController, type: :controller do
    describe "#index" do
      before { get :index }
      it { is_expected.to redirect_to(new_admins_session_path) }
    end
  end
end
```

자동 로딩이 일을 하지 않으면 우리가 할 수 밖에 없잖아! 같은 느낌.

### 상수명 변경하기

```ruby
# spec/controllers/admins/users_controller_spec.rb
require "rails_helper"
require "users_controller"

# 원하는 상수명을 명시하여 루비가 원하지 않는 동작을 하지 않도록 만듬
RSpec.describe Admins::UsersController, type: :controller do
  describe "#index" do
    before { get :index }
    it { is_expected.to redirect_to(new_admins_session_path) }
  end
end
```

상수명에 네임스페이스를 지정하면 상수 탐색 순서가 변경되므로 확실하게 자동 로딩을 부려먹을 수 있습니다.

### 자동 로딩을 정지합니다...

```ruby
# config/environments/test.rb

# Do not eager load code on boot. This avoids loading your whole application
# just for the purpose of running a single test. If you are using a tool that
# preloads Rails for running tests, you may have to set it to true.
config.eager_load = true # 기본값은 false
```

테스트를 실행하기 전에 코드를 미리 읽도록 만듭니다.
설명에도 적혀있듯, 단일 테스트의 속도를 떨어뜨리는 요인이 될 수 있습니다.

## Conclusion

그렇다고 합니다. 여유가 있으시다면 Rails의 상수 자동 로딩이 어떻게 동작하는지에 대해서 읽어보시는 건 어떨까요? XD

ps: 그리고 CI에서만 성공하던 이유는 미궁속에...

## Reference

- [Rails 가이드(한) - 상수 자동 로딩과 리로딩](http://guides.rorlab.org/autoloading_and_reloading_constants.html)
- [Constant resolution in Ruby](http://valve.github.io/blog/2013/10/26/constant-resolution-in-ruby/)
