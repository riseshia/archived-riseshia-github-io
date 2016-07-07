---
layout: post
title: "Elixir - OTP: Quote and unquote"
date: 2016-07-07 21:01:39 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 있습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

Elixir 프로그램은 자기 자신만의 데이터 구조로서 표현됩니다. 이 장에서는 이 데이터 구조가 어떻게 생겼는지, 어떻게 조합할 수 있는지를 배웁니다. 이 장에서 배우는 개념은 다음 장에서 깊게 살펴볼 매크로를 만들기 위한 레고 블럭과 같습니다.

## Quoting

Elixir 프로그램은 원소 3개인 튜플로 구성되어 있습니다. 예를 들어서, `sum(1, 2, 3)` 호출은 내부적으로 다음과 같이 표현됩니다.

```elixir
{:sum, [], [1, 2, 3]}
```

`quote` 매크로를 사용하면 내부 표현식을 얻을 수 있습니다.

```iex
iex> quote do: sum(1, 2, 3)
{:sum, [], [1, 2, 3]}
```

첫 번째 원소는 함수의 이름이며, 두 번째는 메타 정보를 포함하는 키워드 리스트이고, 마지막은 인수 리스트입니다.

연산자들 역시 이러한 튜플로 표현됩니다.

```iex
iex> quote do: 1 + 2
{:+, [context: Elixir, import: Kernel], [1, 2]}
```

맵도 `%{}`를 호출하는 형태로 표현할 수 있죠.

```iex
iex> quote do: %{1 => 2}
{:%{}, [], [{1, 2}]}
```

변수들도 위와 같은 값이 3개 있는 튜플로 표현됩니다. 단, 마지막 원소는 리스트가 아닌 아톰입니다.

```iex
iex> quote do: x
{:x, [], Elixir}
```

복잡한 표현식의 내부 표현식을 드러내면 코드가 이와 같은 튜플들로 구성되어 있으며, 이것들은 서로 간에 트리 형태로 중첩되어 있다는 것을 확인할 수 있습니다. 많은 언어에서 이러한 표현식을 추상 문법 트리(Abstract Syntax Tree, AST)라고 말합니다. Elixir는 이들을 Quoted Expression(역주: 도저히 적당한 표현을 못 찾아서... 이하에서는 내부 표현식이라고 기술합니다)라고 부릅니다.

```iex
iex> quote do: sum(1, 2 + 3, 4)
{:sum, [], [1, {:+, [context: Elixir, import: Kernel], [2, 3]}, 4]}
```

때때로 Quoted Expression을 사용하면 코드의 원래 표현식을 확인하는 것이 유용한 때가 있습니다. 이는 `Macro.to_string/1`을 사용하면 됩니다:

```iex
iex> Macro.to_string(quote do: sum(1, 2 + 3, 4))
"sum(1, 2 + 3, 4)"
```

일반적으로 표현식의 튜플은 다음과 같은 형식으로 구성되어 있습니다.

```elixir
{atom | tuple, list, list | atom}
```

* 첫 번째 원소는 아톰이거나 같은 내부 표현식 튜플입니다;
* 두 번째 원소는 숫자나, 문맥 정보와 같은 메타 정보를 포함하는 키워드 리스트입니다;
* 세 번째 원소는 함수 인자의 리스트이거나, 아톰입니다. 만약 이 원소가 아톰인 경우, 튜플은 변수를 의미합니다.

그리고 내부 표현식으로 변환하더라도 튜플이 아닌 자기 자신을 반환하는 리터럴이 5개 있습니다.

```elixir
:sum         #=> Atoms
1.0          #=> Numbers
[1, 2]       #=> Lists
"strings"    #=> Strings
{key, value} #=> Tuples with two elements
```

대부분의 Elixir 코드는 기저에 있는 내부 표현식으로 직관적인 방법으로 변환됩니다. 다른 코드 예제들을 직접 내부 표현식으로 변환해보고 어떻게 바뀌는지 확인해보길 추천합니다. 예를 들어 `String.upcase("foo")`는 뭐라고 변환되나요? `if(true, do: :this, else: :that)`는 `if true do :this else :that end`와 같다는 것을 앞서 배웠습니다. 내부 표현식을 보며 어떻게 그렇다고 말할 수 있을까요?

## Unquoting

Quote는 어떤 코드의 내부 표현식을 가져오는 과정을 말합니다. 때때로 내부 표현식에 다른 코드를 주입해야 하는 경우가 있습니다.

