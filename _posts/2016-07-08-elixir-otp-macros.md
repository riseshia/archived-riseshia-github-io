---
layout: post
title: "Elixir - OTP: Macros"
date: 2016-07-08 07:42:12 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 있습니다. 원문은 최하단의 링크를 참고하세요.

## 들어가기 전에

Elixir가 매크로를 위한 안전한 환경을 제공하려고 노력한다고 하더라도, 매크로를 사용해서 깨끗한 코드를 작성할 책임은 개발자에게 있습니다. 매크로는 일반적인 Elixir 함수보다 작성하기 어려우며, 필요하지 않은 때 사용하는 것도 좋지 않은 스타일로 여겨집니다. 그러므로 매크로는 신중하게 사용하세요.

Elixir는 이미 데이터 구조와 함수들을 통해 평소의 코드를 간단하고 읽기 쉽게 만들 수 있도록 해주고 있습니다. 매크로는 마지막 선택지가 되어야 합니다. **명시적인 것이 묵시적인 것보다 낫다**는 점을 잊지 마세요. **간결한 코드보다는 명확한 코드가 좋습니다.**

## 매크로 만들기

Elixir의 매크로는 `defmacro/2`를 통해서 정의할 수 있습니다.

> 이 장에서는 IEx에서 코드를 실행하는 대신 파일을 사용할 것입니다. 왜냐하면 코드 예제들은 꽤 기므로 IEx에서 이것들을 전부 타이핑하는 것은 그다지 생산적이지 않습니다. 예제들을 작성하고 `macros.exs`에 저장한 뒤, `elixir macros.exs`나 `iex macros.exs`로 실행해주세요.

매크로가 어떻게 동작하는지 이해하기 위해서, `if`와는 반대의 동작을 하는 `unless`를 함수로, 그리고 매크로로 정의해봅시다.

```elixir
defmodule Unless do
  def fun_unless(clause, expression) do
    if(!clause, do: expression)
  end

  defmacro macro_unless(clause, expression) do
    quote do
      if(!unquote(clause), do: unquote(expression))
    end
  end
end
```

