---
layout: post
title: "Elixir 1.4.0 Preview"
date: 2017-01-10 14:57:16 +0900
categories:
---

1월 5일, Elixir 1.4.0이 [릴리스](http://elixir-lang.org/blog/2017/01/05/elixir-v1-4-0-released/)되었습니다.
어떤 것이 추가되고, 어떤 변경이 있었는지 살펴보도록 하죠.

## `Registry`

Elixir의 표준 라이브러리에 [`Registry`](http://elixir-lang.org/docs/master/elixir/Registry.html)라는 모듈이 추가되었습니다.
한 문장으로 간단하게 요약해보자면 '분산되고 확장 가능한 키-값 저장소`입니다.
이는 프로세스 이름을 다루거나 디스패쳐, pub-sub 시스템을 간단하게 구현할 수
있게 해줍니다. Elixir Tutorial을 진행해보았다면, 또는 이와 비슷한 기능을
구현해보신 적이 있다면 금방 이해하실 수 있을 겁니다. 이전까지였다면 `ets`와
`GenServer`를 이용해서 직접 구현해야 했습니다.
`Registry` 모듈은 다음과 같은 특징을 가지고 있습니다.

- Local: 저장소에는 현재 노드에서만 접근할 수 있습니다.
- Decentralized: 생성된 레지스트리를 단일 프로세스에서 관리하지 않습니다.
- Scalable: 파티션을 나눔에 따라서 코어를 추가하며, 이에 따른 [성능 향상은 선형 함수에 근사합니다](https://docs.google.com/spreadsheets/d/1MByRZJMCnZ1wPiLhBEnSRRSuy1QXp8kr27PIOXO3qqg).

### Options

레지스트리를 만들때에 두가지 옵션이 있습니다.

- `:unique`: 키의 중복 등록을 허용하지 않습니다.
- `:duplicate`: 키의 중복 등록을 허용합니다. 이 경우, `:via`를 사용하여 레지스트리에 프로세스를 등록할 수 없습니다.

### `:via`

프로세스의 이름 레지스트리를 사용하는 경우, 프로세스를 생성(i.e. `start_link`)한 뒤,
프로세스를 레지스트리에 등록하게 됩니다. 매번 이렇게 등록하는 것도 귀찮습니다.
그래서 `:via`라는 옵션이 Registry 모듈과 함께 추가되었는데요. 우선 코드를 보시죠.

```elixir
{:ok, _} = Registry.start_link(:unique, Registry.ViaTest)
name = {:via, Registry, {Registry.ViaTest, "agent"}}
{:ok, _} = Agent.start_link(fn -> 0 end, name: name)
Agent.get(name, & &1)
#=> 0
Agent.update(name, & &1 + 1)
Agent.get(name, & &1)
#=> 1
```

`{:via, Registry, {registry, key}}` 라는 튜플을 만들어서 start_link에 프로세스
이름 대신 넘겨줍니다. 그러면 짠! 마법처럼 프로세스가 생성됩니다.
코드에서 일어난 일을 설명하자면, 'agent'라는 키에 프로세스가 등록되어 있지 않다면,
그 키에 새 프로세스를 만든 뒤, 등록합니다. 그리고 그 뒤로 사용할 때에도
레지스트리를 통해서 프로세스에 접근할 수 있습니다.
이미 키에 프로세스가 등록되어 있다구요? 그 경우 `start_link`는
`{:error, {:already_started, current_pid}}`를 반환하게 됩니다.

### `dispatch`

`dispatch`를 사용하여 디스패쳐인 것처럼 사용할 수도 있습니다. 역시 코드를 봅시다.

```elixir
{:ok, _} = Registry.start_link(:duplicate, Registry.DispatcherTest)
{:ok, _} = Registry.register(Registry.DispatcherTest, "hello", {IO, :inspect})
Registry.dispatch(Registry.DispatcherTest, "hello", fn entries ->
  for {pid, {module, function}} <- entries, do: apply(module, function, [pid])
end)
# Prints #PID<...> where the pid is for the process that called register/3 above
#=> :ok
```

Elixir는 어디랑은 다르게 공식 문서가 참 친절하고 보기 좋아서 행복합니다. 

Pub-sub을 만들어 볼까요?

```elixir
{:ok, _} = Registry.start_link(:duplicate, Registry.PubSubTest,
                               partitions: System.schedulers_online)
{:ok, _} = Registry.register(Registry.PubSubTest, "hello", [])
Registry.dispatch(Registry.PubSubTest, "hello", fn entries ->
  for {pid, _} <- entries, do: send(pid, {:broadcast, "world"})
end)
#=> :ok
```

좋네요. XD

## `Task.async_stream`

```elixir
collection
|> Enum.map(&Task.async(SomeMod, :function, [&1]))
|> Enum.map(&Task.await/1)
```

이런 코드를 작성하는 경우가 가끔 있습니다. 이 코드의 좋지 않은 부분은 두 가지입니다.

- 컬렉션의 크기가 큰 경우, 프로세스를 그 숫자만큼 한번에 생성한다는 점.
- 가독성이 영 좋지 않다는 점.

전자의 경우, 동시에 처리할 수 있는 숫자를 제한하고 싶은 경우가 있을 겁니다.
`Task.async_stream`은 `max_concurrency` 옵션을 통해서 이러한 상황을 제어할 수 있게 도와줍니다.

```elixir
collection
|> Task.async_stream(SomeMod, :function, [], max_concurrency: 8)
|> Enum.to_list()
```

해당 함수들은 지연 평가되며, 특정 슈퍼바이저 하에서 실행하고 싶다면
`Task.Supervisor.async_stream`을 사용하면 됩니다.


## Application inference

`deps`에 포함되어 있는 의존성의 경우, `application`에 명시하지 않더라도
Mix에서 자동으로 가져올 수 있게 되었습니다.

```elixir
def application do
  [applications: [:logger, :plug, :postgrex]]
end

def deps do
  [{:plug, "~> 1.2"},
   {:postgrex, "~> 1.0"}]
end
```

이랬던 설정을,

```elixir
def application do
  [extra_applications: [:logger]]
end

def deps do
  [{:plug, "~> 1.2"},
   {:postgrex, "~> 1.0"}]
end
```

이렇게 작성할 수 있게 됩니다. `deps`에 포함되지만 가져오고 싶지 않은 의존성은 `runtime: false`를 이용해주세요.

## Deprecations

### Soft deprecations

이 제거 예정 목록은 약한 제거 예정입니다. 다시 말해서,
아직 `deprecated` 경고를 보여주지 않으며, 문서를 생성하지 않도록 변경된 경우도 있습니다.

- `Enum.partition/2`가 제거 예정이 되었습니다. `Enum.split_with/2`를 사용해주세요(1.6부터 경고 예정).
- `System`의 `time_unit`에서 복수형 표현이 제거될 예정입니다. 이는 Erlang의 변경을 따라가는 부분입니다(2.0부터 경고 예정).
- `ExUnit`에서 `GenEvent` 대신 `GenServer` 콜백을 사용하는 방식이 권장됩니다. 이는 `GenEvent`가 제거 예정이 된 것에 따른 변경입니다(1.5부터 경고 예정).

### Deprecations

- `Access.key/1`이 제거 예정이 되었으며, `Access.key/2`를 사용하여 찾을 수 없는 경우의 기본값을 제공하도록 권장됩니다.
- `Behaviour` 모듈이 제거 예정이 되었습니다. [`@callback`을 사용하여 직접 정의해주세요](https://elixirschool.com/ko/lessons/advanced/behaviours/).
- `Enum.uniq/2`가 제거 예정이 되었으며, `Enum.uniq_by/2`를 사용해주세요.
- `Stream.uniq/2`가 제거 예정이 되었으며 `Stream.uniq_by/2`를 사용해주세요.
- `Float.to_char_list/2`, `Float.to_string/2`가 제거 예정이 되었습니다. 필요하다면 Erlang 함수를 사용해주세요.
- Private 메소드를 오버라이드할 수 없게 됩니다.
- 변수가 함수 호출처럼 사용되는 경우에 경고를 던집니다.
- `OptionParser`에서 여러 글자로 된 별칭을 사용할 수 없게 됩니다.
- `IEx.Helpers.import_file/2`가 제거 예정이 되었으며 `IEx.Helpers.import_if_available/1`을 사용해주세요.
- `Mix.Utils.underscore/1`과 `Mix.Utils.camelize/1`이 제거 예정이 되었습니다.

## Enhancements

### `Date.compare/2`, `Time.compare/2`, `NaiveDateTime.compare/2`, `DateTime.compare/2` 추가

날짜를 비교하기 위한 함수들이 추가되었습니다. 값을 받아서 비교한뒤, 앞의 인자가 작으면 `:lt`, 같으면 `:eq`, 크면 `:gt`를 반환합니다.
다른 타입의 날짜간에도 비교가 가능하며, 호출한 `compare`의 모듈에 따라서 비교하는 값이 달라집니다.

```iex
iex> Date.compare(~D[2016-04-16], ~N[2016-04-28 01:23:45])
:lt
iex> Date.compare(~D[2016-04-16], ~N[2016-04-16 01:23:45])
:eq
iex> Date.compare(~N[2016-04-16 12:34:56], ~N[2016-04-16 01:23:45])
:eq
```

여기에서는 `Date.compare/2`이므로 날짜만을 보고 비교합니다.

### `NaiveDateTime.add/2`, `NaiveDateTime.diff/2` 추가

`NaiveDateTime`을 가공하기 위한 함수가 추가되었습니다.
이름만으로도 내용은 충분히 추측이 가능하죠? 두번째 인자로는 첫번째 값의 단위를 지정할 수 있습니다.

```iex
iex> NaiveDateTime.add(~N[2014-10-02 00:29:10], -2)
~N[2014-10-02 00:29:08]
# can work with other units
iex> NaiveDateTime.add(~N[2014-10-02 00:29:10], 2_000, :millisecond)
~N[2014-10-02 00:29:12]

iex> NaiveDateTime.diff(~N[2014-10-02 00:29:12], ~N[2014-10-02 00:29:10])
2
iex> NaiveDateTime.diff(~N[2014-10-02 00:29:12], ~N[2014-10-02 00:29:10], :microsecond)
2_000_000
```

### Etc

- `Date.leap_year?/1`, `Date.day_of_week/1` 추가됨.
- `Date`, `Time`, `NaiveDateTime`의 API가 상호 호환될 수 있도록 유연해짐.
- n번째 원소에 대해서만 콜백을 실행하는 `Enum.map_every/2`, `Stream.map_every/2`이 추가됨.
- 여러 엔트리를 한번에 zip할 수 있는 `Enum.zip/1`, `Stream.zip/1`이 추가됨.
- `min/2`, `max/2`, `min_max/2`, `min_by/3`, `max_by/3`, `min_max_by/3`에서 호출된 열거자에 아무것도 없는 경우에 기본값을 반환하는 콜백을 넘길 수 있게 됨.
- `List.pop_at/3`, `List/myers_difference/2`가 추가되었습니다.
- 넘겨준 실수의 분수 표현값을 `{분자, 분모}` 형태로 반환하는 `Float.ratio/1`이 추가됨.
- `GenServer.handle_info/2`의 기본 구현을 사용하면 경고가 나오도록 변경
- `Integer.digits/2`이 이제 음수를 받을 수 있게 되었습니다.
- `Integer.mod/2`와 `Integer.floor_div2/`가 추가되었습니다. 이를 사용하면 음수에서의 나머지 처리가 간편해집니다.
- `IO.inspect/2`에 다수 호출을 구별하기 위한 `:label` 옵션이 추가되었습니다.
- `ExUnit.Formatter`는 이제 `lhs`와 `rhs` 대신에 좀 더 명확한 `left`, `right`를 사용합니다.
- 유니코드 9.0.0을 지원합니다.
- 컴파일/런타임 에러 메시지가 다수 개선되었습니다.

## Etc

### Syntax coloring

이제 IEx 세션에서 데이터를 보여줄때 각 요소별로 색상이 다르게 보입니다!
~~이제 눈이 빠져라 화면을 쳐다보지 않아도 됩니다~~

![Screenshot](http://elixir-lang.org/images/contents/iex-coloring.png)

이 스크린샷은 [여기](http://elixir-lang.org/blog/2017/01/05/elixir-v1-4-0-released/)에서 끌어왔습니다.
각각의 색상은 `:syntax_colors` 옵션을 사용하여 변경할 수 있습니다.

```elixir
IEx.configure [colors: [syntax_colors: [atom: :cyan, string: :green]]]
```

### Mix install from SCM

`escript`를 mix를 통하여 직접 설치할 수 있게 되었습니다.

```elixir
mix escript.install hex ex_doc
```

이런 느낌. PATH에 `~/.mix/escripts`를 추가하면 `ex_doc`을 실행할 수 있게 됩니다.


## Conclusion

개인적으로는 꽤나 괜찮아보이는 개선이 많이 보이는데 어떠셨나요? 앞으로는 Elixir 1.4와 함께 즐거운 개발하세요. XD

## Reference

- [Elixir v1.4 released](http://elixir-lang.org/blog/2017/01/05/elixir-v1-4-0-released/)
- [Elixir v1.4 release note](https://github.com/elixir-lang/elixir/releases/tag/v1.4.0)
- [What's Coming in Elixir 1.4](http://qiita.com/shufo/items/f31478207661f0c34197)
- [Registry](http://elixir-lang.org/docs/master/elixir/Registry.html)
