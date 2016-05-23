---
layout: post
title: "Mastering ruby blocks in less than 5 minutes (번역)"
date: 2016-03-08 07:29:53 +0900
categories:
---

### 들어가기 전에

이 글은 Cezar Halmagean의 [Mastering ruby blocks in less than 5 minutes](http://mixandgo.com/blog/mastering-ruby-blocks-in-less-than-5-minutes)를 번역한 글입니다. 오타, 오역이 차고 넘칠 수 있습니다.

# 루비 블럭 5분 안에 마스터하기

블럭은 루비를 살펴볼 때 가장 강력한 기능 중 하나입니다. 사실 저도 블럭이 어떻게 동작하고, 실제 상황에서 어떻게 유용한지 이해하는데에 시간이 좀 걸렸습니다.

첫번째로 블럭을 이해하기 어렵게 만드는 `yield`가 있습니다. 이 글에서는 이에 대한 개념과 몇몇 예제를 통해서 루비의 블럭에 대해서 잘 이해할 수 있도록 설명해볼까 합니다.

![Mastering Ruby Blocks](/images/20160308/mastering_ruby_blocks.jpg)

## 이 글을 읽으면 배울 수 있는 것들

 * 기본: 루비 블럭이 뭐죠?
 * yield가 동작하는 방식
 * 메소드에 블럭 넘겨주기
 * Yield도 인자를 받습니다
 * `&block`이 뭐죠?
 * 반환값
 * `.map(&:something)`은 어떻게 동작하나요?
 * 이터레이터와 만드는 방법
 * 블럭을 통해서 객체에 초기값 넘겨주기
 * 루비 블럭 예제

## 기본: 루비 블럭이 뭐죠?

기본적으로 블럭은 그냥 `do`와 `end` 사이에 놓여있는 코드 덩어리입니다. 그게 끝이에요. "그럼 마법은 도대체 어디에 있죠?" 라고 물을 수도 있을 겁니다. 거기에 대해서는 금방 설명하겠지만, 우선 해야할 게 있습니다.

블럭은 다음과 같은 두가지 방식으로 작성할 수 있습니다.

 * 여러 줄, `do`와 `end`를 사용해서,
 * 한줄, `{`와 `}`를 사용해서.

(역주: 후자는 여러 줄로도 사용할 수 있는 것이 맞습니다.)

두 가지 모두 동일한 동작을 하며, 무엇을 사용하는가는 주관적인 선택입니다. [일반적인 스타일 가이드](https://github.com/bbatsov/ruby-style-guide#single-line-blocks)에서는 여러 줄일때 `do`-`end`를, 그렇지 않은 경우에는 가독성을 위하여 `{`와 `}`를 권장합니다.

`do`-`end`을 사용하는 예제를 봅시다:

```ruby
[1, 2, 3].each do |n|
  puts "Number #{n}"
end
```

이는 여러줄 블럭(multi-line block)이라고 부르며, 이는 한줄의 코드를 포함해서가 아니라 한줄이 아니기 때문입니다. 같은 예제를 한줄 블럭을 사용해서 다음과 같이 작성할 수 있습니다.

```ruby
[1, 2, 3].each { |n| puts "Number #{n}" }
```

둘 다 숫자 1,2,3을 순서대로 출력합니다. 파이프 사이에 있는 작은 `n`은 **블럭 인자**이며, 이 변수는 배열에서 각각 차례가 된 숫자들을 가져오게 됩니다. 그러므로 첫번째 반복에서 `n`의 값은 1이 되며, 두번째에서는 2, 그 다음에는 3이 될 겁니다.

```ruby
Number 1
Number 2
Number 3
 => [1, 2, 3]
```

## yield가 동작하는 방식

드디어 문제아가 왔습니다. 그는 루비 블럭에 대한 모든 혼란과 마법을 책임지죠. 저는 대부분의 혼란이 어떻게 블럭을 호출하는지, 그리고 어떻게 파라미터를 넘기는지에서 온다고 생각합니다. 이번 장에서는 두 가지 시나리오를 살펴볼 겁니다.

```ruby
def my_method
  puts "reached the top"
  yield
  puts "reached the bottom"
end

my_method do
  puts "reached yield"
end
```

```ruby
reached the top
reached yield
reached the bottom
 => nil
```

`my_method`가 실행되는 도중 `yield` 호출을 만나면, 넘겨졌던 블럭 내에 있는 코드가 실행됩니다. 그리고 블럭에 있는 코드가 전부 실행되면 `my_method`의 나머지 부분이 실행됩나다.

![Ruby block execution](/images/20160308/ruby_block_flow.png)

## 메소드에 블럭 넘겨주기

메소드는 블럭을 받기 위해서 별도로 처리해야할 작업이 전혀 없습니다. 그냥 메소드를 호출할 때에 블럭을 넘겨주면 됩니다. 물론 실행하기 위해서는 `yield`를 호출해야하며, 그렇지 않은 경우, 넘겨진 블럭은 무시됩니다.

반면, 메소드 내부에서 `yield`를 사용한 경우, 블럭 인수는 필수가 되며, 만약 블럭을 넘겨받지 않았을 경우, 에러를 던집니다.

만약 블럭을 조건부로 사용하고 싶다면, `block_given?` 메소드를 사용해서 블럭을 넘겨받았는지, 아닌지를 구분하면 됩니다.

## Yield도 인자를 받습니다

`yield`에 넘겨지는 인수는 블럭에서의 매개변수가 됩니다. 그러므로 블럭에서는 원래 메소드에서 넘긴 값을 매개변수로 사용할 수 있습니다. 이러한 인수들은 `yield`가 살아있는 동안 변수로서 사용할 수 있습니다.

넘긴 값을 순서대로 블럭에서 인수로 사용할 수 있기 때문에 순서는 중요합니다.

![Ruby block arguments](/images/20160308/ruby_block_arguments.png)

하나 기억해두어야 하는 점은 블럭에 있는 매게 변수들은 블럭 내부에서만 살아 있다는 점입니다.

## `&block`이 뭐죠?


루비 코드에서 `&block`를 여기저기에서 보셨을 겁니다. 이는 메소드에 블럭에 대한 참조를 (지역 변수 대신) 넘기는 방법인데요. 사실 루비는 어떤 객체라도 메소드에 넘길 수 있습니다. 메소드는 넘겨진 객체가 블럭이라면 곧바로 실행하며, 그렇지 않다면 `to_proc` 메소드를 호출해서 블럭으로 변환하려고 시도합니다.

그리고 여기에서 `block`은 그저 참조의 이름이라는 것을 기억하세요. 납득할 수 있는 어떤 이름이라도 좋습니다.

```ruby
def my_method(&block)
  puts block
  block.call
end

my_method { puts "Hello!" }
```

```ruby
#<Proc:0x0000010124e5a8@tmp/example.rb:6>
Hello!
```

위의 예제에서 볼 수 있듯이 `my_method` 내부의 변수 `block`은 블럭에 대한 참조값으로, `call` 메소드를 호출할 수 있습니다. 블럭에 `call`을 호출하면 `yield`와 동일한 동작을 수행하며, 사람들에 따라서는 가독성이 좋다는 이유로 `yield`보다 `block.call`을 선호하기도 합니다.

## 반환값

`yield`는 (블럭 내에서) 마지막으로 평가된 표현식을 반환합니다. 다시 말하자면, `yield`의 반환값은 블럭의 반환값이라는 의미입니다.

```ruby
def my_method
  value = yield
  puts "value is: #{value}"
end

my_method do
  2
end
value is 2
=> nil
```

## `.map(&:something)`은 어떻게 동작하나요?

Rails에서 코드를 작성하다 보면, `.map(&:capitalize)`와 같은 코드를 작성하곤 하실텐데요. 이 코드는 `.map { |title| title.capitalize }`를 짧게 축약한 것입니다.

하지만 이 코드는 어떻게 동작하는 걸까요?

Symbol 클래스는 전자의 코드를 후자로 변환해주는 `to_proc`을 구현하고 있기 때문입니다. 멋지지 않나요?

## 이터레이터와 만드는 방법

`yield`는 메소드 내부에서 원하는 만큼 호출 할 수 있습니다. 이게 바로 이터레이터가 동작하는 방식인데요, 배열에서 반복적으로 `yield`를 호출해서 루비에 있는 내장 이터레이터들의 동작을 흉내내어 봅시다.

루비의 map 메소드와 유사한 코드를 어떻게 작성하는지 확인해보세요.

```ruby
def my_map(array)
  new_array = []

  for element in array
    new_array.push yield element
  end

  new_array
end

my_map([1, 2, 3]) do |number|
  number * 2
end
```

```ruby
2
4
6
```

## 블럭을 통해서 객체에 초기값 넘겨주기

루비의 블럭을 사용해서 보여줄 수 있는 멋진 패턴 중 하나는 새 객체에 초기값을 넘겨주는 것입니다. 한번이라도 루비 젬에서 직접 `.gemspec`을 열어보신 적이 있다면 이런 패턴을 보신적이 있을 겁니다.

방법은 간단합니다. 생성자에 `yield(self)`를 추가하세요. `initialize` 메소드에서의 `self`는 초기화된 객체 자신이 됩니다.

```ruby
class Car
  attr_accessor :color, :doors

  def initialize
    yield(self)
  end
end

car = Car.new do |c|
  c.color = "Red"
  c.doors = 4
end

puts "My car's color is #{car.color} and it's got #{car.doors} doors."
```

```ruby
My car's color is Red and it's got 4 doors.
```

## 루비 블럭 예제

이 예제들은 최근에 많이 쓰이고 있는 것들이며, 여기에서 블럭을 사용할 수 있는 실용적인 (또는 실제로 있을 법한) 시나리오를 찾아봅시다.

### html 태그에 있는 문자열 감싸기

한 덩어리의 동적인 코드를 정적인 코드로 감싸야 하는 경우, 블럭은 이 상황에서 완벽한 후보가 될 수 있습니다. 우선, 어떤 문자열을 사용해서 html 태그를 생성하고 싶다고 가정해봅시다. 문자열은 동적인 부분(어떤 내용물이 넘어오게 될지는 이 시점에서 알 수 없습니다)에 해당될 것이며, html 태그가 변하지 않는 정적인 부분이 될 것입니다.

```ruby
def wrap_in_h1
  "<h1>#{yield}</h1>"
end

wrap_in_h1 { "Here's my heading" }

# => "<h1>Here's my heading</h1>"

wrap_in_h1 { "Ha" * 3 }

# => "<h1>HaHaHa</h1>"
```

어떤 코드를 재활용하지만 조금 다르게 사용하고 싶은 경우에, 블럭이 얼마나 작업을 쉽게 만드는지 보세요. 이번엔 문자열을 html에 삽입한 다음 그 태그 문자열 자체를 가지고 무언가를 해보고 싶다고 해보죠.

```ruby
def wrap_in_tags(tag, text)
  html = "<#{tag}>#{text}</#{tag}>"
  yield html
end

wrap_in_tags("title", "Hello") { |html| Mailer.send(html) }
wrap_in_tags("title", "Hello") { |html| Page.create(:body => html) }
```

첫번째 실행에서는 `<title>Hello</title>`라는 문자열을 메일을 통해서 전송했으며, 두번째 실행에서는 `Page`라는 레코드를 생성합니다. 두 경우 모두 같은 코드를 사용하지만 다른 작업을 하고 있죠.

### 노트 만들기

이번에는 데이터베이스에 아이디어들을 쉽게 저장하는 무언가를 만들고 싶다고 해보죠. 이를 위해서는 문자열을 관리하고, 데이터베이스에 대한 커넥션을 열고, 닫아야 합니다. 이상적으로는 `Note.create { "Nice day today" }`라고만 호출하면 데이터베이스 커넥션에 관한 처리를 잊어버려도 되게끔 만들고 싶네요. 그럼 해봅시다.

```ruby
class Note
  attr_accessor :note

  def initialize(note=nil)
    @note = note
    puts "@note is #{@note}"
  end

  def self.create
    self.connect
    note = new(yield)
    note.write
    self.disconnect
  end

  def write
    puts "Writing \"#{@note}\" to the database."
  end

private

  def self.connect
    puts "Connecting to the database..."
  end

  def self.disconnect
    puts "Disconnecting from the database..."
  end
end
```

```ruby
Note.create { "Foo" }
Connecting to the database...
@note is Foo
Writing "Foo" to the database.
Disconnecting from the database...
```

데이터베이스에 연결, 쓰기, 끊기에 대한 구현에 대해서는 이 글에서 설명하고자 하는 내용과는 거리가 멀기에 생략했습니다.

### 배열에서 있는 나눌 수 있는 원소 찾기

"실제 상황을 상정한 시나리오"에서는 거리가 멉니다만, 어쨌든 마지막 예제를 들겠습니다. 배열에 있는 모든 원소중에서 3으로 (또는 선택한 어떤 숫자로) 나눌수 있는 것들만 가져오고 싶다고 해봅시다. 이런 경우에는 어떻게 작성하면 좋을까요?

```ruby
class Fixnum
  def to_proc
    Proc.new do |obj, *args|
      obj % self == 0
    end
  end
end
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10].select(&3)
puts numbers
```

```ruby
3
6
9
```

## 결론

이제 블럭은 단순히 **코드 뭉치**임과 `yield`는 메소드의 특정 위치에 그 코드들을 주입하는 것임을 이해했을 겁니다. 이는 다시 말해서 **여러 개의 메소드를 작성하지 않는 방법**을 하나 더 배웠다는 의미이기도 하죠(코드 블럭을 사용하면 코드 중복을 줄일 수 있기 때문입니다).

해내셨습니다! 이 글을 다 읽었다면, 이제 루비 블럭을 다양하게 사용할 수 있는 방법을 찾아야 할 때입니다. 어떤 이유로든 여전히 혼란스럽거나, 설명에 부족한 부분이 있다면 덧글을 통해서 (원저자에게) 알려주세요.

루비 블럭에 대해서 새로 배운게 있다면 이 글(원문)을 공유해주세요!






