---
layout: post
title: "Refactoring Series - go out"
date: 2016-08-15 17:45:40 +0900
categories:
---

가끔은 의식적으로 이유를 붙여가며 리팩토링해보는 것도 좋은 경험이 될 것 같아서 레포트 형식으로 작성하는 시리즈입니다. 소재는 여기저기서 제가 보던 코드를 가져오고 있으며, 물론 그대로 가져오진 않고 필요한 부분만 가져다가 새로 예제로 만듭니다. 필자가 게으르기 때문에 비정기로 연재될 예정입니다. 의견 및 오타 제보는 언제나 감사하게 받고 있습니다. 😆

ps: 소제목에는 큰 의미가 없습니다. 재미로 봐주세요.

## 배경 설명

레일스 애플리케이션에 `Item`이라는 모델이 있고, 가격(`price`)으로 범위 지정을 하여 목록을 가져오는 기능을 구현해둔 상태입니다. 실제 코드는 대강 다음과 같습니다.

```ruby
# item.rb
class Item < ApplicationRecord
  include RangeParamConvertable
  
  def self.search(params)
    # ...
    price_where = {
      price: convert_ranges(params[:price_range])
    }.compact
    # ...
  end

  private

  def self.convert_ranges(ranges)
    return if ranges.nil?
    ranges.map(&method(:str_to_range))
  end
end

# range_param_convertable.rb
module RangeParamConvertable
  extend ActiveSupport::Concern

  class_methods do
    def str_to_range(string)
      # ...
      Range.new(start, end)
    end
  end
end
```

## 문제점?

문자열에 `str_to_range`를 호출하고 싶어서 `method`를 통해 해당 컨텍스트에 등록된 `str_to_range` 함수를 가져오고 있습니다. 비유하자면 `Array.prototype.slice.call(args, 0)` 같은 느낌이죠. 네.

문자열을 범위 객체로 만드는 함수니까 명백하게 `Item`이라는 모델의 책임 범위를 벗어납니다. 그런데 정작 쓰는 건 `include`해서 쓰니까 호출하는 방식이 뭔가 괴상합니다. 그렇게 생각하니 검색용 파라미터를 받아서 가공하는 작업을 여기서 해도 되는건가? 라는 생각도 듭니다. 어떻게 하면 될까 고민해본 결과, 선택할 수 있는 방식은 대략 3개쯤 있을 거 같습니다.

- 검색용 파라미터를 받아서 가공하는 책임을 가지는 별도의 클래스를 만들고 가공 처리를 위임한다.
- RangeParamConvertable 모듈을 별도의 라이브러리로 분리하고, 이를 통해서 호출한다.
- 몽키패칭으로 String 클래스에 해당 메소드를 주입한다.

여러개를 섞어서 사용할 수도 있고, 아닐 수도 있지만 설명의 편의상 이렇게 적어봅시다.
첫번째의 경우 파라미터 가공에 대한 책임을 외부로 내보낼 수 있다, 같은 장점이 있을거고, 두 번째는 비교적 간단하게 고칠 수 있고, 범용으로 쓰기 편합니다. 세 번째는 가장 자연스럽게 사용할 수 있다(`"1..2".to_range`와 같이 사용할 수 있기 때문이죠.)와 같은 장점이 있습니다.

일단 몽키패칭을 제외합시다. DSL을 만드는 것도 아니고, 기본 클래스를 만지는건 왠지 꺼림칙합니다. 그리고 첫 번째도 제외하죠. 별도로 만드는 것도 괜찮은 선택지입니다만, 현재 문제가 되는 부분 이외에서는 크게 이득을 보진 않습니다(생략된 코드에는 이러한 가공처리가 없습니다). 그러니까 두 번째로 갑시다.

## WIP

```ruby
# item.rb
class Item < ApplicationRecord
  def self.search(params)
    # ...
    price_where = {
      price: convert_ranges(params[:price_range])
    }.compact
    # ...
  end

  private

  def self.convert_ranges(ranges)
    return if ranges.nil?
    ranges.map { |range| RangeParamConvertable.str_to_range(range) }
  end
end

# range_param_convertable.rb
module RangeParamConvertable
  module_function

  def str_to_range(string)
    # ...
    Range.new(start, end)
  end
end
```

함수 호출이 매우[..] 길어졌습니다. 뿐만 아니라, 영 이름도 적절치 않게 되었네요. 변수의 이름과 함수/모듈명도 좀 더 알아보기 쉽게 고쳐보죠. `range`라는 문자열을 `range_string`로, `RangeParamConvertable`는 `StringConverter` 정도로, 클래스명을 함께 쓰게 되었으니 `str_to_range`는 `to_range` 정도는 어떨까요?

```ruby
# item.rb
class Item < ApplicationRecord
  def self.search(params)
    # ...
    price_where = {
      price: convert_ranges(params[:price_range])
    }.compact
    # ...
  end

  private

  def self.convert_ranges(range_strings)
    return if range_strings.nil?
    range_strings.map do |range_string|
      StringConverter.to_range(range_string)
    end
  end
end

# string_converter.rb
module StringConverter
  module_function

  def to_range(string)
    # ...
    Range.new(start, end)
  end
end
```

## 결론

nil 처리를 `convert_ranges` 내부에서 하고 있다든가, 애초에 이 메소드가 여기에 있는게 맞긴 한것인지에 대한 의문이라든가, where 절을 만드는 방법이 영 맘에 안든다든가(방법이 있는 지는 둘째 치고), `map`이 3줄이 되어서 슬프다던가, 이런 부분은 재껴두고, 좀 나아졌습니다~~, 라고 믿습니다~~.

욕심을 내지 않고 조금씩 고치는 것도 요령이니까요! ~~절대 귀찮아서가 아닙니다.~~
