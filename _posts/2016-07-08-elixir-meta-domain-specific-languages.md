---
layout: post
title: "Elixir - Meta: Domain Specific Languages"
date: 2016-07-08 19:35:15 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 있습니다. 원문은 최하단의 링크를 참고하세요.

## 들어가기 전에

[도메인 특화 언어 (DSL)](https://en.wikipedia.org/wiki/Domain-specific_language)는 개발자들에게 특정 도메인에 맞춰서 애플리케이션을 만들 수 있도록 해줍니다. DSL을 사용하기 위해서 반드시 매크로가 필요하지는 않습니다. 모듈에서 정의하는 모든 데이터 구조와 모든 함수가 이미 DSL의 일부입니다.

예를 들어, DSL로 데이터 검증을 제공하는 검증 모듈을 구현한다고 상상해봅시다. 이는 데이터 구조, 함수, 매크로 등을 사용해서 구현될 수 있습니다. 각각 어떻게 구현할 수 있는지 살펴보시죠.

```elixir
# 1. data structures
import Validator
validate user, name: [length: 1..100],
               email: [matches: ~r/@/]

# 2. functions
import Validator
user
|> validate_length(:name, 1..100)
|> validate_matches(:email, ~r/@/)

# 3. macros + modules
defmodule MyValidator do
  use Validator
  validate_length :name, 1..100
  validate_matches :email, ~r/@/
end

MyValidator.validate(user)
```

위에서 볼 수 있는 모든 접근법 중에서 가장 유연한 것은 첫 번째 입니다. 만약 도메인 규칙을 데이터 구조로 변환할 수 있다면, Elixir의 표준 라이브러리는 다른 데이터 타입을 처리하기 위한 함수들로 가득하므로 가장 구현하기 쉽고, 조합하기도 쉽습니다.

두 번째 접근 방법은 함수 호출을 사용하고 있으며, 복잡한 API(예를 들어, 많은 옵션을 넘겨야 하는 경우)에 잘 어울리며, 파이프 연산자 덕분에 읽기도 좋습니다.

세 번째 접근 방법은 매크로를 사용하고 있으며, 가장 복잡합니다. 구현할 때에 가장 많은 코드가 필요하며, (다른 간단한 함수들을 테스트한 것에 비하면) 테스트가 어렵고, 테스트 비용이 비쌉니다. 그리고 모듈 내부에 모든 검증을 구현해야 하므로 사용자가 라이브러리를 사용하는 방법을 제한합니다.

이를 설명하기 위해, 어떤 속성이 주어진 조건이 만족하는 경우에만 유효하길 원한다고 해보죠. 첫 번째 방법으로는 요구되는 데이터구조대로 만들면 간단하게 만들 수 있습니다. 두 번째 해결책은 함수를 호출하기 전에 조건문을 사용할 수 있습니다. 하지만 마지막 경우는 DSL이 확장되지 않으면 그러한 접근이 불가능합니다.

말하자면,

    data > functions > macros

그렇지만 여전히 모듈과 매크로로 DSL을 만드는 경우가 유용한 경우도 있습니다. 'Getting Started'에서 이미 데이터 구조와 함수 정의에 대해서는 알아보았으므로, 이 장에서는 매크로와 모듈 속성을 사용해서 좀 더 복잡한 DSL을 다뤄보죠.

## 테스트 케이스 만들기

이 장의 목표는 다음과 같은 코드를 작성할 수 있게 해주는 `TestCase`라는 모듈을 만드는 것입니다.

```elixir
defmodule MyTest do
  use TestCase

  test "arithmetic operations" do
    4 = 2 + 2
  end

  test "list operations" do
    [1, 2, 3] = [1, 2] ++ [3]
  end
end

MyTest.run
```

위의 예제에서는 `TestCase`를 사용하여 `test` 매크로를 사용해 테스트를 작성하고, `run`이라고 정의된 함수를 사용하여 모든 테스트를 자동으로 실행합니다. 첫 번째 구현에서는 매치 연산자(`=`)를 통해 테스트를 평가하도록 하겠습니다.

## `test` 매크로

간단하게 정의된 모듈을 만들고, 사용되면 `test` 매크로를 주입해봅시다.

```elixir
defmodule TestCase do
  # Callback invoked by `use`.
  #
  # For now it simply returns a quoted expression that
  # imports the module itself into the user code.
  @doc false
  defmacro __using__(_opts) do
    quote do
      import TestCase
    end
  end

  @doc """
  Defines a test case with the given description.

  ## Examples

      test "arithmetic operations" do
        4 = 2 + 2
      end

  """
  defmacro test(description, do: block) do
    function_name = String.to_atom("test " <> description)
    quote do
      def unquote(function_name)(), do: unquote(block)
    end
  end
end
```

`tests.exs`라는 파일에 `TestCase`를 정의했다고 가정하고, `iex tests.exs`를 실행하여 테스트를 정의해봅시다.

```iex
iex> defmodule MyTest do
...>   use TestCase
...>
...>   test "hello" do
...>     "hello" = "world"
...>   end
...> end
```

이 시점에서는 테스트를 실행할 방법이 없습니다만, 이 뒤에서는 "test hello"라는 이름의 함수가 정의되었다는 사실은 알고 있습니다. 함수를 호출하면 이는 실패합니다.

```iex
iex> MyTest."test hello"()
** (MatchError) no match of right hand side value: "world"
```

## 속성과 함께 정보를 저장하기

`TestCase` 구현을 마무리 지으려면, 정의된 모든 테스트 케이스에 접근할 수 있어야 합니다. 이를 위해서 주어진 모듈에 있는 모든 함수 목록을 반환하는 `__MODULE__.__info__(:functions)`를 사용하여 테스트 목록을 실행 시간에 가져올 수도 있습니다. 하지만 테스트 이름뿐만 아니라 추가 정보를 저장하길 원한다면 이보다는 좀 더 유연한 방법이 필요합니다.

이전에 모듈 속성에 관해서 이야기할 때, 그것들을 어떻게 임시 저장소로 사용할 수 있는지에 대해서도 언급했었습니다. 이번 절에서는 바로 그 속성을 사용해보도록 하겠습니다.

`__using__/1` 구현에서 `@tests`라는 모듈 속성을 빈 리스트로 초기화하고, 거기에 정의된 테스트를 저장하여 `run` 함수에서 가져다 사용할 수 있도록 해보겠습니다.

이를 반영한 `TestCase` 모듈의 코드는 다음과 같습니다:

```elixir
defmodule TestCase do
  @doc false
  defmacro __using__(_opts) do
    quote do
      import TestCase

      # Initialize @tests to an empty list
      @tests []

      # Invoke TestCase.__before_compile__/1 before the module is compiled
      @before_compile TestCase
    end
  end

  @doc """
  Defines a test case with the given description.

  ## Examples

      test "arithmetic operations" do
        4 = 2 + 2
      end

  """
  defmacro test(description, do: block) do
    function_name = String.to_atom("test " <> description)
    quote do
      # Prepend the newly defined test to the list of tests
      @tests [unquote(function_name) | @tests]
      def unquote(function_name)(), do: unquote(block)
    end
  end

  # This will be invoked right before the target module is compiled
  # giving us the perfect opportunity to inject the `run/0` function
  @doc false
  defmacro __before_compile__(env) do
    quote do
      def run do
        Enum.each @tests, fn name ->
          IO.puts "Running #{name}"
          apply(__MODULE__, name, [])
        end
      end
    end
  end
end
```

새 IEx 세션을 시작하고, 새 테스트를 정의한 뒤에 실행해봅시다.

```iex
iex> defmodule MyTest do
...>   use TestCase
...>
...>   test "hello" do
...>     "hello" = "world"
...>   end
...> end
iex> MyTest.run
Running test hello
** (MatchError) no match of right hand side value: "world"
```

문제가 있는 부분은 일단 넘어가고, 이게 바로 Elixir에서 특정 도메인을 위한 모듈을 만드는 방법에 대한 개략적인 설명입니다. 매크로는 내부 표현식을 반환하여 호출한 쪽에서 실행할 수 있게끔 해주며, 이것들을 사용해서 코드를 변경하거나 타겟 모듈의 모델 속성을 통해서 관련된 값을 저장할 수도 있습니다. 마지막으로 `@before_compile`와 같은 콜백들은 정의가 완료된 시점에 모듈에 코드를 주입할 수 있게 도와줍니다.

`@before_compile` 이외에도 `@on_definition`이나 `@after_compile`과 같은 유용한 모델 속성들이 있으므로 [`Module` 모듈 문서](http://elixir-lang.org/docs/stable/elixir/Module.html)를 읽어보세요. 그리고 [`Macro` 모듈 문서](http://elixir-lang.org/docs/stable/elixir/Macro.html)와 [`Macro.Env` 문서](http://elixir-lang.org/docs/stable/elixir/Macro.Env.html)에서 매크로와 컴파일 환경에 대한 유용한 정보를 확인할 수 있습니다.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Macros](http://elixir-lang.org/getting-started/meta/domain-specific-languages.html)
