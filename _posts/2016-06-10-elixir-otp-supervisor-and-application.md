---
layout: post
title: "Elixir - OTP: Supervisor and Application"
date: 2016-06-10 21:29:43 +0900
categories:
---

Elixir Tutorial 시리즈입니다. 거의 대부분은 튜토리얼의 한글 번역에 가깝습니다만, 생략되거나 추가로 주석을 달거나 하는 부분이 많습니다. 원문은 최하단의 링크를 참고하세요.

## Introduction

이제 애플리케이션은 수백개까지는 아니지만, 수십개를 감시할 수 있는 레지스트리를 가집니다. 이 구현은 그럭저럭 나쁘지않아 보입니다만, 버그가 없는 소프트웨어는 존재하지 않으므로 분명 문제가 발생할 것입니다.

문제가 발생하면 첫 번째로 취하는 행동은 이럴 것입니다: "일단 에러 처리를 하자". 하지만 Elixir에서는 다른 언어들에서 흔하게 볼 수 있는 예외를 잡아서 처리하는 방어적인 프로그래밍을 피합니다. 대신에 "그럼 실패하자!"라고 말합니다. 레지스트리에서 문제가 발생하게 만드는 버그가 있다고 한다면, 아무것도 걱정할 필요가 없습니다. 왜냐하면 이제 레지스트리를 재시작해 줄 슈퍼바이저를 만들 생각이니까요.

이 장에서는 슈퍼바이저와 애플리케이션에 대해서 배웁니다. 2개의 슈퍼바이저를 만들어서 프로세스들을 담당하도록 할 겁니다.

## 첫번째 슈퍼바이저

