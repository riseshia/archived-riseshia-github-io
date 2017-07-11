---
layout: post
title: "From Rails system test to Travis CI with Headless Chrome"
date: 2017-07-11 20:16:45 +0900
categories:
---

이 글에서는 Rails 5.1에서 도입된 System Test를 Headless Chrome를 사용하여
실행하는 방법에 대해서 알아보고, 이를 Travis CI에서 동작시키는 방법에 대해서 알아봅니다.

## 소개

### Rails System Test

Rails 5.1부터, 프레임워크 레벨에서 Capybara를 사용하는 시스템 테스트를
지원합니다. 동작은 느리지만 자바스크립트 레벨의 동작까지 테스트 가능하며,
무엇보다 프레임워크에서 제공하는 트랜젝션 테스트를 사용할 수 있다는 점이 매력적입니다.

### Headless Chrome

Chrome 59부터 Headless 모드를 제공합니다. 이 덕분에 PhantomJS의 메인테이너는
[유지보수를 때려치기](https://groups.google.com/d/msg/phantomjs/9aI5d-LDuNE/5Z3SMZrqAQAJ)도 했죠.

## Rails system test Configuration

일단 예제부터 시작합시다. [여기](https://github.com/riseshia/system-test-example)에서
예제 프로젝트를 확인해보실 수 있습니다. 환경은 다음과 같습니다.

- Rails 5.1.2
- Ruby 2.4.0
- ChromeDriver 2.30

### Custom Driver

Capybara에서는 기본으로 selenuim, poltergeist, webkit 드라이버를 제공합니다.
그런 이유로 headless_chrome 드라이버를 별도로 등록할 필요가 있는데요.
등록은 다음과 같이 해주시면 됩니다.

```ruby
# test/support/capybara.rb

Capybara.register_driver(:headless_chrome) do |app|
  capabilities = Selenium::WebDriver::Remote::Capabilities.chrome(
    chromeOptions: { args: %w[headless disable-gpu] }
  )

  Capybara::Selenium::Driver.new(
    app,
    browser: :chrome,
    desired_capabilities: capabilities
  )
end

Capybara.javascript_driver = :headless_chrome
```

- `disable-gpu`는 빼먹으시면 안됩니다. 공식에서도 요구하는 설정이므로 조심하세요.
- `javascript_driver`도 잊지말고 설정해줍시다.

### Use Driver

그럼 이제 드라이버를 설정할 차례입니다.

```ruby
# test/application_system_test_case.rb

# https://github.com/rails/rails/issues/29688
ActionDispatch::SystemTestCase.driver
class ActionDispatch::SystemTesting::Driver # :nodoc:
  def register
    return unless [:selenium, :poltergeist, :webkit].include?(@name)

    Capybara.register_driver @name do |app|
      case @name
      when :selenium then register_selenium(app)
      when :poltergeist then register_poltergeist(app)
      when :webkit then register_webkit(app)
      end
    end
  end
end

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  require_relative 'support/capybara'
  driven_by :headless_chrome
end
```

뭔가 이상한 몽키패칭이 끼어 있습니다. 이는 Rails의 버그 때문인데요,
자세한 건 링크를 참조하시고, 이렇게 하지 않으면 등록한 드라이버를 정상적으로
사용할 수 없다는 것만 이해하시면 됩니다. 해당 버그는 이미 수정되었으며 다음
릴리스에서 적용될 예정입니다.

### Write Test

```ruby
require 'application_system_test_case'

class HomeTest < ApplicationSystemTestCase
  test 'visits home index' do
    visit home_index_url
    assert_selector 'h1', text: 'Home#index'
  end

  test 'shows hello if user click "click"' do
    visit home_index_url
    click_on 'click'
    assert_selector '#result', text: 'hello'
  end
end
```

이런 느낌으로 작성하시면 됩니다. 참 쉽죠? 테스트는 `bin/rails test:system`으로
실행하실 수 있습니다. `bin/rails test`로는 시스템 테스트가 함께 실행되지 않는다는
부분만 주의하시면 됩니다.

## Setup `.travis.yml`

```yml
language: ruby
cache: bundler
dist: trusty
env:
  DISPLAY: :99.0
  RAILS_ENV: test
  CHROME_DRIVER_VERSION: '2.30'
addons:
  chrome: stable
before_script:
  - bin/travis_system_setup.sh
script:
  - bin/rails test:system
install: bundle install --without development
```

주의깊게 볼 부분을 설명합니다.

- trusty 이미지를 사용하시면 크롬을 정상적으로(편하게) 깔 수 있습니다.
- `addons`으로 크롬을 설치합니다.
- `bin/travis_system_setup.sh`로 필요한 설정을 수행하고 있습니다.

`bin/travis_system_setup.sh`을 살펴보죠.

```bash
#!/bin/bash
wget "http://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip"
unzip chromedriver_linux64.zip
rm chromedriver_linux64.zip
mkdir /home/travis/bin
mv chromedriver /home/travis/bin
gem update --system
bin/rails db:migrate
```

Selenium에서 Headless Chrome을 조작할 수 있도록 하려면 ChromeDriver가 필요합니다.
그래서 여기에서는 ChromeDriver를 받고 Selenium에서 불러올 수 있도록 PATH 상의
어딘가(여기에서는 `/home/travis/bin`)로 옮기고 있습니다.

## 결론

이제 Phantom.js 설정때문에 CI에서 브라우저 테스트를 못한다는 변명은 할 수
없는 세상이 왔습니다. 간만에 시스템 테스트해보다가 발견한 사실인데,
이제 ajax wait같은거 일일히 신경 안써도 알아서 잘 기다려주는군요.
역시 기술을 발전은 참 좋습니다. Let's Test!

## Reference

- [Rails 5.1 Release](http://weblog.rubyonrails.org/2017/4/27/Rails-5-1-final/)
- [Getting Started with Headless Chrome](https://developers.google.com/web/updates/2017/04/headless-chrome)
- [ChromeDriver](https://github.com/SeleniumHQ/selenium/wiki/ChromeDriver)
- [ChromeDriver on Travis CI Gist](https://gist.github.com/chitoku-k/67068aa62aa3f077f5307ca9a822ce74)
- [Rails bug: Cannot register custom Capybara drivers](https://github.com/rails/rails/issues/29688)
- [Running feature specs with Capybara and Chrome headless](https://drivy.engineering/running-capybara-headless-chrome/)
- [Using the Chrome addon in the headless mode in Travis CI](https://docs.travis-ci.com/user/gui-and-headless-browsers/#Using-the-Chrome-addon-in-the-headless-mode)
