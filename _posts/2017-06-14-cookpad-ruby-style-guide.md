---
layout: post
title: "Cookpad Ruby style guide"
date: 2017-06-14 10:54:24 +0900
categories:
---

[Ruby style guide](https://github.com/bbatsov/ruby-style-guide)와는 다른 부분이
보여서 읽고 신경쓰이는 부분을 정리해보았습니다. 전체는
[여기](https://github.com/cookpad/styleguide)에서 확인하실 수 있습니다.

## 긴 메소드 체인의 마지막 부분이 블럭인 경우

이런 경우에는 앞 부분과 뒷 부분을 나누는 스타일을 권하고 있습니다.

```ruby
# good
posts = Post.joins(:user).
  merge(User.paid).
  where(created_at: target_date)
posts.each do |post|
  next if stuff_ids.include?(post.user_id)
  comment_count += post.comments.size
end

# bad
posts = Post.joins(:user).
  merge(User.paid).
  where(created_at: target_date).each do |post|
    next if stuff_ids.include?(post.user_id)
    comment_count += post.comments.size
  end
```

체이닝중 문제가 생기는 경우 이런 형태로 고쳐서 코딩하는 경우는 꽤 있는데,
이대로 쓰는건 신선하네요. 유지보수 측면해서는 유용하지 않을까 싶기도 하고.
개인적으로는 다음처럼 체인 자체를 끌어내리는 쪽을 선호합니다만...

```ruby
posts =
  Post.joins(:user).
    merge(User.paid).
    where(created_at: target_date).each do |post|
      next if stuff_ids.include?(post.user_id)
      comment_count += post.comments.size
    end
```

## 1행 최대 글자수는 128

80은 확실히 모자라겠구나, 싶긴한데 128이라는 미묘한 숫자는 어디서 나왔는지
궁금합니다[..]
이 길이면 새 레일즈 프로젝트에서 행 길이로 문제가 되는 경우는 없다는 장점은
있겠구나 싶네요.

## 숫자

숫자 관련이 의외로 자잘한 규칙이 많은데, 16진법이랑 분수는 어디서 쓰고 있는지...

- 10진수라면 3글자마다 언더스코어로 구분하기
  - `1_000_000.001_123`
- 2/16진수는 4글자마다 언더스코어로 구분하기
  - `0xABCD_1234`
- 16진법의 숫자는 한 파일내에서는 대/소문자를 한 쪽으로 통일하기
- 분수에 `r` 접미사 사용하기 (>= 2.1)
  - `1/2r #=> (1/2)`
- `r`를 사용할 수 없는 경우는 `Integer#quo`를 사용하기
  - `1.quo(2) #=> (1/2)`
- 복소수는 `i`, `ri` 접미사 사용하기 (>= 2.1)
  - 1 + 2i #=> (1+2i)

## 문자열

작은 따옴표/큰 따옴표 중 하나를 사용한다고 룰을 정하지 않는 부분이 독특하구나
싶습니다. 이스케이프 기호를 최소화하기 위한 전략의 일부인건지,
그냥 자유로운건지...?

- 빈 문자열은 `''`을 쓰기
- 문자열에 이스케이프 기호를 최소화할 수 있는 구분자를 사용할 것
- `%`로 문자리터럴을 만드는 경우, 괄호(`({[`)를 우선해서 사용할 것
- 유니코드 문자를 쓰는 경우 `\u`를 사용할 것. (>= 1.9)
- 루프(`while`, `until`, `for` 등의 이터레이터) 내에서 문자열 리터럴을 사용하지 말 것.
  - 메모리 누수 대책...?
- `String#+` 대신 전개식을 사용할 것.
- 문자열 `+=` 대신 `String#<<`나 `String#concat`을 사용할 것.
  - [여기](https://github.com/JuanitoFatas/fast-ruby#string) 보면 그렇게 압도적으로 빠른거 같진 않은데 말이죠... ;ㅅ;

## 정규식

- x 옵션을 적극적으로 사용할 것

```ruby
float_pat = /\A
  [[:digit:]]+ # 1 or more digits before the decimal point
  (\.          # Decimal point
      [[:digit:]]+ # 1 or more digits after the decimal point
  )? # The decimal point and following digits are optional
\Z/x
```

## 배열

- 여러 줄로 작성하는 경우 `[`, `]`를 별도로, 내부를 한번 들여쓸 것
- 마지막 요소 뒤에도 `,`를 작성할 것
- 빈 배열은 `Array.new` 대신 `[]`
- `[obj] * n` 대신 `Array.new(n, obj)`
- `Range#to_a` 대신 `[*range]`

## 해시

- 여러 줄로 작성하는 경우 `{`, `}`를 별도로, 내부를 한번 들여쓸 것
- 마지막 요소 뒤에도 `,`를 작성할 것

## 수식

- 삼항연산자는 중첩해서 사용하지 말 것
- 삼항연산자는 여러 줄로 작성하지 말 것

## 대입

- 다중 대입은 값을 교환하는 경우, 함수 호출의 결과 대입에만 사용할 것
- 조건식에서는 대입하지 말 것

## 조건

- `unless`, `until`에서 `||` 조건을 사용하지 않기

## 함수 호출

- 괄호 생략이 허용된 경우를 제외하면 생략해서는 안됨
  - 허용된 경우를 구체적으로 적어줬으면 좋겠는데, 그런게 없습니다. 루비 규칙대로라면 아예 불가능한 경우는 문법 에러말고는 없지 않나 싶고...
- 인수 없는 메소드, DSL-like 메소드는 괄호를 생략할 것, 단 경고가 나오는 경우에는 생략하지 않아도 됨
- 메소드 호출을 중첩하는 경우, 다른 규칙에 걸리지 않는다면 가장 바깥 메소드의 괄호는 생략할 수 있음
- 블럭은 `do`/`end`를 사용할 것. 단 반환값을 사용하는 경우에는 중괄호를 사용할 것
- 블럭이 인라인인 경우에도 중괄호를 사용할 것

```ruby
# good
puts [1, 2, 3].map {|i|
  i * i
}

# bad
puts [1, 2, 3].map do |i|
  i * i
end

# good
[1, 2, 3].map {|n|
  n * n
}.each {|n|
  puts Math.sqrt(n)
}

# bad
[1, 2, 3].map do |n|
  n * n
end.each do |n|
  puts Math.sqrt(n)
end
```

## 모듈/클래스

- `alias` 대신 `alias_method`
- 메소드 단위로 공개범위를 지정하는 경우, 메소드 정의 바로 밑에서 처리할 것

```ruby
class Foo
  # good
  def foo
  end
  private :foo

  # bad
  def foo
  end

  private :foo
end
```

- 공개 메소드/접근자는 마크다운 형식으로 문서 주석을 작성할 것
- 문서 형식은 YARD를 권장

## 메소드 정의

- 인수가 없는 경우 괄호를 생략할 것
- 넘겨받은 인수를 변경하지 말 것

## 감상

큰 틀은 Ruby Style Guide와 다를 것이 없습니다만, 중간중간 스타일 가이드라기보단,
더 위의 레이어라고 생각되는 부분이 몇 개 있는게 특이했습니다. 예를 들어, 하나의
클래스에 여러 역할을 주지 말 것, 같은 것들인데요. 독자 레벨을 신입 개발자로
잡아서 그런건지, 그만큼 강조를 하고 싶은 건지는 판단이 어렵네요. 기회가 되면
다른 회사의 스타일 가이드를 읽어보는 것도 재미있을 것 같습니다.
