---
layout: post
title: "Elixir Function How-to"
date: 2016-06-07 20:43:09 +0900
categories:
---

언어를 너무 날로[..] 배우다보니, 꽤 다양해 보이는 함수 선언 방식을 한 번 정리해야 하지 않을까 싶었습니다. 그런 의미로 간단하게 정리.

## 기본적인 함수 선언

### Basic

```elixir
def greet do
  IO.puts "Hello"
end
```

Ruby에서 Elixir로 넘어오면서 제일 적응이 힘들었던 부분은 함수 선언 뒤로 `do-end` 블록이 온다는 점입니다.

### Inline

```elixir
def greet, do: IO.puts "Hello"
```

짧게 쓰라고, 대놓고 `do:`를 통해서 작성할 수 있게 해줍니다. 쉼표(,)의 위치에 주의합시다.

### With guard

```elixir
def greet when is_sleep == false, do: IO.puts "Good Morning"
```

선언 뒤에 `when`을 통해서 작성해주면 됩니다. `do:`와 다르게 쉼표가 없습니다. Core 함수 + 기본 연산을 사용한 평가만 가능합니다. 다시말해 새롭게 정의한 함수는 사용할 수 없습니다. 인자로 넘어온 값은 사용할 수 있으니, 미리 계산해서 넘겨주거나 다른 방법을 찾아야겠죠.

## Lambda

람다 등을 통해서 함수를 만들어서 사용하는 경우의 주의점. 함수 호출시 함수의 이름과 `()` 사이에 반드시 마침표를 끼워넣어야 합니다.

### Basic

```elixir
greet = fn ->
  IO.puts "Hello"
end
greet.()
```

### Short expression

```elixir
greet = &(IO.puts &1)
greet.("Hello")
```

람다를 짧게 줄여 쓸 수 있습니다. `&()`로 함수 컨텍스트를 `&1`로는 호출시에 넘겨준 첫번째 인자를 가져올 수 있습니다. 패턴매칭에서 사용하는 그것과 동일하다고 생각하시면 됩니다. 이렇게 보면 별로 유용하지 않지만, Enum을 통한 연산을 할 때에는 참으로 편리하게 사용할 수 있죠.


## 마무리

막상 정리하고 보니 별 것 없는거 같은데, 왜이리 혼란스러웠는가... 싶습니다.
