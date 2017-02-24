---
layout: post
title: "Dancing with Fixture"
date: 2017-02-24 10:15:39 +0900
categories:
---

Rails에는 테스트 데이터를 위해서 픽스쳐라는 구조를 기본으로 제공하고
있습니다. 지금까지 주변을 봐온 경험상, 저를 포함한 많은 분들이 픽스쳐 대신
[`factory_girl`](https://github.com/thoughtbot/factory_girl)을
쓰고 있습니다. 이전부터 왜 픽스쳐가 인기가 없는지에 대한 의문이 있었는데,
얼마전에 지인과 테스트에 관련된 이야기를 하다가 말이 나온 김에 개인 프로젝트의
테스트에서 픽스쳐를 사용하도록 변경하고, 이에 대한 감상을 정리해봤습니다.

## How to use

픽스쳐 이외의 테스트 데이터용 라이브러리를 사용하고 계시다면 신경써야 할 부분은
두 곳 정도입니다. 테스트 케이스에서 픽스쳐를 로드하도록 설정하기, 그리고
트랜잭션에서 테스트를 실행하는 설정을 활성화하기. Minitest라면 다음과 같은
느낌의 `test_helper.rb`가 되겠네요.

```ruby
module ActiveSupport
  class TestCase
    fixtures :all
    self.use_transactional_tests = true
  end
end
```

가져다 쓰는 방법에 대한 설명은 [API Document](http://api.rubyonrails.org/classes/ActiveRecord/FixtureSet.html)를
참고하세요.

## 감상

### Simple is Best

사용법이 굉장히 심플합니다. API 문서만 한번 슥 읽어보면 그대로 쓸 수 있을
정도입니다. `factory_girl`에서 `build`, `create`정도만 하던
사람이라면 넘어와도 큰 문제가 없겠다 싶은 수준.

### I'm a writer...!!!

픽스쳐 데이터 만드는게 재밌습니다. 페르소나를 만들고 작업하다보면 설정을 짜게
되고, 스토리를 만들고, 정신차리면 테스트를 짜는 게 아니라 소설을 쓰고 있을
뿐이고...

### Speed is a justice XD

무엇보다 빠릅니다. 테스트의 전후 DB 청소를 트랜잭션에 맡기고 있으니 빠른 건
당연하게 보입니다. 실제로 적용해본 결과는 이렇습니다.

```
(i343 ✓)

# Running:

.............................................................................................................

Fabulous run in 1.162851s, 93.7351 runs/s, 147.9123 assertions/s.

109 runs, 172 assertions, 0 failures, 0 errors, 0 skips

(master ✓)
# Running:

................................................................................................................

Fabulous run in 2.123136s, 52.7522 runs/s, 78.6572 assertions/s.

112 runs, 167 assertions, 0 failures, 0 errors, 0 skips
```

테스트 숫자가 적고 db를 사용하는 테스트는 저 중에서 절반정도입니다만, 그럼에도
불구하고 속도가 두 배정도 빨라진 것을 확인할 수 있습니다. 워낙 테스트셋이
작아서 그렇지, 20분짜리 테스트가 10분이 되었다고 생각하면 어떠신가요?

### No hook, No validation

위처럼 빠르게 동작하는 이유에는 몇가지가 있는데, 그 중에서 하나는 훅이나 검증을
전혀 사용하지 않고 픽스쳐에 정의된 사양을 그대로 데이터베이스에 올리고 있다는
부분입니다. 이 때문에 발생하는 귀찮음도 있는데요, 예를 들어, 성능 문제로 다른
데이터를 `before_create`에서 캐싱하고 있었다면 어떨까요? 당연하지만 우리의
픽스쳐는 전혀 고려해주지 않기 때문에, 개발자가 직접 다루어주어야 할 필요가
있습니다.

### No `build`, no `attributes_for`

픽스쳐는 데이터베이스에 이러한 데이터가 들어있다는 상태를 정의하므로
`factory_girls`에서 사용하는 `build`나 `attributes_for`등의
메소드는 직접 구현하거나, 포기해야합니다.
`use_instantiated_fixtures`를 사용하면 못할것도 없습니다만, 이 때에는
반대로 `create`의 사용이 불편해지게 됩니다.

### Auto-generated ID

자동 생성되는 primary key는 임의의 값을 사용하므로, 순서를 보장해주지
않습니다. 예를 들어, 특정 키 이후의 데이터 5개, 라는 동작을 테스트하려면
id를 손으로 지정해줘야 할 필요가 있습니다.

### 지금까지 네가 만든 픽스쳐의 숫자를 기억하고 있나?

테스트를 작성할 때 픽스쳐의 현재 상태를 의식해가면서 작성해야합니다. 예를 들어
`ItemsController`에서 `create` 액션이 정상적으로 데이터를 저장하는지
확인하는 경우를 보죠.

```ruby
def test_create_save_1_item_with_count
  post :create, params: {
    item: { contents: "item" }
  }, format: :json

  assert_equal 1, Item.count
end

def test_create_save_1_item_with_difference
  assert_difference "Item.count" do
    post :create, params: {
      item: { contents: "item" }
    }, format: :json
  end
end
```

`factory_girl`을 사용하는 경우, 전자든 후자든 별 문제가 없습니다.
왜냐하면 "깨끗한" DB에서 테스트를 수행하기 때문에 테스트 전후의 DB 정리가 잘
이루어지고 있다면 이 테스트는 언제나 독립적으로 동작하기 때문입니다. 반면
픽스처를 사용하는 경우라면 어떨까요? 새로운 `item` 픽스쳐가 하나 추가되는
순간 첫 번째 테스트는 소리없이 깨지게 됩니다. 반면 두 번째 테스트라면
`count`가 1 증가했는지를 확인하고 있으므로, 몇 개의 픽스쳐가 추가되더라도
문제없이 동작하게 됩니다.
이는 픽스쳐가 모든 테스트에서 공유되는 "초기 상태"라는 특징에서 기인하며, 이
때문에 덜 깨지는 테스트를 작성하기 위해 좀 더 고민할 필요가 있습니다.

## Conclusion

여기까지 살펴보면 픽스쳐는 사용법이 간단한 만큼, DRY하고 유연한 ~~그리고
느린~~ `factory_girl`보다는 확실히 불편합니다. 하지만 이점도 충분히
제공해준다고 생각합니다. 좀 더 심플한 구조, 많이 빠른 속도. 이 정도면
사용을 고려해볼만한 이유로는 충분하지 않을까요? XD

- ps1: 저는 앞으로 특별한 이유가 없으면 가급적 픽스쳐를 써볼까 하고 있습니다.
- ps2: 픽스쳐는 적당히 가지고 놉시다. 설정놀음을 시작하면 끝이 안납니다OTL

## Reference

- [API Document](http://api.rubyonrails.org/classes/ActiveRecord/FixtureSet.html)
- [Rails 테스트 가이드 - 픽스쳐](http://guides.rorlab.org/testing.html#픽스쳐)