슈퍼바이저를 만드는 것은 GenServer를 만드는 것과 크게 다르지 않습니다. 우리는 `KV.Superisor`라는 이름의 모듈을 만들고, [Supervisor](http://elixir-lang.org/docs/stable/elixir/Supervisor.html)를 사용해서 `lib/kv/supervisor.ex` 파일에 구현할 것입니다:

```elixir
defmodule KV.Supervisor do
  use Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, :ok)
  end

  def init(:ok) do
    children = [
      worker(KV.Registry, [KV.Registry])
    ]

    supervise(children, strategy: :one_for_one)
  end
end
```

슈퍼바이저는 하나의 자식-레지스트리-을 가질 것입니다. worker는 다음과 같은 형식입니다:

```elixir
worker(KV.Registry, [KV.Registry])
```

는 다음을 통해서 프로세스를 시작할 것입니다:

```elixir
KV.Registry.start_link(KV.Registry)
```

`start_link`에 넘기는 인자는 프로세스의 이름입니다. 감시 하에 있는 프로세스에 이름을 주어서 다른 프로세스들이 해당 프로세스의 pid를 모르더라도 접근할 수 있게끔 만드는 것은 일반적인 방식입니다. 감시 받고 있는 프로세스가 죽어서 슈퍼바이저가 이를 재시작히여 pid가 변경되는 경우에도 유용합니다. 이름을 사용하면 최신의 pid를 명시적으로 가져올 필요 없이, 새로 시작되는 프로세스가 같은 이름으로 등록될 것을 보장할 수 있습니다. 또한 프로세스의 이름을 그것이 정의되어 있는 모듈과 같은 이름으로 등록하는 것도 꽤나 일반적입니다. 왜냐하면 디버깅이나, 동작하는 시스템의 내부를 확인할 때 직관적이기 때문입니다.

마지막으로 `supervise/2`를 `:one_for_one` 전략과 자식들의 리스트를 인수로 호출합니다.

관리 전략(supervision strategy)은 자식들에 문제가 발생했을 때 어떻게 되는지에 대한 설명입니다. `:one_for_one`는 자식이 죽으면 단 하나의 프로세스만 재시작될 것입니다. 아직 여기에서는 하나의 자식만을 가지고 있기 때문에 이것이면 충분합니다. `Supervisor`는 많은 종류이ㅡ 전략을 지원하고 있으며, 이는 이 장의 뒷 부분에서 설명하겠습니다.

`KV.Registry.start_link/1`는 이제 인수를 넘겨주길 기대하고 있기 때문에 해당 인수를 처리할 수 있게끔 구현을 변경해야 합니다. `lib/kv/registry.ex` 파일을 열어서 `start_link/0` 정의를 다음과 같이 변경하세요:

```elixir
  @doc """
  Starts the registry with the given `name`.
  """
  def start_link(name) do
    GenServer.start_link(__MODULE__, :ok, name: name)
  end
```

그리고 레지스트리를 시작할 때 이름을 주도록 테스트도 변경해야 합니다. `test/kv/registry_test.exs`의 `setup` 함수를 다음과 같이 변경하세요:

```elixir
  setup context do
    {:ok, registry} = KV.Registry.start_link(context.test)
    {:ok, registry: registry}
  end
```

`setup/2`은 `test/3`와 비슷하게 물론 테스트 컨텍스트를 받습니다. 게다가 어떤 값을 우리가 setup 블럭에 추가하더라도, 컨텍스트는 `:case`, `:test`, `:file`, `:line`와 같은 기본 키들을 포함할 겁니다. `context.test`를 사용해서 현재 동작하고 있는 테스트의 이름을 포함하는 레지스트리를 생성했었습니다.

이제 테스트를 통과할테니 남는 시간에 슈퍼바이저를 보죠. `iex -S mix`로 프로젝트 내부에서 콘솔을 띄운 뒤에, 수동으로 슈퍼바이저를 실행할 수 있습니다:

```iex
iex> KV.Supervisor.start_link
{:ok, #PID<0.66.0>}
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.70.0>}
```

슈퍼바이저를 시작할 때, 레지스트리 워커도 자동으로 시작되어, 곧바로 버킷을 만들 수 있게 해줍니다.

보통 직접 슈퍼바이저를 실행하는 경우는 거의 없습니다. 대신에 이를 애플리케이션 콜백으로 실행합니다.

## 애플리케이션 이해하기

지금까지 우리는 애플리케이션의 내부에서 작업을 하고 있었습니다. 우리가 파일을 변경하고 `mix compile`을 실행할 때마다, 콘솔에서 `Generated kv app`라는 메시지를 볼 수 있습니다.

`_build/dev/lib/kv/ebin/kv.app`에서 생성된 `.app` 파일을 찾을 수 있습니다. 내부가 어덯게 되어있는지 한번 살펴보죠:

```erlang
{application,kv,
             [{registered,[]},
              {description,"kv"},
              {applications,[kernel,stdlib,elixir,logger]},
              {vsn,"0.0.1"},
              {modules,['Elixir.KV','Elixir.KV.Bucket',
                        'Elixir.KV.Registry','Elixir.KV.Supervisor']}]}.
```

이 파일은 Erlang의 용어들을 포함하고 있습니다(당연하지만 Erlang의 문법입니다). Erlang에 친숙하지 않다고 하더라도, 이 파일이 애플리케이션의 정의에 대한 내용이라는 것은 쉽게 추측할 수 있을 겁니다. 여기에는 애플리케이션의 `version`과 내부에 정의되어 있는 모든 모듈의 리스트와 `mix.exs`에서 정의했었던 Erlang의 `kernel`, `elixir` 자신, 그리고 `logger`와 같이 의존하고 있는 외부의 모듈 리스트 등이 있습니다.

새로운 모듈을 추가하거나 할 때마다 이 파일을 매번 손으로 직접 변경하는 것은 무척 지루한 작업일 것입니다. 그것이 Mix는 개발자들을 위해서 이 파일을 생성하고 직접 관리해주는 이유입니다.

프로젝트에 있는 `mix.exs`에서 `application/0`가 반환하는 값을 변경하여 생성되는 `.app` 파일을 수정할 수도 있습니다. 첫 변경은 곧 해보게 될 겁니다.

### 애플리케이션 시작하기

애플리케이션의 명세인 `.app` 파일을 정의할 때, 애플리케이션의 시작과 종료를 한번에 할 수 있습니다. 여기에 대해서는 크게 걱정할 필요가 없는데, 다음과 같은 두가지 이유입니다:

1. Mix가 자동으로 현재 애플리케이션을 시작해줍니다.

2. 만약 Mix가 애플리케이션을 시작해주지 않더라도, 직접 애플리케이션을 시작할 때 아무것도 할 필요가 없다.

어떤 경우이든, 우선 Mix가 어떻게 애플리케이션을 실행하는지 살펴보죠. `iex -S mix`를 통해 콘솔을 열고 다음을 입력해봅시다:

```iex
iex> Application.start(:kv)
{:error, {:already_started, :kv}}
```

오, 이미 실행되어 있네요. Mix는 프로젝트의 `mix.exs` 파일에 정의되어 있는 모든 계층 구조를 실행하며, 의존하고 있는 모든 애플리케이션에 대해서도 동일하게 동작합니다.

Mix에게 애플리케이션을 시작하지 말라고 명령할 수도 있습니다. `iex -S mix run --no-start`를 통해서 실행해보세요:

```iex
iex> Application.start(:kv)
:ok
```

`:kv` 애플리케이션이나 Elixir에 의해서 기본으로 실행되는 `:logger` 애플리케이션을 중지할 수도 있습니다:

```iex
iex> Application.stop(:kv)
:ok
iex> Application.stop(:logger)
:ok
```

그리고 다시 애플리케이션을 실행해보죠:

```iex
iex> Application.start(:kv)
{:error, {:not_started, :logger}}
```

`:kv`가 의존하고 있는 모듈(여기에서는 `:logger`)이 실행도지 않았기 때문에 에러를 받게 됩니다. 수동으로 실행하는 경우에는 각각의 의존하는 프로세스들을 올바른 순서로 실행하거나 `Application.ensure_all_started`를 실행해야 합니다:

```iex
iex> Application.ensure_all_started(:kv)
{:ok, [:logger, :kv]}
```

뭔가 신나게 시작하지는 않지만, 애플리케이션을 어떻게 관리할 수 있는지를 보여주긴 합니다.

> `iex -S mix`를 실행하는 것은 `iex -S mix run`를 실행하는 것과 동등합니다. 그러므로 IEx를 Mix와 함께 실행할 때에 옵션을 추가로 주고 싶다면, 그저 `iex -S mix run`라고 타이핑하고 `run` 명령이 받을 수 있는 옵션들을 넘기세요. `run`에 대한 정보는 `mix help run`을 입력하면 확인할 수 있습니다.

### 애플리케이션 콜백

지금까지는 애플리케이션을 어떻게 시작하고 종료할 지에 대해서 설명해 왔기 때문에, 애플리케이션을 시작할 때 유용한 작업을 덧붙일 수 있는 방법도 분명 존재할 것입니다. 그리고 정말 있죠!

애플리케이션 콜백 함수에 대해서 설명하겠습니다. 이 함수는 애플리케이션이 시작될 때에 호출됩니다. 콜백 함수는 반드시 `{:ok, pid}`를 반환해야 하며, `pid`는 슈퍼바이저 프로세스의 식별자이어야 합니다.

애플리케이션 콜백은 2단계를 걸쳐서 설정할 수 있스빈다. 우선 `mix.exs` 파일을 열어서 `def application`를 다음과 같이 변경하세요:

```elixir
  def application do
    [applications: [:logger],
     mod: {KV, []}]
  end
```

`:mod` 옵션은 "애플리케이션 콜백 모듈"을 의미하며, 이에 들어가는 인수들은 애플리케이션이 시작될 때에 넘겨집니다. 애플리케이션 콜백 모듈은 [Application](http://elixir-lang.org/docs/stable/elixir/Application.html) 행동을 구현하는 모듈이라면 무엇이든 사용할 수 있습니다.

이제 `KV`를 모듈 콜백으로 지정했으니, `lib/kv.ex`에 있는 `KV` 모듈을 변경해야 합니다:

```elixir
defmodule KV do
  use Application

  def start(_type, _args) do
    KV.Supervisor.start_link
  end
end
```

`use Application` 선언을 하게 되면, `Supervisor`나 `GenServer`에서 처럼 한 쌍의 함수를 정의해야합니다. 이번에는 `start/2` 함수만 정의하면 됩니다. 만약 애플리케이션이 정지하는 시점에도 어떤 동작을 추가하고 싶다면 `stop/1` 함수를 정의하세요.

`iex -S mix`를 사용해서 다시 프로젝트 콘솔을 실행합시다. 그러면 `KV.Registry`라는 이름의 프로세스가 이미 동작하고 있는 것을 볼 수 있습니다:

```iex
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.88.0>}
```

훌륭합니다!

### 프로젝트? 애플리케이션?

Mix는 프로젝트와 애플리케이션 간의 차이를 만듭니다. `mix.exs` 파일의 내용에 기반해서 말해보자면, `:kv` 애플리케이션을 정의하고 있는 Mix 프로젝트라고 말할 수 있습니다. 나중에 살펴볼 장에서는 어떤 애플리케이션도 정의하지 않는 프로젝트들도 볼 수 있을 것입니다.

"프로젝트"라고 언급하면 Mix를 떠올려야 합니다. Mix는 프로젝트를 관리하기 위한 도구로 어떻게 프로젝트를 컴파일할 수 있는지, 테스트를 할지 등을 알고 있습니다. 또한 프로젝트와 관계있는 애플리케이션을 어떻게 컴파일하고, 시작할 지도 알고 있습니다.

애플리케이션에 대해서 이야기할 때에는 <abbr title="Open Telecom Platform">OTP</abbr>에 대해서 이야기하는 것입니다. 애플리케이션은 런타임에 의해서 시작되고 종료되는 존재입니다. [Application 모듈에 대한 문서](http://elixir-lang.org/docs/stable/elixir/Application.html)에서 더 많은 정보를 확인하거나, `def application`에서 지원하는 옵션을 `mix help compile.app`을 실행해서 확인할 수 있습니다.

## 간단한 one for one 슈퍼바이저

지금까지 애플리케이션의 생존 주기를 이용해서 자동적으로 시작, 종료되는 슈퍼바이저를 성공적으로 정의했습니다.

하지만 `KV.Registry`는 `handle_cast/2` 콜백을 이용해서 버킷 프로세스를 연결하고 감시한다는 점을 기억하세요.

```elixir
{:ok, pid} = KV.Bucket.start_link
ref = Process.monitor(pid)
```

연결(Link)은 양방향, 다시 말해서 버킷에 문제가 생가면 레지스트리도 죽는다는 것을 함축하고 있습니다. 슈퍼바이저를 설정했으므로, 레지스트리가 죽으면 자동적으로 재시작됩니다만, 레지스트리가 죽는다는것은 버킷 이름과 해당하는 프로세스에 대한 모든 정보를 잃어버린다는 것을 의미합니다.

다시 말하자면, 버킷에 문제가 생겼을 때에도 레지스트리는 정상적으로 동작하도록 만들고 싶습니다. 새 레지스트리 테스트를 작성해보죠:

```elixir
  test "removes bucket on crash", %{registry: registry} do
    KV.Registry.create(registry, "shopping")
    {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    # Stop the bucket with non-normal reason
    Process.exit(bucket, :shutdown)

    # Wait until the bucket is dead
    ref = Process.monitor(bucket)
    assert_receive {:DOWN, ^ref, _, _, _}

    assert KV.Registry.lookup(registry, "shopping") == :error
  end
```

이 테스트는 `:normal` 대신에 `:shutdown`을 보낸다는 점을 제외하면 "removes bucket on exit" 테스트와 비슷합니다. `Agent.stop/1`와는 다르게, `Process.exit/2`는 비동기적으로 동작하므로 버킷이 죽은 이후에 메시지가 전달될 것이라는 보장이 없기 때문에, 단순하게 곧장 `KV.Registry.lookup/2`를 호출할 수는 없습니다. 이 문제를 해결하기 위해서는 테스트 도중에 버킷을 감시하고 있다가, 종료되었다고 확신하는 시점에 레지스트리에 쿼리를 보내야 합니다.

버킷은 테스트 프로세스와 연결되어 있는 레지스트리와 연결되어 있으므로, 버킷을 죽이면 레지스트리도 죽고, 이어서 테스트 프로세스도 죽게 됩니다:

```
1) test removes bucket on crash (KV.RegistryTest)
   test/kv/registry_test.exs:52
   ** (EXIT from #PID<0.94.0>) shutdown
```

이 문제에 대한 한가지 해결책으로 `Agent.start/1`를 호출하고 그것을 레지스트리에서 사용하도록 `KV.Bucket.start/0`를 제공하는 방법이 있습니다. 하지만 이 방법은 버킷들을 어떤 프로세스와도 연결되지 않게 만들기 때문에 좋은 방법이라고는 할 수 없습니다. 다시 말해서 누군가가 `:kv` 애플리케이션을 종료시키면, 모든 버킷들은 접근이 불가능한 상태로 여전히 남아있을 것이라는 소리입니다. 그 뿐만 아니라, 프로세스에 접근할 수 없게 되면, 상태를 확인하기도 어려워집니다.

이런 문제를 해결하기 위해서 버킷들을 감시하고, 생성하는 새로운 슈퍼바이저를 정의합시다. 관리 전략중에는 `:simple_one_for_one`라는 것이 있는데, 이는 이러한 상황에 아주 잘 맞습니다: 개발자에게 워커 템플릿을 작성하고, 이 템플릿을 통해서 많은 자식들을 관리할 수 있게 해줍니다. 이 전략을 사용하면 슈퍼바이저의 초기화 과정에서는 워커가 생성되지 않으며, `start_child/2`가 호출될 때마다 하나의 워커가 시작됩니다.

그러면 `lib/kv/bucket/supervisor.ex`의 `KV.Bucket.Supervisor`에 다음과 같이 정의하세요:

```elixir
defmodule KV.Bucket.Supervisor do
  use Supervisor

  # A simple module attribute that stores the supervisor name
  @name KV.Bucket.Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, :ok, name: @name)
  end

  def start_bucket do
    Supervisor.start_child(@name, [])
  end

  def init(:ok) do
    children = [
      worker(KV.Bucket, [], restart: :temporary)
    ]

    supervise(children, strategy: :simple_one_for_one)
  end
end
```

처음 만든 슈퍼바이저와 다른 부분이 3개 있습니다.

이 프로세스는 단 하나만 생성할 것이므로 인수로서 사용할 프로세스 이름을 받는 대신, `KV.Bucket.Supervisor`라고 정의했습니다. 또한 `start_bucket/0` 함수를 정의하고 `KV.Bucket.Supervisor`의 자식으로서 버킷을 실행하게 만들었습니다. `KV.Bucket.start_link`를 레지스트리에서 직접 호출하는 대신에 `start_bucket/0`를 사용할 겁니다.

마지막으로 `init/1` 콜백에서 워커를 `:temporary`라고 정의했습니다. 이 말은 만약 버킷이 죽으면 재시작되지 않는다는 의미입니다! 이는 슈퍼바이저를 버킷들을 묶는 그룹처럼 사용하고 싶기 때문입니다. 버킷 생성은 언제나 레지스트리를 통해야 합니다.

`iex -S mix`를 실행하고 새 슈퍼바이저를 사용해보죠:

```iex
iex> {:ok, _} = KV.Bucket.Supervisor.start_link
{:ok, #PID<0.70.0>}
iex> {:ok, bucket} = KV.Bucket.Supervisor.start_bucket
{:ok, #PID<0.72.0>}
iex> KV.Bucket.put(bucket, "eggs", 3)
:ok
iex> KV.Bucket.get(bucket, "eggs")
3
```

레지스트리를 이 버킷 슈퍼바이저와 함께 동작하도록 수정해보죠:

```elixir
  def handle_cast({:create, name}, {names, refs}) do
    if Map.has_key?(names, name) do
      {:noreply, {names, refs}}
    else
      {:ok, pid} = KV.Bucket.Supervisor.start_bucket
      ref = Process.monitor(pid)
      refs = Map.put(refs, ref, name)
      names = Map.put(names, name, pid)
      {:noreply, {names, refs}}
    end
  end
```

이 변경을 반영하면, 버킷 슈퍼바이저가 없다는 이유로 테스트는 반드시 실패할 것입니다. 테스트마다 매번 직접 버킷 슈퍼바이저를 시작하는 대신에, 메인 관리 트리의 일부로서 자동으로 시작하게 만들어 봅시다.

## 관리 트리(Supervision tree)

애플리케이션에서 버킷 슈퍼바이저를 사용하려면 `KV.Supervisor`의 자식으로 추가해야합니다. 이제서야 슈퍼바이저를 관리하는 슈퍼바이저를 가지게 되었다는 사실을 눈치채셨나요? 이는 보통 "관리 트리(supervision tree)"라고 부릅니다.

`lib/kv/supervisor.ex`를 열고 `init/1`를 다음과 같이 변경하세요:

```elixir
  def init(:ok) do
    children = [
      worker(KV.Registry, [KV.Registry]),
      supervisor(KV.Bucket.Supervisor, [])
    ]

    supervise(children, strategy: :one_for_one)
  end
```

이번에는 슈퍼바이저를 자식으로 추가했고, 아무런 인자를 주지 않았습니다. 이제 테스트를 다시 실행하면 테스트가 모두 통과할 겁니다.

슈퍼바이저에 다른 자식을 추가했으므로, 이제 `:one_for_one` 전략이 여전히 옳은 선택인지를 다시 고민해볼 필요가 있습니다. 곧바로 떠오르는 결점 하나는 레지스트리와 버킷 슈퍼바이저간의 관계입니다. 만약 레지스트리가 사망한다면, 그 때 레지스트리가 가지고 있던 모든 버킷의 이름과 프로세스에 대한 정보가 소실되기 때문에, 버킷 슈퍼바이저도 함께 사망해야 합니다. 버킷 슈퍼바이저가 여전히 살아 있다면, 이전의 버킷들에는 접근할 수 없을 겁니다.

그러므로 `:one_for_all`이나 `:rest_for_one`와 같은 다른 관리 전략으로 갈아타는 것을 고려해야 합니다. `:one_for_all` 전략은 하나의 자식이라도 사망한다면 남은 모든 자식을 죽이고, 전부 재시작합니다. 이 전략은 현재의 상황에 적당하지만, 너무 과장된 행동을 취합니다. 예를 들어, 버킷 슈퍼바이저가 사망한 경우에는 레지스트리가 굳이 함께 재시작될 필요가 없습니다. 왜냐하면 레지스트리가 모든 버킷들을 관리하고, 정리 작업까지 수행할 수 있기 때문입니다. 이런 경우에는 `:rest_for_one`이 유용합니다: `:rest_for_one`은 관리 트리의 나머지 부분을 그대로 둔 채, 실패한 프로세스만을 재시작합니다. 그럼 이제 관리 트리를 재작성해봅시다:

```elixir
  def init(:ok) do
    children = [
      worker(KV.Registry, [KV.Registry]),
      supervisor(KV.Bucket.Supervisor, [])
    ]

    supervise(children, strategy: :rest_for_one)
  end
```

레지스트리의 워커에 문제가 생기면 레지스트리와 버킷 슈퍼바이저가 재시작됩니다. 만약 버킷 슈퍼바이저에 문제가 생기면 버킷 슈퍼바이저만이 재시작됩니다.

그 외의 전략이나 옵션들을 `worker/2`, `supervisor/2`, 그리고 `supervise/2` 함수에 사용할 수 있으니, [`Supervisor`](http://elixir-lang.org/docs/stable/elixir/Supervisor.html)와 [`Supervisor.Spec`](http://elixir-lang.org/docs/stable/elixir/Supervisor.Spec.html)를 꼭 한번 확인해보세요.

다음 장으로 넘어가기 전에 짚고 넘어갈 부분이 있습니다.

## 옵저버

이제 관리 트리를 정의했으므로, Erlang에 포함된 옵저버 도구를 소개하기 좋은 시점이 되었습니다. `iex -S mix`를 통해서 애플리케이션을 시작하고, 다음을 입력하세요:

```iex
iex> :observer.start
```

그러면 현재 시스템에 대한 모든 정보들, 일반적인 통계부터 차트, 현재 실행중인 모든 프로세스와 애플리케이션에 대한 목록에 이르기까지를 포함하는 팝업 GUI가 나타납니다.

애플리케이션 탭에서 현재 관리 트리와 함께 시스템 상에서 동작하는 모든 애플리케이션을 확인할 수 있습니다. `kv`를 선택해서 더 자세히 살펴볼 수도 있습니다.

 <img src="/images/20160610/kv-observer.png" width="640px">

그 뿐만이 아니라, 터미널에서 새 버킷을 생성하면, 옵저버의 관리 트리에서 새 프로세스가 추가되는 것을 볼 수 있습니다:

```iex
iex> KV.Registry.create KV.Registry, "shopping"
:ok
```

옵저버가 제공하는 것들이 뭐가 더 있는지에 대해서 알아보는 것은 맡기도록 하겠습니다. 다만 관리트리에 있는 프로세스를 더블 클릭하면 그에 대한 더 상세한 정보를 확인할 수 있으며, 우클릭으로 "kill signal"을 보내면 완벽하게 정말로 실패한 것처럼 동작하므로 슈퍼바이저가 기대대로 동작하는지 확인해볼 수 있습니다.

마지막으로, 옵저버와 도구는 언제나 관리 트리에서 프로세스를 시작하고 싶게 만드는 주된 이유입니다. 그게 일시적인 프로세스라고 하더라도 접근 가능하고, 조사 가능한 상태를 유지할 수 있게 해주기 때문입니다.

## 테스트에서 상태를 공유하기

지금까지 우리는 각 테스트가 고립되어 있다는 것을 보장하기 위해서 매번 레지스트리를 시작했습니다:

```elixir
  setup context do
    {:ok, registry} = KV.Registry.start_link(context.test)
    {:ok, registry: registry}
  end
```

하지만 레지스트리가 전역으로 등록되어 있는 `KV.Bucket.Supervisor`를 사용하도록 만들었기 때문에, 테스트들은 각자 자신만의 레지스트리가 있음에도 불구하고 이제 이 공유된 전역 슈퍼바이저에 의존합니다. 정말 그래야 하나요?

그때그때 다릅니다. 만약 우리가 공유되지 않는 전역 상태의 일부에 의존하고 있다면 문제가 없습니다. 예를 들어, 매번 이름을 통해 프로세스를 등록하고, 그리고 공유된 레지스트리에 이 프로세스를 등록합니다. 하지만 `context.test`를 통해 이 이름들이 각 테스트에서 유일하다는 것을 보장할 수 있다면 테스트간에 동시성이나 데이터 의존성 문제가 생기지 않을 것입니다.

버킷 슈퍼바이저에도 비슷한 추론을 적용할 수 있을 겁니다. 여러 레지스트리가 공유 중인 버킷 슈퍼바이저에서 버킷을 실행함에도 불구하고, 이 버킷들과 레지스트리들은 각각 고립되어 있습니다. `Supervisor.count_children(KV.Bucket.Supervisor)`와 같이 테스트들이 동시에 동작하는 경우 다른 결과를 가져다 줄 수 있는 함수를 사용하지 않는 이상 동시성 문제와는 맞닥뜨리지 않습니다.

버킷 슈퍼바이저의 공유되지 않는 부분에 의존하고 있으므로 현재의 테스트에서는 동시성 문제에 대해서 고려하지 않아도 됩니다. 만약 그런 문제가 생기게 된다면 각 테스트마다 슈퍼바이저를 실행하고 레지스트리의 `start_link` 함수에 인자로 넘겨주면 됩니다.

이제 애플리케이션이 적절하게 관리 및 테스트되고 있으므로 이제 어떻게 빠르게 만들지 확인해봅시다.

## Reference

 * [Elixir Homepage](http://elixir-lang.org)
 * [Elixir - OTP: Supervisor and Application](http://elixir-lang.org/getting-started/mix-otp/supervisor-and-application.html)
