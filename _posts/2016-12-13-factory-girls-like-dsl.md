---
layout: post
title: "테스트 헬퍼 만들기"
date: 2016-12-13 14:48:38 +0900
categories:
---

이 글은 [Advent Calender 2016 for Ruby Korea](https://ruby-korea.github.io/advent-calendar/)를 위해서 작성되었습니다.

- 11일: [NMatrix 맛보기](https://github.com/ahastudio/til/blob/master/ruby/20161211-nmatrix.md) - @ahastudio님
- 13일: 미정

## 들어가기 전에

이 글에서 사용된 환경 구성은 다음과 같습니다.

- Ruby 2.3.x
- Rails 5.0
- Minitest 5.10

## 테스트 작성하기

`Item`이라는 클래스를 만들고, `mention`이라는 이름의 메소드를 통해
`name`에 `@`를 붙여서 반환받는 메소드가 있다고 생각해봅시다.

```ruby
class ItemTest < ActiveSupport::Test
  def test_mention_returns_at_with_name
    item = Item.new(name: "some_name")
    assert_equal "@some_name", item.mention
  end
end
```

그 클래스는 이런 테스트를 통과해야할 겁니다.
그리고 이러한 메소드를 여러 개 테스트해야해서 매번 `Item`의 객체를 만드는게 너무 귀찮다고 해보죠.
이 상황에서는 기본값을 가지는 `Item` 객체가 있으면 좀 더 편해질 것 같습니다.

```ruby
class ItemTest < ActiveSupport::Test
  def build_item
    Item.new(name: "some_name")
  end

  def test_mention_returns_at_with_name
    item = build_item
    assert_equal "@some_name", item.mention
  end
end
```

조금이지만 다음 테스트를 작성하기 편해졌습니다.
그런데 name이라는 속성은 저장하기 전에 반드시 가지고 있어야 하는 값이라고 한다면?

```ruby
class ItemTest < ActiveSupport::Test
  def build_item
    Item.new(name: "some_name")
  end

  def test_mention_returns_at_with_name
    item = build_item
    assert_equal "@some_name", item.mention
  end

  def test_validate_name_presence
    item = build_item
    item.name = nil # ???

    # or 
    item = Item.new
    assert item.invalid?
  end
end
```

이러한 정체불명의 코드가 됩니다. 또는 만든걸 쓰지 않고 처음부터 만드는 방법도 있죠.
하지만 이왕 만든 걸 그대로 재활용하면 좋을 것 같습니다. 개선해보죠.
우선 호출 시점에 값을 넘길 수 있게 만들면 아쉬운대로 두가지 사용 방식을 전부 커버할 수 있을 것 같습니다.
그리고 자주 쓰이는 기본값들은 생성 함수가 가지고 있게끔 해보죠.

```ruby
class ItemTest < ActiveSupport::Test
  def build_item(attrs = {})
    default_attrs = { name: "some_name" }
    Item.new(default_attrs.merge(attrs))
  end

  def test_mention_returns_at_with_name
    item = build_item
    assert_equal "@some_name", item.mention
  end

  def test_validate_name_presence
    item = build_item(name: nil)
    assert item.invalid?
  end
end
```

이제 좀 나아졌네요.
새로운 사양을 추가해보죠.

```ruby
class ItemTest < ActiveSupport::Test
  def test_cached_properly_before_create
    item = Item.create(name: "some_name")
    assert_equal item.name, item.cached_name
  end
end
```

이번엔 DB를 저장하지 않으면 테스트할 수 없는 사양입니다. 어떻게 작성하면 좋을까요?
지금까지 써온 방법을 똑같이 써볼까요?

```ruby
class ItemTest < ActiveSupport::Test
  def build_item(attrs = {})
    default_attrs = { name: "some_name" }
    Item.new(default_attrs.merge(attrs))
  end

  def create_item(attrs = {})
    default_attrs = { name: "some_name" }
    Item.create(default_attrs.merge(attrs))
  end

  def test_cached_properly_before_create
    item = create_item(name: "some_name")
    assert_equal item.name, item.cached_name
  end
end
```

음. 기본값 목록은 비슷할텐데 따로 가지고 있는건 DRY하지 않으므로, 맘에 들지 않습니다.
생성하기 위해서는 우선 만들(build) 필요가 있으니, build_item을 재활용할 수 있지 않을까요?

```ruby
class ItemTest < ActiveSupport::Test
  def build_item(attrs = {})
    default_attrs = { name: "some_name" }
    Item.new(default_attrs.merge(attrs))
  end

  def create_item(attrs = {})
    build_item(attrs).tap(&:save)
  end
end
```

아주 좋습니다. > <

이제 이걸 좀 더 여러 모델에서 광범위하게 사용할 수 있도록 만들고 싶습니다.
어떻게 하면 좋을까요?

- 여러 모델명을 사용할 수 있게 한다.
- 각 모델에 맞는 기본값을 등록할 수 있게 만든다.

좋습니다! 한번 해보죠.
우선 모델 이름이 메소드에 포함되어 있으면 여러모로 불편하니 첫번째 인자로 분리시켜 봅시다.

```ruby
module ActiveSupport
  class TestCase
    def build(model, attrs = {})
      klass = Object.get_const(model.capitalize)
      default_attrs = { name: "some_name" }
      klass.new(default_attrs.merge(attrs))
    end

    def create(model, attrs = {})
      build(model, attrs).tap(&:save)
    end
  end
end
```

이 코드에서는 달라진 부분이 2가지 있습니다.

- 구현 위치가 `ActiveSupport::TestCase`로 이동했습니다. 이제 프로젝트 전체의 모델에 대한 생성&저장용 메소드가 되었기 때문입니다.
- 첫번째 인자를 통해서 적당한 모델 클래스를 가져오고 있습니다.
- 겸사겸사 매번 Shift를 누르기 귀찮으니 소문자로 된 모델을 사용할 수 있도록 `capitalize`를 호출해주고 있습니다.

이 DSL을 사용하는 테스트를 확인해보죠.

```ruby
class ItemTest < ActiveSupport::Test
  def test_cached_properly_before_create
    item = create(:item, name: "some_name")
    assert_equal item.name, item.cached_name
  end
end
```

좀 더 부담없이 사용할 수 있게 되었네요. 이걸로 첫번째 문제는 해결되었습니다.
다만 지금대로라면 `Item`의 기본값을 모든 모델이 공유하게 되므로 좀 곤란합니다.
이 값들도 사용자들이 원하는 방식대로 등록할 수 있다면 좋겠네요. 이를 위한 함수도 하나 만들어보죠.

```ruby
module ActiveSupport
  class TestCase
    def self.factory(model, attrs)
      factory_dictionary[model] = attrs
    end

    def self.default_attrs(model)
      factory_dictionary[model]
    end

    def self.factory_dictionary
      @@dictionary ||= {}
    end
  end
end
```

`@@`를 사용해서 클래스 멤버 변수를 만들고, 거기에 기본값들을 저장할 수 있도록 해봤습니다.
이를 통해서 테스트 어딘가에서,

```ruby
module ActiveSupport
  class TestCase
    factory :item, { name: "some_name" }
    factory :article, { title: "some_title" }
    # ...
  end
end
```

이와 같은 방식으로 각 모델에 필요한 기본값을 등록할 수 있습니다.

## 정리하기

이 시점에서 전체 테스트 코드 + factory 코드는 다음과 같습니다.

```ruby
# 테스트 준비시에 로드
module ActiveSupport
  class TestCase
    def self.factory(model, attrs)
      factory_dictionary[model] = attrs
    end

    def self.default_attrs(model)
      factory_dictionary[model]
    end

    def self.factory_dictionary
      @@dictionary ||= {}
    end
  end
end

# 기본값 등록
module ActiveSupport
  class TestCase
    factory :item, { name: "some_name" }
    factory :article, { title: "some_title" }
    # ...
  end
end

# 실제 테스트
class ItemTest < ActiveSupport::Test
  def test_mention_returns_at_with_name
    item = build(:item)
    assert_equal "@some_name", item.mention
  end

  def test_validate_name_presence
    item = build(:item, name: nil)
    assert item.invalid?
  end

  def test_cached_properly_before_create
    item = create(:item, name: "some_name")
    assert_equal item.name, item.cached_name
  end
end
```

## 결론

여러가지로 코드의 위치라던가, 구현 방법이라든가, 테스트 자체에 불만이 많을 수는 있습니다만,
일단 하고 싶은 목표는 달성했습니다.
어떤 모델 객체를 쉽게 생성하고, 삭제할 수 있게 만들었으며 이를 모든 모델에 걸쳐서
사용할 수 있도록 확장도 했습니다.
어디서 많이 본 것 같지 않나요? `FactoryGirl`과 무척 유사하지 않나요?
지금까지 저희는 픽스쳐와 이를 편하게 가져다 쓸 수 있는 DSL을 만들어 본거죠. ~~바퀴의 재발명~~

DSL은 생각보다 어렵지 않습니다. 그렇지 않나요? XD

ps: 모델 코드는 의도적으로 생략했습니다. 경험이 없으신 분들은 이번 기회에 TDD를 접해보시는건 어떨까요?