예를 들어, 숫자를 가지고 있는 `number`라는 변수를 가지고 있고, 내부 표현식에 이 값을 주입하고 싶습니다.

```iex
iex> number = 13
iex> Macro.to_string(quote do: 11 + number)
"11 + number"
```

이는 우리가 원하는 모습이 아닙니다. `number` 변수의 값이 주입되지 않았고, `number`는 내부 표현식이 아니기 때문입니다. `number` 변수의 *값*을 주입하려면, 내부 표현식 안에서 `unquote`를 사용해야 합니다.

```iex
iex> number = 13
iex> Macro.to_string(quote do: 11 + unquote(number))
"11 + 13"
```

`unquote`는 함수의 이름을 주입할 때에도 사용할 수 있습니다.

```iex
iex> fun = :hello
iex> Macro.to_string(quote do: unquote(fun)(:world))
"hello(:world)"
```

어떤 경우에는 리스트의 내부에 많은 값을 주입해야 할 때도 있습니다. 예를 들어 `[1, 2, 6]`이라는 리스트를 가지고 있고, 여기에 `[3, 4, 5]`를 주입하고 싶다고 합시다. `unquote`를 사용해서는 원하는 결과를 얻을 수 없습니다:

```iex
iex> inner = [3, 4, 5]
iex> Macro.to_string(quote do: [1, 2, unquote(inner), 6])
"[1, 2, [3, 4, 5], 6]"
```

지금이 바로 `unquote_splicing`을 소개할 때입니다.

```iex
iex> inner = [3, 4, 5]
iex> Macro.to_string(quote do: [1, 2, unquote_splicing(inner), 6])
"[1, 2, 3, 4, 5, 6]"
```

Unquote는 매크로와 관련된 작업을 할 때 무척 유용합니다. 개발자가 매크로를 작성할 때에는 코드 조각들을 받아서 이들을 다른 코드 조각에 주입하고, 이것들이 코드로 변환되거나 컴파일 시간에 코드를 생성하는 코드를 쓰게 됩니다.

## Escaping

이 장의 처음에서부터 보아왔듯, 몇몇 값들만이 Elixir에서 유효한 내부 표현식이 될 수 있습니다. 예를 들어 맵은 유효한 내부 표현식이 아닙니다. 값이 4개인 튜플도 마찬가지입니다. 하지만 그런 값들도 내부 표현식으로 *표현될 수 있습니다*.

```iex
iex> quote do: %{1 => 2}
{:%{}, [], [{1, 2}]}
```

어떤 경우에는 *저런 값*을 *내부 표현식*에 주입해야 할 수도 있습니다. 이를 위해서는 일단 이러한 값들을 `Macro.escape/1`을 사용하여 내부 표현식으로 변환할 필요가 있습니다.

```iex
iex> map = %{hello: :world}
iex> Macro.escape(map)
{:%{}, [], [hello: :world]}
```

매크로는 내부 표현식을 받아서 내부 표현식을 반환해야 합니다. 하지만 매크로를 실행하는 동안 내부 표현식이 아닌 값들을 사용해서 작업하고, 이를 내부 표현식과 구분하는 경우가 있습니다.

다시 말하자면, 일반 Elixir 값(예를 들어, 리스트, 맵, 프로세스, 레퍼런스 등)과 내부 표현식을 구별하는 것은 중요합니다. 정수, 아톰, 문자열과 같은 값들은 자기 자신이 내부 표현식입니다. 맵과 같은 다른 값들은 명시적으로 변환되어야 합니다. 마지막으로 함수나 레퍼런스와 같은 값들은 내부 표현식으로 변환될 수 없습니다.

`quote`와 `unquote`에 대한 더 자세한 설명은 [`Kernel.SpecialForms` 모듈](http://elixir-lang.org/docs/stable/elixir/Kernel.SpecialForms.html)을 참고하세요. `Macro.escape/1`에 대한 문서와 내부 표현식에 대한 다른 함수들에 대한 설명은 [`Macro` 모듈](http://elixir-lang.org/docs/stable/elixir/Macro.html)에서 찾을 수 있습니다.

이 소개를 통해서 드디어 첫 번째 매크로를 구현할 준비가 되었습니다. 다음 장으로 넘어가도록 하죠.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Distributed tasks and configuration](http://elixir-lang.org/getting-started/mix-otp/distributed-tasks-and-configuration.html)
