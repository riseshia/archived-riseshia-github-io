---
layout: post
title: "include, extend in Ruby"
date: 2017-07-10 19:31:32 +0900
categories:
---

## 들어가기 전에

- Ruby 2.4.x에서 동작하는 코드입니다. 아마 1.9까진 잘 돌아갑니다.
- 싱글톤 클래스에 대한 이해가 필요합니다. 이해하시는 분은 이 글을 읽을 필요가 없겠지만...

## `include`

우선 문서를 봅시다.

> Invokes Module.append_features on each parameter in reverse order.

영문모를 말이 쓰여 있네요. 그럼 `append_features`를 확인해보죠.

> When this module is included in another, Ruby calls append_features in this module, passing it the receiving module in mod. Ruby’s default implementation is to add the constants, methods, and module variables of this module to mod if this module has not already been added to mod or one of its ancestors. See also Module#include.

드디어 어디서 본 내용이 나옵니다. 뭔가 무한루프에 빠질법한 설명이지만 모른 척 합니다. 요는 타겟 모듈에 넘겨받은 모듈의 상수, 메소드, 모듈 변수를 추가해준다는 부분인데, 정확하게는 넘겨받은 모듈을 상속 체인에 추가해줍니다. 그래서 메소드, 변수 탐색이 가능하도록 만드는 것이죠.

```ruby
module A; end
module B; end
Base = Class.new

Base.include A
Base.include B

Base.ancestors
# => [Base, B, A, Object, Kernel, BasicObject]
```

여기서 알 수 있는 사실은 다음과 같습니다.

- include는 상속 체인에 직접 끼워 넣는다.
- include한 모듈을 타겟 모듈의 위에 순서대로 쌓아 올린다(append)

이를 보면 include된 모듈의 함수를 인스턴스 메소드로 호출할 수 있는 이유를 알 수 있습니다.

## `extend`

그럼 `extend`는 어떨까요? 이번에도 문서부터 시작합시다.

> Adds to obj the instance methods from each module given as a parameter.

넘겨 받은 모듈로부터 타겟 객체의 인스턴스 메소드를 추가합니다. 코드를 보죠.

```ruby
module A
  def hello
    "Hello"
  end
end

class Base
  def hello
    "Base"
  end
end

base = Base.new
base.hello #=> "Base"
base.extend(A)
base.hello #=> "Hello"

base.include?(:hello) # => false
base.singleton_methods.include?(:hello) # => true
```

이로부터 다음과 같은 사실을 얻을 수 있습니다.

- 해당 객체의 싱글톤 클래스에 메소드를 추가합니다.
- 탐색 순서는 `A` -> `Base`가 됩니다.

## `include`, `extend`

여기까지만 보면 왜 `include`와 `extend`를 헷갈리는지 이해할 수 없습니다. 혼란을 야기하는 포인트는 클래스에 대해서도 `extend`가 가능하다는 지점이죠.

```ruby
module A
  def hello
    "Hello"
  end
end

class Klass
  def hello
    "klass"
  end
end

instance = Klass.new
instance.hello # => "klass"
Klass.extend(A)
instance.hello # => "klass"
Klass.hello # => "Hello"
```

클래스를 `extend`하는 경우, 해당 메소드가 클래스 메소드가 되는 것을 확인할 수 있습니다. 싱글톤 클래스에 메소드를 추가한다는 사실을 생각해보면 간단하게 이해할 수 있습니다. 어떤 클래스의 싱글톤 클래스의 인스턴스 메소드는 어떤 클래스의 클래스 메소드라는 점이죠. 설명이 복잡하니 다시 코드로 돌아갑시다.

```ruby
class A
  def self.hello
    "A"
  end

  class << self # A의 싱글톤 클래스를 연다
    def hello
      "A"
    end
  end
end
```

위에서 선언한 두 함수는 완전히 동치입니다. 다시 말하자면 루비에서의 클래스 메소드란, 싱글톤 클래스의 인스턴스 메소드입니다. 그러므로 클래스에 대해서 `extend`하는 경우, 결과적으로 클래스 메소드를 추가하는 것이 됩니다.

## 마무리

- `include`는 상속 체인을 확장합니다.
- `extend`는 객체의 싱글톤 클래스를 확장합니다.
- 잘 이해가 안되면 [이 책](https://pragprog.com/book/ppmetr2/metaprogramming-ruby)을 추천합니다.
