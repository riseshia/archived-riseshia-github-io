---
layout: post
title: "Ruboty 로 공부하는 플러그인 시스템"
date: 2018-04-22 21:26:01 +0900
categories:
---

[Ruboty](https://github.com/r7kamura/ruboty)는 루비로 작성된 봇 프레임워크입니다.
Hubot의 Ruby 판이라고 생각하면 이해하기 쉽습니다. 다른 점이라면, http endpoint 가
기본으로 제공되지 않는다는 점일까요[이 기능에 대한 제안 PR이 현재 진행형입니다].
[Lita](https://github.com/litaio/lita)도 좋지만 Ruboty는 간단한 코드와 적당한
추상화로 직접 플러그인을 추가하기 편하다는 장점이 있습니다.
이 글에서는 Ruboty 의 플러그인 시스템에 대해서 설명합니다.

## 내부 구조 설명

Ruboty의 내부 클래스는 이렇습니다.

### Ruboty::Commands::*

Ruboty 의 내부 명령을 다룹니다. 정확히는, `ruboty`를 실행하였을 때 사용되는 명령어 클래스입니다.
공통 인터페이스로 `#call` 을 가지고 있습니다. 실행시 넘겨준 옵션에 따라 적당한 클래스가 호출됩니다.

### Ruboty::Robot

챗봇의 본체입니다.

### Ruboty::Brains::*

hubot 의 brain 과 동일한 개념이며, `#data` 를 통해서 데이터를 조작할 수 있습니다. 

### Ruboty::Handlers::*

Ruboty 가 인식하는 명령어 패턴을 가지는 핸들러입니다. 각 핸들러는 여러 개의 Action을 가질 수 있습니다.

`#on` DSL을 통해서 어떤 패턴에 반응할 지를 결정할 수 있습니다.

### Ruboty::Actions::*

Action 은 구체적으로 행동을 수행하고, 넘겨받은 `message.reply` 응답을 돌려줍니다. 공통 인터페이스로 `#call`을 가집니다.

### Ruboty::Adapters::*

어댑터는 로봇과 실제 채팅방 사이의 통신을 관리합니다.
구체적으로는, `#run`을 통해서 사용자의 입력을 받을 수 있는 상태를 유지하고 로봇이 돌려주는 응답을 실제로 처리할 수 있도록 `#say` 메소드를 제공합니다.

### Ruboty::Env::Validatable

플러그인에서 어떤 환경변수를 요구하는 경우, 이 모듈을 통해서 환경변수를 넘기도록 강제할 수 있습니다.

## 요청을 처리하는 흐름

- Robot#run을 통해서 시스템을 기동합니다.
- Adaptor#run 에서 사용자의 입력을 기다립니다.
- 받은 입력을 Rubot#receive 에서 각 핸들러로 넘기고, 가능한 응답을 모두 생성합니다.
- 두번째로 돌아갑니다.

이상. 구체적인 코드는 저장소를 확인하세요.

## 그래서 플러그인 시스템

위에서 설명한 것들중, `Commands` 를 제외한 `::*` 로 적혀있는 네임스페이스는 플러그인 시스템을 지원합니다. 동작 방식은 다음과 같습니다.

- `Base` 클래스를 상속한다.
- 필요한 인터페이스(i.e. `#call`)를 구현한다.

마법이 벌어지는 지점은 첫번째 부분입니다. 필요한 `Base` 클래스를 상속하는 것으로 플러그인으로서 동작가능하게 되는데, 비밀은 `.inherited`에 있습니다.

`.inherited`는 클래스가 상속된 직후에 불리는 콜백 메소드입니다. 예를 들어, `Ruboty::Handlers::Base`를 볼까요.

```ruby
class Ruboty::Handers::Base
  class << self
    def inherited(child)
      Ruboty.handlers << child
    end
  end
end
```

그리고 기본으로 포함되어있는 Ping 핸들러를 보죠.

```ruby
module Ruboty
  module Handlers
    class Ping < Base
      on(/ping\z/i, name: "ping", description: "Return PONG to PING")

      def ping(message)
        Ruboty::Actions::Ping.new(message).call
      end
    end
  end
end
```

`Base` 클래스를 상속하면, `Ruboty::Handers::Base.inherited`가 호출되고, 이 안에서 Roboty가 사용 가능한 핸들러 목록에 해당 클래스를 추가하게 됩니다.
그리고 계속해서 이야기하지만, 이는 Brains, Actions, Adaptors에 모두 일관성있게 적용되는 부분입니다.

## 결론

Ruby에서 플러그인 시스템을 구현하는건 참 쉽습니다.
ps: Ruby에는 `.inherited` 이외에도 `.method_added` 와 같은 다양한 콜백 메소드가 존재합니다.