이 함수는 인수들을 넘겨받아 `if`에 그대로 넘겨주고 있습니다. 그렇지만 [이전 장](http://elixir-lang.org/getting-started/meta/quote-and-unquote.html)에서 배웠듯, 매크로는 내부 표현식을 받으므로, 인수를 그 안으로 주입한 뒤, 이를 다른 내부 표현식으로 반환합니다.

모듈 위에서 `iex`를 실행합시다.

```bash
$ iex macros.exs
```

그리고 다음을 실험해보세요.

```iex
iex> require Unless
iex> Unless.macro_unless true, IO.puts "this should never be printed"
nil
iex> Unless.fun_unless true, IO.puts "this should never be printed"
"this should never be printed"
nil
```

함수에서는 문장이 출력되었음에도 불구하고, 매크로에서는 문장이 출력되지 않았습니다. 이는 함수를 호출하기 전에 함수에 넘겨진 인수들이 평가되기 때문입니다. 하지만 매크로는 이 인수들을 평가하지 않습니다. 대신, 인자들은 내부 표현식으로 넘겨받고 이것들이 다른 내부 표현식으로 변환됩니다. 여기에서는 `unless` 매크로가 `if` 뒤에서 동작하게끔 재작성했습니다.

다시 말해, 이것을 호출하면,

```elixir
Unless.macro_unless true, IO.puts "this should never be printed"
```

`macro_unless` 매크로는 다음을 넘겨받습니다.

{% raw %}
```elixir
macro_unless(true, {{:., [], [{:aliases, [], [:IO]}, :puts]}, [], ["this should never be printed"]})
```
{% endraw %}

그리고 다음과 같은 내부 표현식을 반환합니다.

{% raw %}
```elixir
{:if, [], [
  {:!, [], [true]},
  {{:., [], [IO, :puts], [], ["this should never be printed"]}}]}
```
{% endraw %}

정말 이렇게 동작하는지, `Macro.expand_once/2`를 사용해서 확인할 수 있습니다.

```iex
iex> expr = quote do: Unless.macro_unless(true, IO.puts "this should never be printed")
iex> res  = Macro.expand_once(expr, __ENV__)
iex> IO.puts Macro.to_string(res)
if(!true) do
  IO.puts("this should never be printed")
end
:ok
```

`Macro.expand_once/2`는 내부 표현식을 받아서, 현재의 환경에 이를 전개합니다. 여기에서는 `Unless.macro_unless/2`를 전개하고 실행한 뒤, 결과를 반환합니다. 그러면 돌려받은 내부 표현식을 문자열로 변환해서 출력해볼 수 있을 것입니다(`__ENV__`에 대해서는 이 장의 뒤에서 다룹니다).

이것이 매크로의 모든 것입니다. 내부 표현식을 받아서 다른 형태로 가공합니다. 사실 Elixir의 `unless/2`도 매크로로 구현되어 있습니다.

```elixir
defmacro unless(clause, options) do
  quote do
    if(!unquote(clause), do: unquote(options))
  end
end
```

`unless/2`, `defmacro/2`, `def/2`, `defprotocol/2`와 같이 지금까지 가이드에서 보았던 순수 Elixir 표현들은 대부분이 매크로로 되어 있습니다. 이 말은 언어를 만들기 위해서 사용된 구조를 통해 개발자들이 일하고 있는 영역으로 언어를 확장할 수 있다는 의미입니다.

Elixir에서 제공되는 내장된 정의들을 덮어쓰는 등, 원하는 어떤 매크로나 함수도 정의할 수 있습니다. 유일한 예외는 Elixir로 구현되지 않은 Elixir의 특별한 형식들로 이들은 덮어쓸 수 없으며, [`Kernel.SpecialForms`에서 그 전체 목록](http://elixir-lang.org/docs/stable/elixir/Kernel.SpecialForms.html#summary)을 확인할 수 있습니다.

## 청결한 매크로(Macros hygiene)

Elixir의 매크로는 나중에 처리됩니다. 이는 매크로 내부에서는 문맥에 정의된 어떤 변수와도 충돌하지 않는다는 것을 보장합니다. 예를 들어,

```elixir
defmodule Hygiene do
  defmacro no_interference do
    quote do: a = 1
  end
end

defmodule HygieneTest do
  def go do
    require Hygiene
    a = 13
    Hygiene.no_interference
    a
  end
end

HygieneTest.go
# => 13
```

위 예제에서는 매크로가 `a = 1`이라는 코드를 주입했음에도 불구하고, `go` 함수의 `a`에는 영향을 주지 않았습니다. 만약 매크로가 명시적으로 실행 문맥에 영향을 주고 싶을 경우에는 `var!`를 사용하면 됩니다.

```elixir
defmodule Hygiene do
  defmacro interference do
    quote do: var!(a) = 1
  end
end

defmodule HygieneTest do
  def go do
    require Hygiene
    a = 13
    Hygiene.interference
    a
  end
end

HygieneTest.go
# => 1
```

변수는 Elixir가 각 변수가 어느 문맥에 존재하는지를 표시하고 있으므로 청결합니다. 예를 들어 모듈의 3번째 줄에서 정의된 `x`라는 변수에는 다음과 같이 표현됩니다.

```elixir
{:x, [line: 3], nil}
```

그러나 내부 표현식에서는 다음과 같이 표현됩니다.

```elixir
defmodule Sample do
  def quoted do
    quote do: x
  end
end

Sample.quoted #=> {:x, [line: 3], Sample}
```

내부 표현식에서 세 번째 원소로 `nil` 대신에, `Sample`이라는 아톰이 들어 있으며, 이는 `Sample` 모듈에 속해있다는 표시입니다. 그러므로 Elixir는 이 두 변수를 다른 문맥에서 왔다고 여기며, 거기에 맞게 다룹니다.

Elixir는 import나 alias에서도 비슷한 동작을 제공합니다. 이는 매크로가 전개된 상황에서의 문맥에 따라 동작하는 것이 아니라, 자신이 정의되어 있는 대로 동작할 수 있도록 보장합니다. 이 청결성은 `var!/2`나 `alias!/2`와 같은 매크로를 통해서 우회할 수 있습니다. 다만 사용자의 환경을 직접 변경할 수 있으므로 반드시 주의해야 합니다.

변수의 이름을 동적으로 생성할 수도 있습니다. 그런 경우에는 `Macro.var/2`를 사용해서 새 변수를 정의할 수도 있습니다:

```elixir
defmodule Sample do
  defmacro initialize_to_char_count(variables) do
    Enum.map variables, fn(name) ->
      var = Macro.var(name, nil)
      length = name |> Atom.to_string |> String.length
      quote do
        unquote(var) = unquote(length)
      end
    end
  end

  def run do
    initialize_to_char_count [:red, :green, :yellow]
    [red, green, yellow]
  end
end

> Sample.run #=> [3, 5, 6]
```

`Macro.var/2`의 두 번째 인수에 주의하세요. 이것은 문맥을 의미하며, 다음 절에서 설명할 청결 범위를 결정합니다.

## 환경

이 장의 앞에서 `Macro.expand_once/2`를 호출할 때에는 `__ENV__`라는 특별한 것을 사용했습니다.

`__ENV__`는 현재의 모듈, 파일, 실행 중인 줄 수, 현재 스코프에 정의되어 있는 모든 변수를 포함한 컴파일 환경같은 유용한 정보를 포함하는 `Macro.Env` 구조체의 인스턴스를 반환합니다. 이는 import, require로 불러온 정보도 포함됩니다.

```iex
iex> __ENV__.module
nil
iex> __ENV__.file
"iex"
iex> __ENV__.requires
[IEx.Helpers, Kernel, Kernel.Typespec]
iex> require Integer
nil
iex> __ENV__.requires
[IEx.Helpers, Integer, Kernel, Kernel.Typespec]
```

`Macro` 모듈의 많은 함수는 환경을 요구합니다. 이러한 함수들에 대한 설명은 [`Macro` 모듈 문서](http://elixir-lang.org/docs/stable/elixir/Macro.html)에서 볼 수 있으며, 컴파일 환경에 대한 정보는 [`Macro.Env`에 대한 문서](http://elixir-lang.org/docs/stable/elixir/Macro.Env.html)에서 볼 수 있습니다.

## 비공개 매크로

Elixir는 `defmacrop`를 통해 비공개 매크로도 지원하고 있습니다. 비공개 함수처럼 이런 매크로들은 자신들이 정의된 함수에서만 사용할 수 있으며, 컴파일 시점에서만 사용 가능합니다.

사용하기 전에 매크로는 정의하는 것은 무척 중요합니다. 호출하기 전에 매크로를 정의하지 못하면, 실행 시점에 매크로를 전개하지 못해 함수 호출로 처리되기 때문에 에러를 발생시킵니다.

```iex
iex> defmodule Sample do
...>  def four, do: two + two
...>  defmacrop two, do: 2
...> end
** (CompileError) iex:2: function two/0 undefined
```

## 확실히 동작하는 매크로 만들기

매크로는 강력한 구조이며, Elixir는 이들이 확실히 동작할 수 있게끔 많은 방법을 제공합니다.

* 매크로는 청결합니다: 기본적으로 매크로에서 정의되는 변수들은 외부 코드에 영향을 주지 않습니다. 나아가서 매크로 내부의 함수 호출과 alias는 실행 문맥에도 노출되지 않습니다.

* 매크로는 어휘 범위(lexcal scope)를 사용합니다: 코드나 매크로를 전역에 주입하는 것은 불가능합니다. 매크로를 사용하고 싶다면 명시적으로 `require`나 `import`를 사용하여 해당 매크로가 정의되어 있는 모듈을 가져와야 합니다.

* 매크로는 명시적입니다: 매크로를 명시적으로 호출하지 않고 실행하는 것은 불가능합니다. 예를 들어, 어떤 언어는 문법 변형이나 리플랙션을 통해 개발자들이 보이지 않는 곳에서 함수를 완전히 새롭게 쓸 수 있게 해줍니다. Elixir에서는, 매크로는 반드시 컴파일 시점에 명시적으로 호출되어야 합니다.

* 매크로 문법이 명확합니다: 많은 언어는 `quote`와 `unquote`를 위한 간략한 문법을 제공합니다. Elixir에서는 매크로 정의와 내부 표현식의 범위를 명확하게 구분 짓기 위해 이들을 명시적으로 사용하는 것을 선호합니다.

하지만 이런 보장에도 불구하고 개발자는 잘 동작하는 매크로를 작성함에 있어 커다란 역할을 합니다. 매크로를 사용해야겠다는 확신이 들었다면, 매크로는 자신의 API가 아니라는 점을 상기하세요. 내부 표현식을 포함한 매크로 정의를 짧게 유지하세요. 예를 들어 다음처럼 매크로를 작성하지 말고,

```elixir
defmodule MyModule do
  defmacro my_macro(a, b, c) do
    quote do
      do_this(unquote(a))
      ...
      do_that(unquote(b))
      ...
      and_that(unquote(c))
    end
  end
end
```

이렇게 작성하세요.

```elixir
defmodule MyModule do
  defmacro my_macro(a, b, c) do
    quote do
      # Keep what you need to do here to a minimum
      # and move everything else to a function
      do_this_that_and_that(unquote(a), unquote(b), unquote(c))
    end
  end

  def do_this_that_and_that(a, b, c) do
    do_this(a)
    ...
    do_that(b)
    ...
    and_that(c)
  end
end
```

이는 코드를 좀 더 명확하게 만들뿐 아니라, `do_this_that_and_that/3`을 직접 호출할 수 있으므로 테스트, 유지보수를 쉽게 만듭니다. 나아가 매크로에 의존하고 싶어 하지 않는 개발자들을 위한 API를 제공해줄 수도 있습니다.

이 강의와 함께, 매크로에 대한 소개를 마칩니다. 다음 장에서는 DSL에 대한 간략한 논의를 통해 매크로와 모듈 속성을 사용하여 모듈과 함수를 수식하거나 확장할 수 있는지 알아보죠.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Macros](http://elixir-lang.org/getting-started/meta/macros.html)
